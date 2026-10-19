# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Nikky Navvya |
| Version | v1.0 |
| Last updated | 2026-10-19 |
| Status | Draft |

> **How to read this document**
> - This file is the single source of truth. A developer must be able to build the whole module from it without asking the author anything. Where the product brief was silent, the decision is written here with its reason (see section 3.3).
> - IDs are permanent. Nothing is renumbered or reused. Removed items are marked **Deprecated** and kept.
> - "Platform" always means the main SurfingBear web platform that already handles sign-up, login and billing. "Module" always means the Resume Optimizer described here.
> - Sizes use binary units: 1 MiB = 1,048,576 bytes. Times in the API are ISO 8601 in UTC. Calendar-month quotas use the `Asia/Kolkata` timezone (IST).
>
> **ID rules**
>
> | Prefix | Used for | Example |
> |---|---|---|
> | `CR-` | Company-wide common requirement | CR-001 |
> | `FR-` | Functional requirement | FR-001 |
> | `NFR-` | Non-functional requirement | NFR-001 |
> | `CMP-` | Architecture component | CMP-001 |
> | `TBL-` | Database table | TBL-001 |
> | `API-` | API endpoint | API-001 |
> | `ERR-` | Error code | ERR-001 |

---

## 0. Product summary (from the brief, with the decisions made here)

SurfingBear users, already logged in on the platform, open the Resume Optimizer module. They upload a resume (PDF or DOCX), pick a **target job role** from a fixed catalogue and an **experience level** (`entry`, `mid`, `senior`, `lead`), and start an **analysis**. A background worker sends the extracted resume text plus the role's skill list to the AI provider (Anthropic Claude). The result is:

- an **overall ATS score 0–100**, computed by the backend from **six dimension scores** (section 3.1, FR-009);
- the list of **missing skills** for that role and level, taken from the role's skill catalogue;
- **rewrite suggestions** for weak lines (original line, suggested line, reason).

Plans come from the platform: `free` or `pro`. Free users get **3 analyses per calendar month (IST)** and see only the **first 3 missing skills** and the **first 2 rewrite suggestions**; the rest are shown as locked with a count. Pro users get **50 analyses per month** and see everything. Any user can download the analysis as a **PDF**. Resume files and extracted text are **deleted 30 days** after upload; analysis results are kept **365 days** or until the user deletes them. Users can **export** or **erase** all their module data at any time (DPDP Act rights). The module stores all data in **AWS ap-south-1 (Mumbai)**.

---

## 1. Company-wide common requirements

Rules every Divii / Hanriver project must follow. Each rule has an ID and must be specific enough to be checked.

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-001 | Authentication | Every request to the module API, except `GET /health` and the `/internal/*` webhooks, must carry a platform session JWT (RS256) either in the httpOnly cookie `sb_session` or in `Authorization: Bearer <jwt>`. The JWT is verified against the platform JWKS (`PLATFORM_JWKS_URL`), issuer must equal `PLATFORM_JWT_ISSUER`, and `exp` must be in the future. Invalid or missing token → `401 UNAUTHENTICATED`. | Integration test: request without token, with expired token, with wrong issuer, with tampered signature → all 401 |
| CR-002 | Authentication | The module never stores passwords, emails, names or phone numbers. The only user identifier stored is the platform user UUID (`sub` claim). | DB schema review: no such columns exist (section 5) |
| CR-003 | Authentication | `/internal/*` webhooks from the platform are authenticated with HMAC-SHA256 over the raw body using `PLATFORM_WEBHOOK_SECRET`, sent in header `X-SurfingBear-Signature: sha256=<hex>`, plus header `X-SurfingBear-Timestamp` (unix seconds) that must be within ±300 s of server time. Failure → `401 INVALID_SIGNATURE`. | Integration test with valid, stale and wrong-secret signatures |
| CR-004 | Roles | Roles are `user` and `admin`, taken from the JWT claim `role`. Plan is `free` or `pro`, taken from the JWT claim `plan` and mirrored into `users.plan` (TBL-001) on every authenticated request. The role × permission table in 1.2 is enforced in the auth middleware (CMP-003), not in handlers. | Integration test: `user` calling an `/admin/*` endpoint → `403 FORBIDDEN` |
| CR-005 | Roles | A user can read, download or delete only rows whose `user_id` equals their own `sub`. Accessing another user's resource returns `404 NOT_FOUND` (not 403), so resource IDs cannot be enumerated. | Integration test: user B requests user A's analysis id → 404 |
| CR-006 | Logging | Logs are JSON, one object per line, written to stdout, with fields `ts` (RFC 3339 UTC), `level` (`debug`/`info`/`warn`/`error`), `request_id`, `user_id`, `method`, `path`, `status`, `duration_ms`, `msg`. Personal data (resume text, file contents, original file names, AI prompt or response bodies) is **never** logged. | Code review + grep of log sample for `extracted_text`, `original_filename`, `suggested_line` fields → none found |
| CR-007 | Logging | Every request gets a `request_id` (UUID v4) generated in middleware or taken from an incoming `X-Request-Id` header, returned in the response header `X-Request-Id`, included in every log line and in every error body. | Integration test checks the header on success and error responses |
| CR-008 | Error handling | Every non-2xx response uses exactly the format in 1.4. No other error shape is allowed, including framework defaults for 404 and 405 and panics (a recovery middleware converts panics into `500 INTERNAL_ERROR`). | Integration test for unknown route, wrong method and a forced panic |
| CR-009 | Error handling | Every error code is listed once in section 6.3 with a fixed HTTP status. Handlers may only return codes from that list. | Code review: `ErrCode` is a Go enum generated from section 6.3 |
| CR-010 | Security | Secrets (DB password, Redis password, AWS keys, `ANTHROPIC_API_KEY`, `PLATFORM_WEBHOOK_SECRET`) are read from environment variables only, never from files committed to git. `.env.example` lists every variable with a dummy value. | CI step runs `gitleaks detect` and fails on any finding |
| CR-011 | Security | CORS: only origins in `CORS_ALLOWED_ORIGINS` (comma-separated) are allowed; `Access-Control-Allow-Credentials: true`; allowed methods `GET, POST, PUT, DELETE, OPTIONS`; allowed headers `Content-Type, Authorization, X-Request-Id`. Production value is `https://www.surfingbear.com` only. | Integration test: preflight from an unlisted origin receives no CORS headers |
| CR-012 | Security | All input is validated before any DB or external call: JSON bodies against the field tables in section 6, path IDs must be UUID v4, file uploads against FR-002. Validation failure → `400 VALIDATION_ERROR` with a `details` array naming each field. Unknown JSON fields are rejected. | Unit tests per endpoint for each validation rule |
| CR-013 | Security | Rate limiting per user (by `sub`) in Redis: 60 requests per rolling 60 s on all authenticated endpoints, and additionally 10 uploads (`POST /resumes`) per rolling 60 minutes. Exceeding → `429 RATE_LIMITED` with `Retry-After` header in seconds. | Load test with 61 requests in 60 s → the 61st is 429 |
| CR-014 | Security | Data retention: resume files and extracted text are deleted 30 days after upload; analyses and their child rows 365 days after completion; audit logs 365 days; application logs 30 days. Deletion is done by the retention scheduler (CMP-009) daily at 02:00 IST and is logged in `audit_logs`. | Scheduler unit test with a fake clock; audit row exists after purge |
| CR-015 | Security | Transport is HTTPS only (TLS 1.2 or higher) at the load balancer; the API responds with `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Data at rest: RDS storage encryption and S3 SSE-KMS are enabled. | Infra review; `curl -I` shows the HSTS header |
| CR-016 | Security | Dependencies are scanned in CI (`govulncheck` for Go, `npm audit --audit-level=high` for the frontend). A high or critical finding fails the build. | CI pipeline configuration |
| CR-017 | i18n | v1 UI and API messages are English only. All UI strings live in `web/messages/en.json`; no string literals in components. API `message` fields are for developers; the frontend maps `error.code` to a user-facing string from `en.json`. Dates are sent as ISO 8601 UTC and rendered in the browser's timezone. | Code review: `grep` for JSX text literals in `components/` returns none |
| CR-018 | Environments | Four environments: `local`, `dev`, `staging`, `prod` (env var `APP_ENV`). The differences are listed in 1.7. Code must not branch on environment name except for the items in that table. | Code review: `APP_ENV` referenced only in `config/` |
| CR-019 | Deployment | Every merge to `main` runs CI (lint, tests, build, image push). Deploy to `dev` is automatic; `staging` and `prod` are manual approvals in GitHub Actions. DB migrations run as a separate job before the new version starts and must be backward compatible with the previous version (expand/contract). | Pipeline definition `.github/workflows/ci.yml`, `deploy.yml` |
| CR-020 | Deployment | Rollback = redeploy the previous image tag (tags are the git SHA). Migrations are never rolled back automatically; a failed migration stops the deploy and alerts `#alerts-resume-optimizer`. | Runbook `docs/runbook.md` and a rehearsed rollback on staging |

### 1.1 Authentication

- The platform issues a JWT (RS256) at login and sets it as cookie `sb_session` (`HttpOnly; Secure; SameSite=Lax; Domain=.surfingbear.com`). The module reads that cookie or the `Authorization: Bearer` header (mobile clients). Cookie takes precedence if both are present.
- JWT claims used by the module:

| Claim | Type | Meaning |
|---|---|---|
| `sub` | string (UUID v4) | Platform user id. Used as `users.id`. |
| `iss` | string | Must equal `PLATFORM_JWT_ISSUER` (prod: `https://auth.surfingbear.com`). |
| `exp`, `iat` | number | Standard. Clock skew tolerance 60 s. |
| `role` | `"user"` \| `"admin"` | Authorization role (1.2). Missing → treated as `user`. |
| `plan` | `"free"` \| `"pro"` | Billing plan. Missing → treated as `free`. |

- JWKS is fetched from `PLATFORM_JWKS_URL`, cached in Redis key `jwks:platform` for 3600 s. On a `kid` not in cache, the module refetches once, then fails with 401.
- The module does not issue tokens, refresh tokens or sessions of its own.

### 1.2 Roles and permissions

| Permission | `user` (free) | `user` (pro) | `admin` |
|---|---|---|---|
| Record consent, view own profile and usage (API-001, API-002) | ✔ | ✔ | ✔ |
| Browse job roles and skills (API-003, API-004) | ✔ | ✔ | ✔ |
| Upload, list, view, delete own resumes (API-005–API-008) | ✔ | ✔ | ✔ |
| Create analyses (API-009) | ✔ up to 3 / month | ✔ up to 50 / month | ✔ up to 50 / month (admins are treated as `pro`) |
| View own analyses (API-010, API-011) | ✔ partial results (FR-014) | ✔ full | ✔ full |
| Download PDF of own analysis (API-012) | ✔ partial | ✔ full | ✔ full |
| Delete own analysis (API-013) | ✔ | ✔ | ✔ |
| Export / erase own data (API-014, API-015) | ✔ | ✔ | ✔ |
| Manage job roles and skills (API-017, API-018) | ✘ | ✘ | ✔ |
| Change plan limits (API-019) | ✘ | ✘ | ✔ |
| Internal webhooks (API-020, API-021) | platform only (HMAC) | platform only | platform only |

Admins never see other users' resumes or analyses through this module (there is no admin read endpoint for user data). Support cases are handled on the platform side.

### 1.3 Logging

- Library: `log/slog` with the JSON handler. Level from `LOG_LEVEL` (`debug` in local/dev, `info` elsewhere).
- Access log: one line per request at `info` with the fields in CR-006.
- Worker log: one line per job state change (`queued`, `started`, `completed`, `failed`) with `analysis_id`, `user_id`, `attempt`, `duration_ms`, `error_code`.
- AI calls log only `model`, `input_tokens`, `output_tokens`, `latency_ms`, `http_status`. Never the prompt or completion.
- Logs are shipped from stdout to CloudWatch Logs with 30-day retention.

### 1.4 Error handling

The **one** error format used by every endpoint:

```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Monthly analysis limit of 3 reached. Resets on 2026-11-01T00:00:00+05:30.",
    "details": [],
    "request_id": "7a0f0a1c-9d9c-4d0a-8b86-2b6d2b24a9c1"
  }
}
```

| Field | Type | Rule |
|---|---|---|
| `code` | string | One of the codes in 6.3. Upper snake case. Stable; the frontend switches on it. |
| `message` | string | English, developer-oriented, may include values (limit, reset date). Never includes resume content. |
| `details` | array of `{ "field": string, "issue": string }` | Non-empty only for `VALIDATION_ERROR` and `ANALYSIS_IN_PROGRESS`; otherwise `[]`. `field` uses the JSON path (`"file"`, `"experience_level"`). |
| `request_id` | string (UUID) | Same as the `X-Request-Id` header. |

Successful responses are always wrapped: `{ "data": <object or array> }`, plus `"page"` for lists (6.1). `204 No Content` is used for deletes.

### 1.5 Security

| Topic | Rule |
|---|---|
| Secrets | CR-010. In AWS they are injected from Secrets Manager into the ECS task definition. |
| CORS | CR-011. |
| Input validation | CR-012. File validation rules are in FR-002 and are applied in this order: size → extension → magic bytes → page count → text extraction → text length. |
| Rate limiting | CR-013, implemented as a Redis sliding window (`ZADD`/`ZREMRANGEBYSCORE` on key `rl:{sub}:{bucket}`). |
| Data retention | CR-014, FR-017. |
| Encryption | CR-015. `extracted_text` is additionally encrypted at the application layer with AES-256-GCM using `DATA_ENCRYPTION_KEY` (32 bytes, base64) before being stored in PostgreSQL, so a DB dump alone does not expose resume text. |
| Object storage access | S3 bucket is private; the API never returns S3 URLs. Files are read by the server and streamed. Bucket policy denies non-TLS access. |
| AI provider | Resume text is sent to Anthropic over TLS. The account is configured with zero data retention (ZDR). No user identifier is sent; only the text and the role catalogue. |
| Uploads | Files are stored under a server-generated key (`resumes/{user_id}/{resume_id}.{pdf\|docx}`), never under the original filename. The original filename is stored only for display, sanitised to 255 characters. |
| Headers | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store` on all API responses. |

### 1.6 Internationalization (i18n)

CR-017. v1 is English only. The design keeps the door open: all UI strings in `web/messages/en.json` keyed by `errors.<CODE>` and `pages.<page>.<key>`; backend messages are not translated. The job role catalogue has a `title` column in English only; adding a language later means a `job_role_translations` table, which is out of scope.

### 1.7 Environments

| Item | local | dev | staging | prod |
|---|---|---|---|---|
| Runs on | Docker Compose on the developer machine | AWS ECS Fargate, ap-south-1 | AWS ECS Fargate, ap-south-1 | AWS ECS Fargate, ap-south-1 |
| PostgreSQL | container `postgres:16` | RDS PostgreSQL 16, single AZ | RDS PostgreSQL 16, Multi-AZ | RDS PostgreSQL 16, Multi-AZ |
| Redis | container `redis:7` | ElastiCache Redis 7, 1 node | ElastiCache Redis 7, 1 node | ElastiCache Redis 7, 2 nodes with failover |
| Object storage | MinIO container (`S3_ENDPOINT` set) | S3 bucket `sb-resume-dev` | S3 bucket `sb-resume-staging` | S3 bucket `sb-resume-prod` |
| AI provider | Real Anthropic API with a dev key, or `AI_PROVIDER=fake` which returns a fixed canned result | Real, dev key, `AI_MODEL=claude-haiku-5-5` | Real, staging key, `AI_MODEL=claude-sonnet-5-5` | Real, prod key, `AI_MODEL=claude-sonnet-5-5` |
| Platform auth | Local JWKS served from `web/dev/jwks.json` and a helper script `make dev-token` that mints a test JWT | Platform dev issuer | Platform staging issuer | Platform prod issuer |
| `LOG_LEVEL` | `debug` | `debug` | `info` | `info` |
| CORS origins | `http://localhost:3000` | `https://dev.surfingbear.com` | `https://staging.surfingbear.com` | `https://www.surfingbear.com` |
| Retention scheduler | runs, 30 / 365 days | runs | runs | runs |
| Seed data | `make seed` loads `seed/job_roles.json` and `seed/plan_limits.json` | same, run once by CI | same | same, run once at first deploy |

Configuration variables (all read in `config/config.go`, app refuses to start if a required one is missing):

| Variable | Required | Example | Used by |
|---|---|---|---|
| `APP_ENV` | yes | `local` | all |
| `HTTP_PORT` | no, default `8080` | `8080` | API |
| `DATABASE_URL` | yes | `postgres://app:pass@db:5432/resume_optimizer?sslmode=disable` | API, worker, scheduler |
| `REDIS_URL` | yes | `redis://redis:6379/0` | API, worker, scheduler |
| `S3_BUCKET` | yes | `sb-resume-dev` | API, scheduler |
| `S3_REGION` | yes | `ap-south-1` | API, scheduler |
| `S3_ENDPOINT` | no (MinIO only) | `http://minio:9000` | API, scheduler |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | local only (IAM role elsewhere) | | API, scheduler |
| `PLATFORM_JWKS_URL` | yes | `https://auth.surfingbear.com/.well-known/jwks.json` | API |
| `PLATFORM_JWT_ISSUER` | yes | `https://auth.surfingbear.com` | API |
| `PLATFORM_WEBHOOK_SECRET` | yes | 32+ random chars | API |
| `PLATFORM_UPGRADE_URL` | yes | `https://www.surfingbear.com/pricing` | API (returned in API-001), web |
| `AI_PROVIDER` | no, default `anthropic` | `anthropic` \| `fake` | worker |
| `ANTHROPIC_API_KEY` | yes unless `AI_PROVIDER=fake` | | worker |
| `AI_MODEL` | no, default `claude-sonnet-5-5` | | worker |
| `AI_TIMEOUT_SECONDS` | no, default `60` | | worker |
| `DATA_ENCRYPTION_KEY` | yes | base64 of 32 bytes | API, worker, scheduler |
| `CORS_ALLOWED_ORIGINS` | yes | `http://localhost:3000` | API |
| `LOG_LEVEL` | no, default `info` | | all |
| `RETENTION_RESUME_DAYS` | no, default `30` | | scheduler |
| `RETENTION_ANALYSIS_DAYS` | no, default `365` | | scheduler |
| `PDFTOTEXT_PATH` | no, default `pdftotext` | | API |
| `NEXT_PUBLIC_API_BASE_URL` (web) | yes | `http://localhost:8080/api/v1/resume-optimizer` | web |

### 1.8 Deployment

- **CI** (`.github/workflows/ci.yml`, on every push and PR): `go vet`, `golangci-lint run`, `go test ./... -race -cover` (coverage gate 70 % on `services/` and `repositories/`), `govulncheck`, `gitleaks`; frontend `npm ci`, `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`. Docker images `resume-optimizer-api`, `resume-optimizer-worker` (same binary, different command), `resume-optimizer-web` are built and pushed to ECR tagged with the git SHA.
- **CD** (`.github/workflows/deploy.yml`): `dev` on every merge to `main`; `staging` and `prod` by manual approval. Steps: run migration job (`golang-migrate`, `migrations/` folder, `up` only) → update ECS services (API, worker, scheduler, web) with rolling deploy, minimum healthy 100 %, health check `GET /health` → smoke test (`GET /health`, `GET /job-roles` with a test token).
- **Release process:** semantic version tag `vX.Y.Z` is created on the commit deployed to prod; `CHANGELOG.md` in the repo lists user-facing changes.
- **Rollback:** CR-020.
- **Processes:** three runtime processes from the Go binary: `server` (HTTP API), `worker` (analysis jobs), `scheduler` (retention cron). Each is an ECS service; the worker scales on queue depth (metric `asynq_queue_size`, target 20 jobs per worker task, min 1, max 10).

---

## 2. Coding conventions

### 2.1 Naming

| Item | Convention | Example |
|---|---|---|
| Go packages | lowercase, single word, no underscores | `services`, `repositories` |
| Go types / structs | PascalCase; exported only when used by another package | `AnalysisService`, `resumeRepo` |
| Go functions / methods | PascalCase exported, camelCase private; verbs first | `CreateAnalysis`, `parseDocx` |
| Go variables | camelCase, short in small scopes, descriptive otherwise | `userID`, `ctx`, `resumeText` |
| Go constants / enums | PascalCase with type prefix | `AnalysisStatusQueued`, `ErrCodeQuotaExceeded` |
| Go errors | `ErrXxx` sentinel values in `utils/apperror` | `apperror.ErrQuotaExceeded` |
| Go interfaces | noun or `-er`, defined where used | `AIClient`, `ObjectStore` |
| Go files | snake_case; tests `_test.go` | `analysis_service.go`, `analysis_service_test.go` |
| DB tables | snake_case, plural | `rewrite_suggestions` |
| DB columns | snake_case; FKs `<table_singular>_id`; booleans `is_` / `has_`; timestamps `_at` | `resume_id`, `is_active`, `completed_at` |
| DB indexes | `idx_<table>_<cols>`; unique `uq_<table>_<cols>` | `idx_analyses_user_id_created_at` |
| DB enums | PostgreSQL `text` with a `CHECK` constraint, values UPPER_SNAKE for statuses, lower_snake for categories | `status IN ('QUEUED','PROCESSING','COMPLETED','FAILED')` |
| API paths | kebab-case, plural nouns, no verbs, no trailing slash | `/job-roles/{slug}/skills` |
| JSON fields | snake_case | `experience_level`, `created_at` |
| Query params | snake_case | `page_size` |
| Error codes | UPPER_SNAKE_CASE | `RESUME_UNREADABLE` |
| Env vars | UPPER_SNAKE_CASE | `AI_TIMEOUT_SECONDS` |
| TypeScript components | PascalCase file and export | `AnalysisResult.tsx` |
| TypeScript hooks / functions | camelCase, hooks prefixed `use` | `useAnalysis`, `formatScore` |
| TypeScript types | PascalCase, no `I` prefix | `Analysis`, `MissingSkill` |
| Next.js routes | kebab-case folders under `app/resume-optimizer` | `app/resume-optimizer/analyses/[id]/page.tsx` |
| Redis keys | colon-separated, prefix per purpose | `rl:{sub}:api`, `jwks:platform`, `export:{sub}` |
| Asynq queue / task names | task type `analysis:run` on queue `analysis`; task id `analysis:{analysis_id}` | |

### 2.2 Folder / module structure

Backend repository `resume-optimizer-api` (Go module `resumeoptimizer`). Flat layered layout, same as the GoTalk backend:

```
resume-optimizer-api/
├── cmd/
│   ├── server/main.go          # HTTP API process
│   ├── worker/main.go          # asynq worker process
│   └── scheduler/main.go       # retention cron process
├── config/
│   └── config.go               # loads env vars (1.7), fails fast on missing
├── controllers/                # Gin handlers: parse request, call service, write response
│   ├── me_controller.go
│   ├── consent_controller.go
│   ├── job_role_controller.go
│   ├── resume_controller.go
│   ├── analysis_controller.go
│   ├── report_controller.go
│   ├── privacy_controller.go
│   ├── health_controller.go
│   ├── internal/               # platform webhooks (API-020, API-021)
│   │   └── platform_webhook_controller.go
│   └── admin/
│       ├── job_role_admin_controller.go
│       └── plan_limit_admin_controller.go
├── middleware/
│   ├── auth.go                 # JWT verify, loads/updates users row, sets ctx user
│   ├── require_role.go
│   ├── require_consent.go
│   ├── hmac.go                 # webhook signature (CR-003)
│   ├── rate_limit.go
│   ├── request_id.go
│   ├── logger.go
│   ├── recover.go              # panic → 500 INTERNAL_ERROR
│   └── cors.go
├── models/                     # GORM models, one file per table (section 5)
│   ├── user.go
│   ├── consent.go
│   ├── job_role.go
│   ├── role_skill.go
│   ├── resume.go
│   ├── analysis.go
│   ├── analysis_score.go
│   ├── missing_skill.go
│   ├── rewrite_suggestion.go
│   ├── usage_counter.go
│   ├── plan_limit.go
│   └── audit_log.go
├── repositories/               # DB access only, no business rules
│   ├── user_repo.go
│   ├── consent_repo.go
│   ├── job_role_repo.go
│   ├── resume_repo.go
│   ├── analysis_repo.go
│   ├── usage_repo.go
│   ├── plan_limit_repo.go
│   └── audit_repo.go
├── services/                   # business rules
│   ├── consent_service.go
│   ├── job_role_service.go
│   ├── resume_service.go       # validation, extraction, storage
│   ├── analysis_service.go     # quota, create, read with gating
│   ├── analysis_worker.go      # job handler: AI call, validation, persist
│   ├── scoring.go              # weights, overall score, response validation
│   ├── report_service.go       # PDF
│   ├── privacy_service.go      # export, erase
│   ├── retention_service.go
│   └── admin/
│       ├── job_role_admin_service.go
│       └── plan_limit_admin_service.go
├── routes/
│   └── routes.go               # all routes registered here, grouped by auth level
├── utils/
│   ├── apperror/               # ErrCode enum + Error type (section 6.3)
│   ├── response/               # {"data":...}, {"error":...} writers
│   ├── crypto/                 # AES-256-GCM helpers
│   ├── extract/                # pdf.go (pdftotext), docx.go (zip + xml)
│   ├── ai/                     # AIClient interface, anthropic.go, fake.go, prompt.go
│   ├── storage/                # ObjectStore interface, s3.go
│   ├── pdfgen/                 # report layout with go-pdf/fpdf
│   └── clock/                  # Clock interface for tests
├── migrations/                 # golang-migrate SQL files 000001_init.up.sql ...
├── seed/
│   ├── job_roles.json
│   └── plan_limits.json
├── docs/
│   └── runbook.md
├── .env.example
├── .golangci.yml
├── Dockerfile                  # multi-stage; final image includes poppler-utils
├── docker-compose.yml          # postgres, redis, minio, api, worker, scheduler
├── Makefile                    # run, test, lint, migrate, seed, dev-token
└── go.mod
```

Rules: `controllers` may import `services` and `utils`; `services` may import `repositories`, `models`, `utils`; `repositories` may import `models` only. No package imports `controllers`. Enforced by `depguard` in `.golangci.yml`.

Frontend repository `resume-optimizer-web` (Next.js 15, App Router, TypeScript strict):

```
resume-optimizer-web/
├── app/
│   ├── layout.tsx
│   ├── providers.tsx                         # TanStack Query provider
│   └── resume-optimizer/
│       ├── page.tsx                          # dashboard: usage + analyses list
│       ├── new/page.tsx                      # upload + role + level
│       ├── resumes/page.tsx                  # my resumes
│       ├── analyses/[id]/page.tsx            # result (polls while running)
│       └── privacy/page.tsx                  # export / erase my data
├── components/resume-optimizer/
│   ├── ConsentDialog.tsx
│   ├── UsageBanner.tsx
│   ├── ResumeUploader.tsx
│   ├── RoleLevelPicker.tsx
│   ├── AnalysisStatus.tsx
│   ├── ScoreGauge.tsx
│   ├── DimensionScores.tsx
│   ├── MissingSkills.tsx
│   ├── RewriteSuggestions.tsx
│   ├── LockedItems.tsx                       # "N more, upgrade" block
│   ├── AnalysisList.tsx
│   └── ErrorMessage.tsx
├── lib/
│   ├── api/
│   │   ├── client.ts                         # fetch wrapper: credentials, X-Request-Id, error mapping
│   │   ├── types.ts                          # mirrors section 6 schemas
│   │   └── resumeOptimizer.ts                # one function per API-xxx
│   └── hooks/
│       ├── useMe.ts
│       ├── useJobRoles.ts
│       ├── useResumes.ts
│       ├── useAnalysis.ts                    # refetchInterval 2000 ms while QUEUED/PROCESSING
│       └── useAnalyses.ts
├── messages/en.json
├── dev/jwks.json
├── public/
├── __tests__/
├── .eslintrc.json
├── .prettierrc
├── next.config.ts
├── tsconfig.json
└── package.json
```

### 2.3 Branching and commits

- Default branch `main`, protected: PR required, CI green, 1 approval.
- Branch names: `feature/FR-007-create-analysis`, `fix/ERR-014-file-size-message`, `chore/ci-govulncheck`, `docs/runbook`. The ID in the name must exist in this spec.
- Commit format (Conventional Commits): `<type>(<scope>): <imperative message>`; types `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`; scope is the package or page, for example `feat(analysis): enforce monthly quota (FR-013)`.
- One logical change per PR; PR title equals the squash commit title; PR description links the FR/NFR IDs.

### 2.4 Code review rules

- Every PR needs 1 approval from a maintainer who is not the author; the coordinator's own PRs are reviewed by another backend developer.
- Reviewer checklist (in the PR template): (1) every new endpoint has an `API-` block in the spec or a spec PR; (2) every new error has an `ERR-` row; (3) no personal data in logs; (4) tests cover the acceptance criteria of the FR referenced; (5) migrations are backward compatible; (6) no secret in diff.
- Max PR size 600 changed lines excluding generated files and tests; larger PRs are split.
- Review turnaround target: 1 working day.

### 2.5 Lint and format

| Tool | Config | When it runs |
|---|---|---|
| `gofmt` + `goimports` | default | pre-commit hook (`lefthook.yml`) and CI |
| `golangci-lint` v1.61+ | `.golangci.yml`: `errcheck`, `govet`, `staticcheck`, `gosec`, `revive`, `depguard`, `gocyclo` (max 15), `misspell` | pre-commit and CI; CI fails on any issue |
| `govulncheck` | default | CI |
| `gitleaks` | default | CI |
| ESLint | `.eslintrc.json`: `next/core-web-vitals`, `@typescript-eslint/recommended`, `jsx-a11y/recommended` | pre-commit (`lint-staged`) and CI |
| Prettier | `.prettierrc`: 2 spaces, single quotes, semicolons, width 100 | pre-commit and CI (`--check`) |
| `tsc --noEmit` | `tsconfig.json` strict | CI |
| Go tests | `go test ./... -race -cover` | CI, coverage gate 70 % on `services/` and `repositories/` |
| Vitest + Testing Library | `vitest.config.ts` | CI |

---

