# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Kakali Dinda |
| Version | v0.1 |
| Last updated | 2026-10-20 |
| Status | Draft |

**What this document is.** The single source for building the SurfingBear Resume Optimizer module: a NestJS backend (API + worker) and a Next.js web client, on PostgreSQL and Redis. A developer must be able to build, test and deploy the module from this file alone. Where this file and any other source disagree, this file wins; where this file is silent, that is a defect in this file and must be raised, not guessed.

**How to read it.**
- Every rule, requirement, component, table, endpoint and error has a permanent ID (`CR-`, `FR-`, `NFR-`, `CMP-`, `TBL-`, `API-`, `ERR-`). IDs are never renumbered or reused; a removed item is marked **Deprecated** and kept.
- Every number that appears in more than one place (limits, timeouts, retention) is defined **once** as a constant `K-xx` in [section 3.4](#34-configuration-constants) and referenced by name elsewhere. If a number in the text disagrees with section 3.4, section 3.4 is correct.
- "MUST" means a build that does not do it fails acceptance. "SHOULD" means it is expected unless a written reason is recorded in the pull request.
- All times are stored in UTC. "Month" for quotas means a calendar month in the `Asia/Kolkata` (IST, UTC+05:30) time zone.

---

## 1. Company-wide common requirements

Rules every Divii / Hanriver project follows, written for this module. Each rule says how it is checked.

### 1.1 Authentication

The module does **not** sign users up or log them in. The SurfingBear platform does, and gives the browser a session token. The module only verifies that token.

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-001 | Authentication | Every request to `/api/resume-optimizer/v1/*` MUST carry the platform access token in the httpOnly cookie `sb_access` (set by the platform on domain `.surfingbear.com`, `Secure`, `SameSite=Lax`). The module never reads tokens from `localStorage`, query strings or request bodies. An `Authorization: Bearer <token>` header is accepted **only** when `NODE_ENV` is `local` or `test` (for curl and automated tests). | Unit test of `AuthGuard`; e2e test that a Bearer header is rejected with `401 UNAUTHENTICATED` in `staging` config |
| CR-002 | Authentication | The token is a JWT signed with RS256 by the platform. The module verifies: signature against the platform JWKS at `K-01 PLATFORM_JWKS_URL`; `iss` equals `K-02 PLATFORM_ISSUER`; `aud` contains `surfingbear-web`; `exp` is in the future with 30 s allowed clock skew; `nbf` (if present) is in the past with 30 s skew; `sub` is a UUID. Any failure → `401 UNAUTHENTICATED` (ERR-001). The module never issues, refreshes or revokes tokens. | Unit tests with a test key pair: bad signature, wrong `iss`, wrong `aud`, expired, `sub` not UUID — each returns ERR-001 |
| CR-003 | Authentication | JWKS keys are cached in process memory for 10 minutes. When a token's `kid` is not in the cache, the module refetches JWKS immediately, but at most once per 60 seconds per process; if the `kid` is still unknown the request gets ERR-001. If the JWKS endpoint is unreachable and the cache is still valid, the cached keys are used; if the cache is empty or older than 24 hours, requests get `503 SERVICE_UNAVAILABLE` (ERR-024). | Unit test with a mocked JWKS server: rotation, outage with warm cache, outage with cold cache |
| CR-004 | Authentication | Claims the module uses: `sub` (platform user ID, UUID) → `userId`; `plan` (`"free"` or `"premium"`; a missing or unknown value is treated as `"free"`); `roles` (string array; the module only looks for `"ro_admin"`). No other claim is trusted. The user's email, name and phone are **not** read from the token and are never stored. | Code review of `AuthGuard`; unit test that a token without `plan` is treated as free |
| CR-005 | Authentication | Plan changes take effect on the next request that carries a new token. The platform access-token lifetime is 15 minutes (platform setting, not controlled here), so an upgrade is visible to this module within 15 minutes at most. | Documented assumption A-03; no test in this module |
| CR-006 | Authentication | State-changing requests (`POST`, `PUT`, `PATCH`, `DELETE`) MUST carry an `Origin` header equal to `K-03 WEB_ORIGIN`. Otherwise → `403 FORBIDDEN` (ERR-002). This is the CSRF defence (the cookie is `SameSite=Lax`, and the API is same-origin with the web app, see CR-022). | e2e test: `POST` with `Origin: https://evil.example` → 403 ERR-002 |

### 1.2 Roles and permissions

Roles come from the token (CR-004). "Plan" is not a role: it changes limits and what is shown, not which endpoints may be called.

| Role | How a request gets it | Meaning |
|---|---|---|
| `user` | Any valid token | A logged-in SurfingBear user. Free or premium is decided by the `plan` claim. |
| `ro_admin` | `roles` claim contains `"ro_admin"` | SurfingBear staff who manage the job-role catalog. An admin is also a `user` and has a personal quota like any user. |
| `system` | Internal only (worker process and scheduled jobs) | Not reachable over HTTP. |

**Role × permission table.** ✅ = allowed, ❌ = refused with `403 FORBIDDEN` (ERR-002), — = not applicable. "Own" means rows whose `user_id` equals the caller's `userId`; another user's row is answered with `404 NOT_FOUND` (ERR-004), never 403, so IDs cannot be probed.

| Permission | API | user (free) | user (premium) | ro_admin | system |
|---|---|---|---|---|---|
| List active job roles | API-001 | ✅ | ✅ | ✅ | — |
| Read own quota | API-002 | ✅ | ✅ | ✅ | — |
| Read consent status / give consent | API-003, API-004 | ✅ | ✅ | ✅ | — |
| Create an analysis (upload resume) | API-005 | ✅ (K-10 per month) | ✅ (K-11 per month) | ✅ (by own plan) | — |
| List / read own analyses | API-006, API-007 | ✅ (rewrites limited to K-14) | ✅ (all rewrites) | ✅ (by own plan) | — |
| Download own analysis PDF | API-008 | ✅ (visible content only) | ✅ | ✅ (by own plan) | — |
| Delete own analysis | API-009 | ✅ | ✅ | ✅ | — |
| Erase all own data and withdraw consent | API-010 | ✅ | ✅ | ✅ | — |
| Read **another user's** analysis | — | ❌ (404) | ❌ (404) | ❌ (404) | — |
| List all job roles incl. inactive; create / update roles; replace role terms | API-011 … API-014 | ❌ | ❌ | ✅ | — |
| Run analysis job, purge expired data | — (queue / cron) | — | — | — | ✅ |

Admins deliberately cannot read users' resumes or analyses. There is no support "view as user" feature (see 3.3 out of scope).

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-007 | Roles | Role checks are done by one `RolesGuard` using a `@Roles('ro_admin')` decorator on admin controllers. Ownership checks are done in the service layer by always adding `user_id = :userId` to the query; a repository method that reads analyses without a `userId` argument MUST NOT exist outside the `system` code paths (`AnalysisProcessor`, `RetentionJob`). | Code review; e2e test reading another user's analysis → 404 ERR-004 |

### 1.3 Logging

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-008 | Logging | Logs are JSON, one object per line, written to stdout by `pino` (via `nestjs-pino`). Required fields on every line: `time` (ISO-8601 UTC with ms), `level` (`debug`/`info`/`warn`/`error`), `service` (`ro-api`, `ro-worker` or `ro-web`), `env`, `requestId`, `msg`. When known: `userId`, `analysisId`, `route`, `status`, `durationMs`, `errorCode`. | Log sample review on staging; unit test of logger config |
| CR-009 | Logging | `requestId`: taken from the incoming `X-Request-Id` header if it matches `^[A-Za-z0-9-]{8,64}$`, otherwise a new UUIDv4. Returned in the `X-Request-Id` response header and in every error body. Copied into the queue job data so worker logs for an analysis carry the `requestId` of the request that created it. | e2e test: response header present and equal to error body `requestId` |
| CR-010 | Logging | Personal data is **never** logged: resume file bytes, extracted resume text, any line of the resume, rewrite suggestions, the original file name, email addresses, phone numbers, the raw JWT and cookies. `pino` `redact` paths MUST include `req.headers.cookie`, `req.headers.authorization`, `res.headers["set-cookie"]`, `*.resumeText`, `*.originalText`, `*.suggestedText`, `*.fileName`. AI request and response bodies are never logged, only their token counts and latency. | Unit test that logs a fake request with all of these and asserts `[Redacted]`; CI job `log-pii-scan` greps staging logs for `@` followed by a domain and for 10-digit numbers and fails on any hit |
| CR-011 | Logging | Levels: `error` = a request returned 5xx or a job failed for a system reason; `warn` = a retry happened, a circuit breaker opened, a 4xx that indicates abuse (429); `info` = one line per request (method, route template, status, durationMs) and one per job state change; `debug` = off in `staging` and `prod`. | Code review |
| CR-012 | Logging | Retention of logs: 30 days in CloudWatch Logs, then deleted. | Infrastructure-as-code review (log group `retention_in_days = 30`) |

### 1.4 Error handling

**The one error format.** Every error response from every endpoint, including 404 for unknown routes and 500, has `Content-Type: application/json` and this body:

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "The file is larger than 5 MB.",
    "details": [
      { "field": "file", "issue": "max_size", "limit": 5242880 }
    ],
    "requestId": "8f14e45f-ceea-467a-9575-5a2c9f2a1b11"
  }
}
```

| Field | Type | Always present | Meaning |
|---|---|---|---|
| `error.code` | string | yes | Stable machine code from the error list (section 6.3). Clients branch on this, never on `message`. |
| `error.message` | string | yes | Human-readable text in the request locale (CR-019), safe to show to the user. Never contains stack traces, SQL, file paths or personal data. |
| `error.details` | array of objects | yes (may be empty `[]`) | Extra structured data. For `VALIDATION_FAILED`: one object per invalid field `{ "field": string, "issue": string, "limit"?: number|string }`. For `QUOTA_EXCEEDED`: `[{ "limit": number, "used": number, "resetsAt": string }]`. Other codes use `[]` unless section 6.3 says otherwise. |
| `error.requestId` | string | yes | Same as the `X-Request-Id` response header. |

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-013 | Errors | All errors are thrown as `AppException(code, details?)` and turned into the format above by one global `AppExceptionFilter`. The filter maps `code` → HTTP status and message key using the table in section 6.3; a code not in that table is a build failure (the code list is a TypeScript `enum ErrorCode` and the map is `Record<ErrorCode, …>`, so a missing entry does not compile). | Unit test per error code; TypeScript compile |
| CR-014 | Errors | Unknown exceptions are logged at `error` with the stack trace and returned as `500 INTERNAL_ERROR` (ERR-019) with the generic message. NestJS `HttpException`s thrown by the framework (e.g. body parser) are mapped: 400 → `VALIDATION_FAILED`, 404 → `NOT_FOUND`, 413 → `FILE_TOO_LARGE`, 415 → `UNSUPPORTED_FILE_TYPE`. | e2e test: unknown route → 404 in standard format |
| CR-015 | Errors | Failures that happen after the HTTP request has returned (inside the analysis job) are not HTTP errors. They are stored on the analysis as `status = "failed"` plus `failure_code` (one of the codes marked "job" in section 6.3) and shown by the web client from API-007. | e2e test with a mocked AI outage |

### 1.5 Security

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-016 | Secrets | Secrets (`DATABASE_URL`, `REDIS_URL`, `ANTHROPIC_API_KEY`, `RESUME_TEXT_KEY`) are stored in AWS Secrets Manager and injected as environment variables by ECS at task start. They are never committed, never in `.env` files outside `local`, never logged. `.env*` (except `.env.example`) is in `.gitignore`. The app refuses to start (exit code 1, one `error` log line naming the missing variable, not its value) if any required variable from section 4.6 is missing or malformed. | `gitleaks` in CI (2.5); unit test of config schema |
| CR-017 | CORS | The API is same-origin with the web app (CR-022), so CORS is **disabled**: the API sends no `Access-Control-Allow-*` headers for any origin. Preflight `OPTIONS` requests get `404`. | e2e test: `OPTIONS` with `Origin: https://evil.example` → no `Access-Control-Allow-Origin` header |
| CR-018 | Input validation | Every request body, query and path parameter is validated by a DTO class with `class-validator` and the global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`. Unknown fields → `400 VALIDATION_FAILED`. Path IDs must be UUIDs (`ParseUUIDPipe`), otherwise `400 VALIDATION_FAILED`. Uploaded files are validated by content (magic bytes), not by extension or `Content-Type` header (FR-002). | e2e tests per endpoint with an unknown field and a bad UUID |
| CR-019 | Rate limiting | Per user (`userId`), counted in Redis with a fixed 60-second window: at most `K-20` requests per minute across all endpoints, and at most `K-21` calls to API-005 per hour. Over the limit → `429 RATE_LIMITED` (ERR-014) with header `Retry-After: <seconds>`. If Redis is down, rate limiting fails **open** (request allowed, `warn` logged) because quota (FR-008) is enforced in PostgreSQL. | e2e test: 61st request in a minute → 429 with `Retry-After` |
| CR-020 | Data retention | Personal data is kept only as long as section 3.4 says: original files are never stored (K-30); extracted text is purged after K-31; whole analyses are deleted after K-32; deleted on user request immediately (FR-015, FR-016). | Retention job tests (FR-017) |
| CR-021 | Transport and headers | HTTPS only (TLS 1.2+ at the load balancer; HTTP redirects to HTTPS). API responses carry `Strict-Transport-Security: max-age=31536000; includeSubDomains`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store` (except API-001, see there), `Referrer-Policy: no-referrer`. Set with `helmet`. The web app sets a Content-Security-Policy: `default-src 'self'; img-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline'; connect-src 'self'; frame-ancestors 'none'`. | e2e header assertions; `curl -I` on staging in the release checklist |
| CR-022 | Topology | The web app is served at `https://<host>/resume-optimizer` and the API at `https://<host>/api/resume-optimizer/v1` on the **same host** (`K-03 WEB_ORIGIN`), routed by path at the load balancer. This keeps the platform cookie first-party and removes the need for CORS. | Infrastructure review |
| CR-023 | Encryption | Data at rest: RDS storage encryption (AWS KMS) and ElastiCache encryption at rest and in transit are on. In addition, the extracted resume text column is encrypted by the application with AES-256-GCM (format in TBL-004 `resume_text_enc`), so DB backups and DB read access alone do not reveal resume text. | Infra review; unit test round-trip of the cipher |
| CR-024 | Dependencies | `npm audit --audit-level=high --omit=dev` must pass in CI; Renovate opens dependency PRs weekly; a `high` or `critical` advisory in a runtime dependency must be fixed or have a written exception within 7 days. | CI job `audit` |

### 1.6 Internationalization (i18n)

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-025 | i18n | Supported UI locales: `en` (default) and `hi` (Hindi). The web client chooses the locale from the platform cookie `sb_locale` (`en`/`hi`); anything else → `en`. It sends `Accept-Language: <locale>` on every API call. The API uses the first supported tag in `Accept-Language` for `error.message` and for localized fields (`JobRole.name`); otherwise `en`. | e2e: `Accept-Language: hi` returns a Hindi error message |
| CR-026 | i18n | All user-visible strings live in message files: web `apps/web/messages/{en,hi}.json` (`next-intl`), API `apps/api/src/i18n/{en,hi}.json` keyed by error code. No user-visible string literals in components or services. A CI check fails if a key exists in `en.json` but not in `hi.json`. | CI script `check-i18n-keys` |
| CR-027 | i18n | Content that is **not** translated: the resume itself, AI-generated rewrites and explanations (always English, because only English resumes are supported, A-05), skill/term names from the catalog, and the PDF report (English only in this version, see 3.3). | Documented scope |
| CR-028 | i18n | Dates shown to users are formatted in the browser with `Intl.DateTimeFormat(locale, { dateStyle: "medium", timeStyle: "short", timeZone: "Asia/Kolkata" })`. Numbers use `Intl.NumberFormat(locale)`. The API always returns ISO-8601 UTC strings (6.1). | Component test |

### 1.7 Environments

| Item | `local` | `dev` | `staging` | `prod` |
|---|---|---|---|---|
| Purpose | Developer laptop | Integration of merged `main` | Release candidate; same infra shape as prod | Real users |
| Deployed from | `docker compose up` | Every merge to `main` (automatic) | Every tag `vX.Y.Z-rc.N` (automatic) | Tag `vX.Y.Z` after manual approval |
| `K-03 WEB_ORIGIN` | `http://localhost:3000` | `https://dev.surfingbear.com` | `https://staging.surfingbear.com` | `https://www.surfingbear.com` |
| Auth | Local JWKS from `tools/dev-auth` (signs test tokens; Bearer allowed, CR-001) | Platform dev JWKS | Platform staging JWKS | Platform prod JWKS |
| AI provider | `AI_MODE=fake` (deterministic fake from `test/fixtures/ai/`) unless the developer sets a key | Real API, separate key, `K-50` monthly spend cap $50 | Real API, separate key, cap $200 | Real API, prod key |
| PostgreSQL | Docker `postgres:16` | RDS `db.t4g.small` single-AZ | RDS `db.t4g.medium` Multi-AZ | RDS `db.m7g.large` Multi-AZ |
| Redis | Docker `redis:7` | ElastiCache `cache.t4g.micro`, 1 node | ElastiCache `cache.t4g.small`, primary + replica | ElastiCache `cache.m7g.large`, primary + replica |
| Data | Seed data only | Seed data; wiped on every deploy of a new migration baseline | Seed data + synthetic resumes; **no real user data** | Real data |
| Log level | `debug` | `debug` | `info` | `info` |
| Retention job | Manual (`npm run job:retention`) | Daily | Daily | Daily |

Real resumes MUST NOT be copied out of `prod` into any other environment (CR-029 below).

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-029 | Environments | No production personal data in non-production environments. Bugs are reproduced with the synthetic resumes in `test/fixtures/resumes/`. | Access review: only the on-call role can read prod DB |
| CR-030 | Environments | Configuration differs **only** through environment variables listed in section 4.6. No `if (env === 'prod')` branches in code except in the config module. | Code review; `grep` check in CI for `NODE_ENV ===` outside `src/config/` |

### 1.8 Deployment

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-031 | CI | GitHub Actions on every pull request: install → lint → format check → typecheck → unit tests → integration tests (Testcontainers PostgreSQL 16 + Redis 7) → build Docker images → `npm audit` → `gitleaks`. All must pass before merge. Coverage gate: 80 % lines for `apps/api/src/**` (excluding `main.ts`, `*.module.ts`). | Branch protection on `main` requires the `ci` workflow |
| CR-032 | Build | One Docker image per app: `ro-api` (also used for the worker with command `node dist/worker.js`, and for migrations with `node dist/migrate.js`) and `ro-web`. Images are tagged with the git SHA and pushed to Amazon ECR. Base image `node:22-bookworm-slim`, non-root user `node`. | Dockerfile review |
| CR-033 | Release | Release = git tag. `vX.Y.Z-rc.N` deploys to staging; `vX.Y.Z` on the same commit deploys to prod after one approval from a tech lead in the GitHub `production` environment. Versioning is SemVer; API breaking changes require a new `/v2` path, never a change in `/v1`. | GitHub environment protection rules |
| CR-034 | Deploy order | Per deploy: (1) run migrations as a one-off ECS task `node dist/migrate.js` (Prisma `migrate deploy`); stop if it fails; (2) rolling update of `ro-worker`, then `ro-api`, then `ro-web` (ECS rolling, minimum healthy 100 %, maximum 200 %). Migrations MUST be backward-compatible with the previous app version (expand → migrate → contract over two releases); a column is dropped only in a release after the code stopped using it. | Migration review checklist in PR template |
| CR-035 | Rollback | ECS deployment circuit breaker with automatic rollback is on: if new tasks fail `/health/ready` (API-016) for 3 consecutive checks, ECS rolls back to the previous task definition. Manual rollback = redeploy the previous tag (GitHub Actions `deploy` workflow with input `tag`). Database migrations are not rolled back automatically; because of CR-034 the previous app version runs on the new schema. | Staging rollback drill once per quarter |
| CR-036 | Post-deploy | After a prod deploy the pipeline runs the smoke test `npm run smoke -- --env=prod`: GET API-015, GET API-016, GET API-001 with a synthetic test-user token. Failure pages the on-call engineer and triggers manual rollback. | Pipeline log |

---

## 2. Coding conventions

Repository: one monorepo `surfingbear-resume-optimizer`, npm workspaces, Node.js 22 LTS, TypeScript 5.x with `"strict": true`.

### 2.1 Naming

| Item | Convention | Example |
|---|---|---|
| Files (TS) | kebab-case, with a NestJS role suffix | `analysis.service.ts`, `create-analysis.dto.ts`, `jwt-auth.guard.ts` |
| Classes | PascalCase; suffix by role (`Controller`, `Service`, `Repository`, `Guard`, `Processor`, `Dto`, `Exception`) | `AnalysisService`, `CreateAnalysisDto` |
| Interfaces / types | PascalCase, **no** `I` prefix | `ScoringResult`, `RewriteSuggestion` |
| Variables / functions | camelCase; functions start with a verb; booleans start with `is`/`has`/`can` | `extractText()`, `isLocked`, `hasConsent` |
| Constants | UPPER_SNAKE_CASE; the K-constants of section 3.4 keep their names | `FREE_MONTHLY_LIMIT`, `MAX_FILE_BYTES` |
| Enums | PascalCase name, UPPER_SNAKE members whose string value equals the DB / API value | `enum AnalysisStatus { QUEUED = 'queued' }` |
| Error codes | UPPER_SNAKE_CASE, a noun phrase of the problem | `FILE_TOO_LARGE`, `QUOTA_EXCEEDED` |
| DB tables | snake_case, plural | `analyses`, `rewrite_suggestions` |
| DB columns | snake_case; timestamps end in `_at`; foreign keys `<singular>_id`; booleans `is_`/`has_` | `created_at`, `job_role_id`, `is_active` |
| DB indexes / constraints | `<table>_<columns>_<kind>` with kind `idx`, `uniq`, `fkey`, `check` | `analyses_user_id_created_at_idx` |
| API paths | lowercase, kebab-case, plural nouns, no verbs, no trailing slash | `/api/resume-optimizer/v1/job-roles/{jobRoleId}` |
| JSON fields | camelCase | `overallScore`, `createdAt` |
| Query parameters | camelCase | `?pageSize=20` |
| Queue names / job names | kebab-case | queue `resume-analysis`, job `analyze` |
| Redis keys | `ro:<purpose>:<id>` with colons | `ro:rl:min:<userId>:<epochMinute>` |
| React components | PascalCase file and export | `ScoreCard.tsx` |
| Env variables | UPPER_SNAKE_CASE, `RO_` prefix only for module-specific tunables | `DATABASE_URL`, `RO_AI_TIMEOUT_MS` |

### 2.2 Folder / module structure

```text
surfingbear-resume-optimizer/
├── apps/
│   ├── api/                          # NestJS 11 — HTTP API and worker share this codebase
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/           # generated SQL, edited by hand only for partial indexes / checks
│   │   │   └── seed.ts               # job-role catalog seed (section 5.5)
│   │   ├── src/
│   │   │   ├── main.ts               # HTTP entrypoint
│   │   │   ├── worker.ts             # BullMQ worker entrypoint (no HTTP listener)
│   │   │   ├── migrate.ts            # runs `prisma migrate deploy` then exits
│   │   │   ├── app.module.ts
│   │   │   ├── config/               # zod-validated env config (section 4.6); only place reading process.env
│   │   │   ├── common/
│   │   │   │   ├── auth/             # JwtAuthGuard, RolesGuard, OriginGuard, @CurrentUser()
│   │   │   │   ├── errors/           # ErrorCode enum, AppException, AppExceptionFilter, error map
│   │   │   │   ├── i18n/             # en.json, hi.json, locale resolver
│   │   │   │   ├── logging/          # pino config and redaction
│   │   │   │   ├── rate-limit/       # RateLimitGuard (Redis)
│   │   │   │   └── crypto/           # TextCipher (AES-256-GCM)
│   │   │   ├── modules/
│   │   │   │   ├── job-roles/        # catalog read + admin CRUD (CMP-004)
│   │   │   │   ├── consent/          # consent records (CMP-005)
│   │   │   │   ├── quota/            # usage counters (CMP-006)
│   │   │   │   ├── analyses/         # upload, list, read, delete, erase (CMP-003)
│   │   │   │   ├── parsing/          # PDF/DOCX text extraction (CMP-007)
│   │   │   │   ├── scoring/          # deterministic scorers + AI client + processor (CMP-008, CMP-009)
│   │   │   │   ├── report/           # PDF report renderer (CMP-010)
│   │   │   │   ├── retention/        # scheduled purge + stuck-job sweeper (CMP-011)
│   │   │   │   └── health/           # liveness / readiness
│   │   │   └── prisma/               # PrismaService
│   │   └── test/
│   │       ├── unit/                 # mirrors src/ paths
│   │       ├── integration/          # Testcontainers
│   │       ├── e2e/                  # supertest against the full app
│   │       └── fixtures/
│   │           ├── resumes/          # synthetic PDF/DOCX files (section 3.5)
│   │           └── ai/               # recorded fake AI responses
│   └── web/                          # Next.js 15 App Router
│       ├── messages/                 # en.json, hi.json
│       ├── src/
│       │   ├── app/resume-optimizer/
│       │   │   ├── page.tsx                      # upload + history (W-01)
│       │   │   ├── analyses/[analysisId]/page.tsx  # result (W-02)
│       │   │   └── admin/job-roles/page.tsx      # catalog admin (W-03)
│       │   ├── components/           # ScoreCard, DimensionBar, MissingSkills, RewriteList, UploadForm, ConsentDialog, QuotaBadge
│       │   ├── lib/api/              # typed fetch client, one function per API-xxx
│       │   └── lib/i18n/
│       └── test/                     # Vitest + React Testing Library; Playwright e2e in test/e2e
├── packages/
│   └── contracts/                    # shared TS types for API DTOs and ErrorCode (imported by api and web)
├── infra/                            # Terraform (ECS, RDS, ElastiCache, ALB, Secrets Manager)
├── tools/dev-auth/                   # local JWKS + token signer for `local`
├── docker-compose.yml
├── .github/workflows/                # ci.yml, deploy.yml
└── package.json                      # npm workspaces
```

Rules: a module may import another module only through that module's exported service (no reaching into another module's repository). `common/` must not import from `modules/`. Prisma is used only inside `*.repository.ts` files.

### 2.3 Branching and commits

| Item | Rule | Example |
|---|---|---|
| Default branch | `main`, protected; no direct pushes | — |
| Branch names | `<type>/<ID>-<short-kebab-description>` where `<ID>` is the FR/NFR/ERR/CR the work is for, or the issue number | `feature/FR-005-upload-analysis`, `fix/ERR-015-ai-retry`, `chore/deps-weekly` |
| Branch types | `feature`, `fix`, `chore`, `docs`, `refactor`, `test` | — |
| Commit format | Conventional Commits: `<type>(<scope>): <imperative summary>`, summary ≤ 72 chars, lowercase start, no trailing period. Scope = module folder name. Body (optional) explains *why*. Footer `Refs: FR-005` lists requirement IDs. | `feat(analyses): reject encrypted PDFs with FILE_ENCRYPTED` |
| Commit types | `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `build`, `ci` | — |
| Enforcement | `commitlint` (`@commitlint/config-conventional`) in a `commit-msg` hook (husky) and in CI on the PR title | — |
| Merge | Squash-merge; the PR title becomes the commit message and must follow the commit format | — |

### 2.4 Code review rules

| ID | Rule |
|---|---|
| CV-01 | Every PR needs 1 approval from a team member who did not write it; PRs touching `common/auth`, `common/crypto`, `retention/` or `prisma/migrations/` need 2, one of them a tech lead. |
| CV-02 | PR description uses the template: *What*, *Why*, *Requirement IDs*, *How tested*, *Migration? (yes/no + backward-compatible?)*, *Personal data touched? (yes/no)*. |
| CV-03 | A PR that changes behaviour listed in this spec must update this spec in the same PR (or link the spec PR). |
| CV-04 | Reviewer checklist: no personal data in logs (CR-010); new endpoints have DTO validation (CR-018) and an ownership filter (CR-007); new errors are in `ErrorCode` and section 6.3; new queries have a matching index (section 5.3); tests cover each acceptance criterion touched. |
| CV-05 | PRs over 400 changed lines (excluding generated files and fixtures) are split unless the reviewer agrees in writing. |
| CV-06 | Review turnaround: first response within 1 working day. |

### 2.5 Lint and format

| Tool | Config file | What it enforces | When it runs |
|---|---|---|---|
| ESLint 9 (flat config) with `typescript-eslint` (strict-type-checked), `eslint-plugin-import`, `@next/eslint-plugin-next` (web) | `eslint.config.mjs` at repo root | No `any` (`@typescript-eslint/no-explicit-any: error`), no floating promises, import order, no `console.*` (use the logger), no default exports except Next.js pages/layouts | `lint-staged` pre-commit on staged files; CI `npm run lint` on all files |
| Prettier 3 | `.prettierrc` = `{ "singleQuote": true, "trailingComma": "all", "printWidth": 100, "semi": true }` | Formatting | `lint-staged` pre-commit; CI `npm run format:check` |
| TypeScript | `tsconfig.base.json` with `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` | Types | CI `npm run typecheck` |
| commitlint | `commitlint.config.cjs` | Commit messages (2.3) | `commit-msg` hook; CI on PR title |
| gitleaks | `.gitleaks.toml` (default rules) | No secrets in the diff | CI |
| Prisma format | — | `schema.prisma` formatting | CI `npx prisma format --check` |
| markdownlint | `.markdownlint.json` | Docs | CI on `*.md` |

---

## 3. Requirements definition

_To be written in the next revision._

---

## 4. Architecture

_To be written._

---

## 5. ERD and database design

_To be written._

---

## 6. API design

_To be written._

---

## 7. Traceability matrix

_To be written._

---

## 8. Change log

| Version | Date | Author | What changed | Why | Affected IDs |
|---|---|---|---|---|---|
| v0.1 | 2026-10-20 | Kakali Dinda | Sections 1 (common requirements) and 2 (coding conventions) written | First draft | CR-001 … CR-036 |
