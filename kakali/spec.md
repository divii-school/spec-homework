# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Kakali Dinda |
| Version | v0.2 |
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
│   │           ├── resumes/          # synthetic PDF/DOCX files (section 3.7)
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

### 3.1 Functional requirements

Priorities: **Must** = needed for launch; **Should** = launch unless it slips, then next release; **Could** = nice to have. Every acceptance criterion (AC) is written so that a test can pass or fail it. Error codes refer to section 6.3; constants `K-xx` refer to section 3.4.

| ID | Title | Description | Priority | Acceptance criteria |
|---|---|---|---|---|
| FR-001 | Choose target role and level | Before uploading, the user picks one **target job role** from the active catalog and one **experience level**. | Must | 1. API-001 returns only roles with `is_active = true`, sorted by localized name ascending (locale per CR-025). 2. Experience level is one of `fresher` (0–1 yr), `junior` (1–3 yrs), `mid` (3–6 yrs), `senior` (6–10 yrs), `lead` (10+ yrs); any other value on API-005 → `400 VALIDATION_FAILED` with `details[0].field = "experienceLevel"`. 3. A `jobRoleId` that does not exist → `404 NOT_FOUND`; one that exists but is inactive → `422 JOB_ROLE_INACTIVE`. 4. The web upload form (W-01) cannot be submitted until a role, a level and a file are chosen (submit button disabled). |
| FR-002 | Validate uploaded file | The user uploads one resume file in PDF or DOCX. The system checks the file before doing anything else. | Must | Checks run in this order and the **first** failing check decides the error: 1. No `file` part → `400 VALIDATION_FAILED` (`field: "file"`, `issue: "required"`). 2. Size > `K-04` (5 MiB = 5,242,880 bytes) → `413 FILE_TOO_LARGE`; the upload stream is cut at `K-04` + 1 bytes so larger files are never fully read. 3. Type is decided by magic bytes only: PDF = file starts with `%PDF-`; DOCX = file starts with `PK\x03\x04` **and** the ZIP contains `[Content_Types].xml` and `word/document.xml`. Anything else (including `.doc`, `.docm`, images, a PNG renamed `.pdf`) → `415 UNSUPPORTED_FILE_TYPE`. The file extension and the multipart `Content-Type` are ignored. 4. PDF protected by a password (open or owner password that blocks text extraction) → `422 FILE_ENCRYPTED`. 5. PDF/DOCX that the parser cannot open (corrupt, truncated, ZIP bomb: total uncompressed DOCX size > 20 MiB) → `422 FILE_UNREADABLE`. 6. PDF with more than `K-05` (10) pages → `422 TOO_MANY_PAGES`. 7. All checks pass for fixtures in section 3.7 with the expected outcome listed there. |
| FR-003 | Extract and prepare resume text | The system extracts plain text from the file, checks it is usable, removes direct contact details, and stores only the cleaned text. | Must | 1. PDF text is extracted with `pdfjs-dist` page by page, joining text items in reading order, one `\n` between lines and `\n\n` between pages; DOCX with `mammoth.extractRawText` (paragraphs joined by `\n`). Embedded images are ignored (no OCR). 2. Text is normalized: Unicode NFKC; `\r\n` and `\r` → `\n`; tabs → space; runs of spaces → one space; more than 2 consecutive blank lines → 2. 3. If the normalized text has fewer than `K-06` (300) non-whitespace characters → `422 NOT_ENOUGH_TEXT` (typical for scanned resumes). 4. If more than `K-07` (40,000) characters → `422 RESUME_TOO_LONG`. 5. Language check: of all Unicode letters (`\p{L}`), at least 80 % must be ASCII `A–Z`/`a–z`, otherwise `422 RESUME_LANGUAGE_UNSUPPORTED`. 6. Before redaction, the system records `has_email` and `has_phone` (regexes in 3.5.4). 7. Redaction: every email → `[EMAIL]`, every phone number → `[PHONE]`, every URL (`https?://…`, `www.…`, and `linkedin.com/…`, `github.com/…` without scheme) → `[URL]`. 8. Only the redacted text is stored (encrypted, CR-023) and only it is sent to the AI provider. The original file bytes and un-redacted text exist only in process memory for the duration of the request and are never written to disk, object storage, the database, the queue or logs (NFR-008). 9. Steps 1–7 run inside a worker thread pool (`piscina`, 2 threads per API task) so the HTTP event loop is not blocked; extraction that takes longer than 10 s is aborted → `422 FILE_UNREADABLE`. |
| FR-004 | Consent before processing | Because resumes contain personal data (DPDP Act 2023), the user must accept the current privacy notice before the first upload, and again whenever the notice version changes. | Must | 1. API-005 by a user with no active consent for version `K-08` → `403 CONSENT_REQUIRED`; nothing is parsed or stored. 2. API-004 with `noticeVersion = K-08` creates an active consent; with any other version → `409 CONSENT_VERSION_OUTDATED`. 3. Giving consent when an active consent for `K-08` already exists returns `200` with the existing record (idempotent, no new row). 4. When `K-08` changes in a release, all existing consents for older versions stop counting (API-003 returns `hasValidConsent: false`) and the web client shows the consent dialog (W-01) again. 5. The notice text (section 3.6, `ConsentDialog`) states: what is collected (resume content), purpose (scoring and suggestions only), that the redacted text is sent to Anthropic (USA) for AI processing, retention (`K-31`, `K-32`), the user's rights (access, erasure, withdraw consent, grievance), and the grievance contact `grievance@surfingbear.com`. 6. Withdrawing consent is done through FR-016. |
| FR-005 | Start an analysis | The user submits role + level + file; the system validates synchronously, creates the analysis, and processes it in the background. | Must | 1. On success API-005 returns `202 Accepted` with the analysis in status `queued` within NFR-001. 2. Order of checks: auth → origin (CR-006) → rate limit (CR-019) → consent (FR-004) → AI circuit breaker (FR-013 AC 5) → body fields (FR-001) → in-progress limit (AC 3) → file (FR-002, FR-003) → quota (FR-008). 3. If the user already has `K-12` (1) analysis in `queued` or `processing` → `409 ANALYSIS_IN_PROGRESS`. 4. Quota reservation and the `analyses` insert happen in **one** DB transaction (FR-008 AC 3). 5. After commit, a BullMQ job `analyze` is added to queue `resume-analysis` with `jobId = analysisId` and data `{ analysisId, requestId }`. If adding the job fails, the analysis is set to `failed` with `failure_code = INTERNAL_ERROR`, the reservation is released, and the request returns `503 SERVICE_UNAVAILABLE`. 6. Status lifecycle: `queued` → `processing` → `completed` or `failed`. No other transitions. `processing` is set by the worker when it starts the job (`started_at` set). 7. The worker writes results and `completed` in **one** transaction, and only if the row is still `processing` (`UPDATE … WHERE id = $1 AND status = 'processing'`); if 0 rows change (deleted or timed out meanwhile) the results are discarded. |
| FR-006 | Deterministic scoring | Four of the six score dimensions are computed by code, not by AI, so that they are repeatable and explainable. | Must | 1. Dimensions `keyword_match`, `skills_coverage`, `structure_completeness`, `formatting_parseability` are computed exactly as in 3.5.1–3.5.3. 2. Each dimension score is an integer 0–100. 3. Each dimension gets a one-sentence English explanation built from the template in 3.5.5 (no AI). 4. Running the scorers twice on the same text, role terms and level gives identical scores (NFR-017). |
| FR-007 | AI scoring and rewrites | Two dimensions (`experience_relevance`, `impact_quantification`) and the rewrite suggestions are produced by one AI call. | Must | 1. One call to the Anthropic Messages API per analysis, built exactly as in 4.7 (model `K-46`, effort `K-48`, structured JSON output). 2. The response must validate against the JSON schema in 4.7; each AI dimension score must be an integer 0–100 with an explanation of 1–300 characters. 3. A response that fails validation counts as a failed attempt (FR-013). 4. Rewrites are post-validated by FR-011 before storing. 5. Model ID actually used (`response.model`), prompt version (`K-49`) and input/output token counts are stored on the analysis. |
| FR-008 | Monthly analysis quota | Free users get `K-10` (3) analyses per month; premium users `K-11` (50). Only successful analyses use up the quota. | Must | 1. Period = calendar month in IST, written `YYYY-MM` (e.g. an analysis created 2026-10-31T19:00:00Z = 2026-11-01 00:30 IST counts to `2026-11`). 2. Limit is taken from the `plan` claim at the moment of creation. 3. In the API-005 transaction: upsert `usage_counters(user_id, period)`, lock it (`SELECT … FOR UPDATE`), and if `used + reserved >= limit` → `429 QUOTA_EXCEEDED` with `details: [{ "limit", "used", "resetsAt" }]` (`resetsAt` = first instant of next IST month, in UTC, e.g. `2026-10-31T18:30:00.000Z`); otherwise `reserved += 1`. 4. On `completed`: `reserved -= 1`, `used += 1` in the same transaction as the results. 5. On `failed` (any reason) or on erase of an in-progress analysis: `reserved -= 1` only. 6. Deleting a completed analysis does **not** give the quota back. 7. A premium user who downgrades mid-month keeps `used`; if `used >= K-10` they cannot start more that month. 8. Two parallel API-005 calls by a free user with `used = 2` result in exactly one `202` and one `409 ANALYSIS_IN_PROGRESS` or `429 QUOTA_EXCEEDED` (integration test with real PostgreSQL). |
| FR-009 | Overall score and result | When the analysis is complete the user sees the overall score out of 100, the band, and all six dimension scores with explanations. | Must | 1. `overallScore = round_half_up( Σ(score_d × weight_d) / 100 )` with the weights in 3.5 (sum 100). 2. Band: 0–49 `needs_work`, 50–74 `fair`, 75–89 `good`, 90–100 `excellent`. 3. API-007 returns dimensions always in this order: `keyword_match`, `skills_coverage`, `experience_relevance`, `impact_quantification`, `structure_completeness`, `formatting_parseability`. 4. Free and premium users see the same scores and explanations (only rewrites differ, FR-012). |
| FR-010 | Missing skills | The system lists catalog skills for the chosen role that were not found in the resume. | Must | 1. Missing skills = role terms of type `core` and `preferred` that do not match the resume text under the matching rule in 3.5.1. Terms of type `keyword` are not listed (they only feed `keyword_match`). 2. Order: all `core` first, then `preferred`; within each group by `term` ascending (case-insensitive). 3. Each item has `term` and `importance` (`core`/`preferred`). 4. The list is stored as a snapshot (TBL-006) so later catalog changes do not change old results. 5. Free and premium users see the full list. 6. If nothing is missing the list is empty and W-02 shows "No missing skills found for this role." |
| FR-011 | Rewrite suggestions | The system suggests better wording for up to `K-13` (10) weak lines, without inventing facts. | Must | After the AI returns, each suggestion is checked in this order and **dropped** (not stored) if any check fails: 1. `originalText`, after collapsing whitespace, is an exact substring of the whitespace-collapsed stored (redacted) resume text. 2. Lengths: `originalText` 20–500 chars, `suggestedText` 20–600 chars. 3. Every run of digits in `suggestedText` also appears as the same run in `originalText` (no invented numbers); the AI must write `[X]` where a number should be added, and `[X]` is allowed. 4. `suggestedText` differs from `originalText` (case-insensitive, whitespace-collapsed). 5. `reason` is one of `missing_metric`, `weak_verb`, `vague`, `too_long`, `passive_voice`, `missing_keyword`. 6. Duplicate `originalText` → keep the first. Survivors are numbered `rank` 1…n in the AI's order. 0 survivors is valid; the analysis still completes. The number of dropped suggestions is stored (`rewrites_dropped_count`) for monitoring. |
| FR-012 | Free vs premium suggestions | Free users see only some suggestions. | Must | 1. Visibility is decided at **read** time from the caller's current `plan` claim (so upgrading unlocks old analyses). 2. Free: suggestions with `rank <= K-14` (3) are returned in full; suggestions with `rank > K-14` are returned as `{ rank, reason, isLocked: true, originalText: null, suggestedText: null }`. 3. Premium: all suggestions with `isLocked: false`. 4. Locked text is never sent to the browser (checked by e2e test on the raw JSON). 5. W-02 shows locked items as blurred placeholder cards with the text "Upgrade to Premium to see this suggestion" and a link to `K-09 UPGRADE_URL`. 6. The same rule applies to the PDF report (FR-018). |
| FR-013 | AI failure handling | AI timeouts, errors and bad output must never leave an analysis stuck or charge the user. | Must | 1. Each attempt has a timeout of `K-40` (60 s). 2. Retry on: timeout, network error, HTTP 429, HTTP 5xx, HTTP 529, invalid JSON or schema mismatch. No retry on other 4xx (logged as `error`). 3. At most `K-41` (3) attempts with waits of 5 s then 15 s (`K-42`), honouring a `retry-after` header if it is longer, up to 30 s. 4. After the last failed attempt: status `failed`, `failure_code = AI_UNAVAILABLE` (timeouts, network, 429, 5xx) or `AI_BAD_RESPONSE` (invalid output, `stop_reason = "refusal"` or `"max_tokens"`); quota reservation released (FR-008 AC 5). 5. Circuit breaker shared across all worker tasks in Redis: when `K-43` (10) failed **attempts** happen within 120 s the breaker opens for 60 s. While open: API-005 returns `503 AI_UNAVAILABLE` before reading the file and without reserving quota; jobs that start while open fail at once with `AI_UNAVAILABLE`. After 60 s one job is let through (half-open); success closes the breaker, failure opens it for another 60 s. 6. W-02 shows, for `failed`: the localized message of `failure_code` and a "Try again" button that goes back to W-01 (the user must re-upload, since the file was not stored). |
| FR-014 | Analysis history | The user can see their past analyses. | Must | 1. API-006 returns the caller's analyses newest first (`created_at DESC, id DESC`), paginated (6.1), each with `id`, `status`, `jobRole` (`id`, `name`), `experienceLevel`, `overallScore` (null unless completed), `band`, `fileName`, `createdAt`, `completedAt`. 2. Optional filter `status`. 3. Analyses deleted by the user or by retention do not appear. 4. W-01 shows the first page under the upload form with "Load more". |
| FR-015 | Delete one analysis | The user can delete an analysis at any time. | Must | 1. API-009 hard-deletes the analysis and its child rows (cascade) and returns `204`. 2. Deleting an analysis in `queued`/`processing` → `409 ANALYSIS_IN_PROGRESS`. 3. Another user's or an unknown ID → `404 NOT_FOUND`. 4. An `audit_events` row `analysis_deleted` is written. 5. Quota is not returned (FR-008 AC 6). |
| FR-016 | Erase all my data / withdraw consent | DPDP right to erasure and to withdraw consent. | Must | 1. API-010 in one transaction: deletes all the caller's analyses (any status, cascade), releases reservations of in-progress ones, sets `withdrawn_at = now()` on the active consent, writes `audit_events` row `data_erased` with the number of analyses deleted; returns `200` with `{ deletedAnalyses: n }`. 2. Afterwards API-006 returns an empty list and API-003 returns `hasValidConsent: false`. 3. Usage counters are kept (they hold no resume content) so erasing does not reset the monthly quota. 4. In-flight worker jobs for deleted analyses finish without writing (FR-005 AC 7). 5. The web client (W-01 "Delete all my data") asks for confirmation by typing `DELETE` before calling the API. |
| FR-017 | Automatic data retention | Personal data is removed automatically when no longer needed. | Must | 1. A scheduled job runs daily at 02:00 IST (20:30 UTC) in the worker (BullMQ repeatable job `retention`, one run at a time). 2. Sets `resume_text_enc = NULL`, `text_purged_at = now()` for analyses with `created_at < now() - K-31` (30 days) and `resume_text_enc IS NOT NULL`. 3. Hard-deletes analyses with `created_at < now() - K-32` (180 days). 4. Deletes `usage_counters` older than `K-35` (13 months), consents withdrawn more than `K-33` (3 years) ago, audit events older than `K-36` (3 years). 5. Works in batches of 1,000 rows, each in its own transaction. 6. Logs one `info` line with counts per category. 7. After text purge the analysis remains fully viewable (scores, missing skills, rewrites), since none of these need the full text. |
| FR-018 | Download PDF report | The user can download a completed analysis as a PDF. | Must | 1. API-008 returns `application/pdf` with `Content-Disposition: attachment; filename="resume-analysis-<YYYY-MM-DD>.pdf"` (date of `created_at` in IST). The user's name and the original file name are **not** in the file name. 2. Not `completed` → `409 ANALYSIS_NOT_READY`. 3. Content, in order: title "Resume Analysis Report"; analysis date (IST), target role (English name), experience level; overall score and band; six dimensions each with score, a horizontal bar, and explanation; missing skills grouped core / preferred; rewrite suggestions visible to the caller (FR-012) as "Before / After / Why"; for free users, one line "N more suggestions are available with SurfingBear Premium." when N > 0; footer on each page "Generated by SurfingBear Resume Optimizer · Page x of y". 4. The PDF is generated on request and never stored. 5. Font: Noto Sans (bundled in the image) so all Latin characters render. |
| FR-019 | Manage job-role catalog | Admins maintain the list of roles and the terms used for matching. | Must | 1. API-011 lists all roles (active and inactive) with term counts. 2. API-012 creates a role: `slug` matches `^[a-z0-9]+(-[a-z0-9]+)*$`, 3–60 chars, unique (`409 JOB_ROLE_SLUG_TAKEN`); `nameEn` 2–80 chars required; `nameHi` 2–80 chars required; new roles start **inactive**. 3. API-013 updates `nameEn`, `nameHi`, `isActive`; `slug` cannot change. 4. API-014 replaces the full term list of a role in one transaction. Rules: 1–60 chars per term; up to 10 aliases each, 1–60 chars; at most 80 terms; no two terms or aliases of the same role normalize (3.5.1) to the same string. 5. A role can be activated only if it has ≥ 5 `core`, ≥ 3 `keyword` terms; otherwise API-013 with `isActive: true` → `422 VALIDATION_FAILED` (`field: "isActive"`, `issue: "insufficient_terms"`). Replacing terms of an active role with a list that breaks this rule → the same error (`field: "terms"`). 6. Roles are never deleted (analyses reference them); they are deactivated. 7. Every admin change writes an `audit_events` row (`job_role_created`, `job_role_updated`, `job_role_terms_replaced`) with the admin's `userId`. 8. Changes affect only analyses created afterwards. |
| FR-020 | Show remaining quota | The user sees how many analyses are left this month. | Should | 1. API-002 returns `{ plan, period, limit, used, reserved, remaining, resetsAt }` with `remaining = max(0, limit − used − reserved)`. 2. W-01 shows "X of Y analyses left this month" and, for free users with `remaining = 0`, disables the submit button and shows the upgrade link (`K-09`). |
| FR-021 | Progress while processing | The user sees that the analysis is running and is taken to the result when done. | Must | 1. After `202`, W-01 navigates to W-02 for the new `analysisId`. 2. W-02 polls API-007 every `K-60` (2 s) while status is `queued` or `processing`, and shows a progress state ("Reading your resume…" for `queued`, "Scoring with AI…" for `processing`). 3. Polling stops on `completed`/`failed`, when the tab is hidden (resumes when visible), or after `K-44` (5 min) + 30 s, after which W-02 shows "This is taking longer than expected. Refresh to check again." |
| FR-022 | No analysis stays stuck | An analysis must always reach a final state. | Must | 1. A sweeper (BullMQ repeatable job `sweep-stuck`, every 60 s) sets `status = failed`, `failure_code = PROCESSING_TIMEOUT` on analyses in `queued`/`processing` with `created_at < now() − K-44` (5 min), releasing their reservations in the same transaction. 2. Together with FR-005 AC 7, a late worker cannot overwrite the `failed` state. |

### 3.2 Non-functional requirements

| ID | Category | Requirement | Measurable target | How it is verified |
|---|---|---|---|---|
| NFR-001 | Performance | Time for API-005 to answer `202` (includes upload, validation, extraction, DB, enqueue) | p95 ≤ 3 s, p99 ≤ 5 s for files ≤ `K-04` on a 20 Mbit/s client link; measured server-side as `durationMs` | k6 load test `load/upload.js` on staging (30 virtual users, 10 min, fixture `valid-backend-mid.pdf` 1.2 MB and `large-4900kb.pdf`); prod dashboard |
| NFR-002 | Performance | Time from `created_at` to `completed_at` | p95 ≤ 45 s, p99 ≤ 90 s at the load of NFR-006 | Metric `ro_analysis_duration_seconds` histogram; k6 staging test |
| NFR-003 | Performance | Read endpoints API-001, 002, 003, 006, 007, 011 | p95 ≤ 300 ms, p99 ≤ 800 ms server-side | k6; prod dashboard |
| NFR-004 | Performance | PDF report API-008 | p95 ≤ 3 s, p99 ≤ 6 s | k6 |
| NFR-005 | Availability | API and web availability per calendar month, measured as successful synthetic checks (every 60 s from 2 regions: GET API-015 and GET API-001 with a synthetic token) | ≥ 99.5 % (≈ 3 h 39 min downtime/month); AI provider outages that only fail jobs (FR-013) do not count as downtime, but `503` on API-005 for > 5 min does | CloudWatch Synthetics canary; monthly SLO report |
| NFR-006 | Scalability | Sustained load without breaking NFR-001–004 | 300 analyses/hour sustained, 30 analyses/minute bursts for 5 min, 500 concurrent users browsing | k6 staging; ECS autoscaling (4.5) |
| NFR-007 | Security | Web and API pass a dynamic scan | 0 high / 0 medium findings in OWASP ZAP baseline + API scan (OpenAPI from section 6) before each `vX.Y.0` release | ZAP in the release pipeline against staging |
| NFR-008 | Privacy | Original resume files are never persisted | 0 bytes of uploaded files in DB, Redis, disk or logs. Multer uses `memoryStorage()`; the container root filesystem is read-only except `/tmp` (tmpfs, 64 MiB) | Integration test: upload, then scan DB, Redis dump and `/tmp` for a unique marker string embedded in the fixture; ECS task definition `readonlyRootFilesystem: true` |
| NFR-009 | Privacy | Retention deadlines (FR-017) are met | Resume text purged ≤ 24 h after day `K-31`; analyses deleted ≤ 24 h after day `K-32` | Integration test with clock set forward; daily job success metric + alert |
| NFR-010 | Privacy | Erasure (FR-015, FR-016) is immediate and also leaves backups in bounded time | API returns after the DB commit (p95 ≤ 1 s); deleted data is gone from all RDS automated backups after `K-34` (7 days) | e2e test; RDS `backup_retention_period = 7` in Terraform |
| NFR-011 | Privacy / residency | Where data is stored and processed | All stored data (RDS, ElastiCache, logs, backups) in AWS `ap-south-1` (Mumbai). Only redacted resume text leaves India, to the Anthropic API (USA), as stated in the consent notice | Terraform review; FR-003 AC 7–8 tests |
| NFR-012 | Observability | Metrics, traces and alerts exist for every main flow | Metrics in 4.8 are emitted; OpenTelemetry traces cover HTTP request → DB → queue → worker → AI call with one trace per analysis; alerts in 4.8 page the on-call within 5 min of the condition | Staging check in release checklist; alert fire-drill quarterly |
| NFR-013 | Reliability | Every analysis reaches a final state | 100 % of analyses are `completed` or `failed` within `K-44` + 60 s of creation | Daily query `SELECT count(*) FROM analyses WHERE status IN ('queued','processing') AND created_at < now() - interval '6 minutes'` must be 0; alert if not |
| NFR-014 | Reliability | Successful analyses when the AI provider is healthy | ≥ 98 % of analyses that pass validation end `completed` (7-day rolling) | Metric `ro_analysis_finished_total{status}` |
| NFR-015 | Cost | AI cost per completed analysis | Mean ≤ US$0.20, p95 ≤ US$0.40 (Claude Opus 5.5 at $4 / $20 per million input / output tokens) | Computed from stored token counts (TBL-004) in a weekly query; alert if 7-day mean > $0.20 |
| NFR-016 | Accessibility | Web pages W-01–W-03 | WCAG 2.1 AA; 0 `serious`/`critical` violations from axe-core; full keyboard use; score bars have text equivalents | Playwright + `@axe-core/playwright` in CI |
| NFR-017 | Consistency | Deterministic dimensions are repeatable | Same redacted text + role terms + level → identical scores for the 4 deterministic dimensions (100 % of runs) | Unit test runs scorers 100× on fixtures |
| NFR-018 | Maintainability | Test coverage of backend code | ≥ 80 % lines and ≥ 70 % branches in `apps/api/src/**` (CR-031) | Jest coverage gate in CI |
| NFR-019 | Compatibility | Supported browsers | Last 2 major versions of Chrome, Edge, Firefox, Safari (desktop) and Chrome Android, Safari iOS; layouts work from 360 px width | Playwright projects for Chromium, Firefox, WebKit; manual check on one Android and one iPhone per release |
| NFR-020 | Scalability | Data growth | Design for 2 million analyses rows and 20 million child rows without changing the schema; read queries use the indexes in 5.3 | `EXPLAIN` checks in the migration PR with 2 M synthetic rows on staging |

### 3.3 Assumptions and out of scope

**Assumptions** (decisions not given in the product description, with reasons):

| ID | Assumption | Reason |
|---|---|---|
| A-01 | The platform issues an RS256 JWT access token in the httpOnly cookie `sb_access` with claims `sub`, `plan`, `roles`, `iss`, `aud`, `exp`, and publishes a JWKS endpoint. | The module must not handle login (brief). Reading a signed token is the least coupling: no call to the platform per request. If the platform differs, only `JwtAuthGuard` changes. |
| A-02 | The platform owns user accounts. This module stores only the platform `userId` (UUID), never name/email/phone. | Data minimization (DPDP). Nothing in this module needs to contact the user. |
| A-03 | Platform access tokens live 15 minutes, so plan changes appear within 15 minutes. | Platform setting; acceptable delay for plan changes. |
| A-04 | Plans are only `free` and `premium`. Payment and upgrade happen on the platform at `K-09 UPGRADE_URL`. | Brief mentions free users only; billing is not part of this module. |
| A-05 | Only English-language resumes are supported. | The scoring terms and AI prompt are English; Indian tech resumes are overwhelmingly English. Others are rejected with a clear error instead of producing a bad score. |
| A-06 | Scanned (image-only) resumes are not supported; no OCR. | OCR adds cost, latency and error; most ATS systems also fail on scans, so "not enough text" is honest feedback. |
| A-07 | Free = 3 analyses/month and 3 of up to 10 suggestions visible; premium = 50/month (fair-use cap). | 3 lets a free user try, fix, and re-check once or twice. A premium cap protects AI cost (NFR-015) and against scripted abuse. |
| A-08 | The original file is not stored at all. | It is only needed once, for extraction. Not storing it removes the biggest privacy risk and removes object storage from the design. Consequence: "re-analyse the same file" needs a re-upload (accepted). |
| A-09 | AI provider is Anthropic (Claude) via its public API. Anthropic processes data in the USA; SurfingBear has a commercial agreement under which API inputs are not used for training. | See 4.5 for the choice. DPDP permits transfer outside India except to countries the government restricts; the USA is not restricted as of 2026-10. Disclosed in the consent notice. |
| A-10 | Months for quota are IST calendar months. | Users are in India; "monthly" should reset at local midnight on the 1st. |
| A-11 | The job-role catalog is seeded with the 6 roles in 5.5 at launch; admins extend it. | A useful launch set for SurfingBear's audience (tech roles); the catalog is data, not code. |
| A-12 | The grievance contact is `grievance@surfingbear.com`, run by the platform's Data Protection team. | DPDP requires a grievance channel; the platform already has one. |

**Out of scope** (deliberately not covered in this version):

- Sign-up, login, logout, session refresh, account deletion (platform).
- Payments, upgrades, plan management (platform).
- OCR, image resumes, `.doc`, `.rtf`, `.odt`, `.txt`, and non-English resumes.
- Editing the resume inside the product, or generating a new resume file.
- Matching against a specific job posting / pasted job description (only catalog roles).
- Translating the PDF report or AI suggestions into Hindi.
- A mobile app client (the web app is responsive, NFR-019).
- Admin/support access to users' analyses; analytics dashboards beyond the metrics in 4.8.
- Sharing an analysis via public link.
- Bulk/API access for third parties.

### 3.4 Configuration constants

Single source for every number used in more than one place. Values marked *env* are read from environment variables (section 4.6) and validated at startup; all others are code constants in `apps/api/src/config/constants.ts` (and mirrored in `packages/contracts` where the web needs them).

| ID | Name | Value | Used by |
|---|---|---|---|
| K-01 | `PLATFORM_JWKS_URL` | *env*, e.g. `https://auth.surfingbear.com/.well-known/jwks.json` | CR-002, CR-003 |
| K-02 | `PLATFORM_ISSUER` | *env*, e.g. `https://auth.surfingbear.com` | CR-002 |
| K-03 | `WEB_ORIGIN` | *env*, per environment (1.7) | CR-006, CR-022 |
| K-04 | `MAX_FILE_BYTES` | 5,242,880 (5 MiB) | FR-002, API-005, ERR-007 |
| K-05 | `MAX_PDF_PAGES` | 10 | FR-002, ERR-008 |
| K-06 | `MIN_TEXT_CHARS` | 300 non-whitespace characters | FR-003, ERR-011 |
| K-07 | `MAX_TEXT_CHARS` | 40,000 characters after normalization | FR-003, ERR-025 |
| K-08 | `CONSENT_NOTICE_VERSION` | `2026-10-01` | FR-004, API-003, API-004 |
| K-09 | `UPGRADE_URL` | *env*, e.g. `https://www.surfingbear.com/premium` | FR-012, FR-020 |
| K-10 | `FREE_MONTHLY_LIMIT` | 3 analyses | FR-008, API-002 |
| K-11 | `PREMIUM_MONTHLY_LIMIT` | 50 analyses | FR-008, API-002 |
| K-12 | `MAX_IN_PROGRESS_PER_USER` | 1 | FR-005, ERR-013 |
| K-13 | `MAX_REWRITES` | 10 | FR-011, AI schema (4.7) |
| K-14 | `FREE_VISIBLE_REWRITES` | 3 | FR-012, FR-018 |
| K-20 | `RATE_LIMIT_PER_MINUTE` | 60 requests per user per minute | CR-019 |
| K-21 | `UPLOAD_LIMIT_PER_HOUR` | 10 calls to API-005 per user per hour | CR-019 |
| K-30 | `ORIGINAL_FILE_RETENTION` | 0 — never stored | CR-020, NFR-008 |
| K-31 | `TEXT_RETENTION_DAYS` | 30 days after `created_at` | FR-017 |
| K-32 | `ANALYSIS_RETENTION_DAYS` | 180 days after `created_at` | FR-017 |
| K-33 | `CONSENT_RECORD_RETENTION` | 3 years after `withdrawn_at` | FR-017 |
| K-34 | `DB_BACKUP_RETENTION_DAYS` | 7 days | NFR-010 |
| K-35 | `USAGE_COUNTER_RETENTION_MONTHS` | 13 months after the period | FR-017 |
| K-36 | `AUDIT_RETENTION` | 3 years after `created_at` | FR-017 |
| K-40 | `AI_TIMEOUT_MS` | 60,000 ms per attempt | FR-013 |
| K-41 | `AI_MAX_ATTEMPTS` | 3 | FR-013 |
| K-42 | `AI_RETRY_DELAYS_MS` | `[5000, 15000]` | FR-013 |
| K-43 | `AI_BREAKER` | open after 10 failed attempts in 120 s; stay open 60 s | FR-013 |
| K-44 | `PROCESSING_TIMEOUT_MS` | 300,000 ms (5 min) from `created_at` | FR-021, FR-022 |
| K-45 | `WORKER_CONCURRENCY` | 5 jobs per worker task | 4.5 |
| K-46 | `AI_MODEL` | *env*, default `claude-opus-5-5` | FR-007, 4.7 |
| K-47 | `AI_MAX_TOKENS` | 16,000 | 4.7 |
| K-48 | `AI_EFFORT` | `medium` | 4.7 |
| K-49 | `PROMPT_VERSION` | `rv1` | FR-007, TBL-004 |
| K-50 | AI monthly spend cap | dev $50, staging $200, prod $3,000 (set in the Anthropic console per API key) | 1.7, NFR-015 |
| K-60 | `POLL_INTERVAL_MS` | 2,000 | FR-021 |
| K-61 | `PAGE_SIZE_DEFAULT` / `PAGE_SIZE_MAX` | 20 / 50 | 6.1 |

### 3.5 Scoring rules

Weights (sum 100):

| Dimension (`dimension` value) | Weight | Computed by |
|---|---|---|
| `keyword_match` | 20 | Code (3.5.1) |
| `skills_coverage` | 25 | Code (3.5.1) |
| `experience_relevance` | 20 | AI (4.7) |
| `impact_quantification` | 15 | AI (4.7) |
| `structure_completeness` | 10 | Code (3.5.2) |
| `formatting_parseability` | 10 | Code (3.5.3) |

#### 3.5.1 Term matching, `keyword_match`, `skills_coverage`

`normalize(s)`: Unicode NFKC → lowercase → replace every character that is not `a–z`, `0–9`, `+`, `#`, `.` with a space → remove `.` at the start or end of each space-separated token → collapse spaces → trim.

A term **matches** the resume when, for the term itself or any of its aliases, `" " + normalize(term) + " "` is a substring of `" " + normalize(resumeText) + " "`. (So `java` does not match `javascript`, `c++` matches `C++,`, and `node.js` matches `Node.js.`.)

- `keyword_match = round_half_up(100 × matchedKeywordTerms / totalKeywordTerms)` over terms of type `keyword`.
- `skills_coverage = round_half_up(100 × (2 × matchedCore + 1 × matchedPreferred) / (2 × totalCore + 1 × totalPreferred))`.

An active role always has ≥ 5 core and ≥ 3 keyword terms (FR-019), so denominators are never 0.

#### 3.5.2 `structure_completeness`

Five sections, 20 points each:

| Section | Found when |
|---|---|
| Contact | `has_email` or `has_phone` is true (FR-003 AC 6) |
| Summary | A heading line matches one of: `summary`, `profile`, `professional summary`, `career summary`, `objective`, `career objective`, `about me` |
| Experience | A heading line matches one of: `experience`, `work experience`, `professional experience`, `employment history`, `work history`, `internship`, `internships`, `internship experience`. For level `fresher` also: `projects`, `academic projects`, `personal projects` |
| Education | A heading line matches one of: `education`, `academic background`, `academics`, `educational qualifications`, `qualifications`, `education and training` |
| Skills | A heading line matches one of: `skills`, `technical skills`, `key skills`, `core competencies`, `skills and tools`, `technologies`, `tech stack` |

A **heading line** is a line of the normalized text whose trimmed length is ≤ 40 characters and whose `normalize()` value, after removing a trailing `:`, equals one of the listed strings (`normalize()` also turns `&` into a space, so `Skills & Tools` → `skills tools`; the list therefore contains both `skills and tools` and `skills tools` — implementers add the `&`-variant of every entry containing `and`).

#### 3.5.3 `formatting_parseability`

Start at 100, subtract each that applies, floor at 0:

| Check | Deduction |
|---|---|
| Word count (tokens split on whitespace) < 250 | −20 |
| Word count > 1,200 for `fresher`/`junior`, > 1,800 for others | −10 |
| Pages over the guide: PDF page count, or for DOCX `ceil(wordCount / 500)`, greater than 1 (`fresher`), 2 (`junior`, `mid`), 3 (`senior`, `lead`) | −15 |
| `has_email` is false | −15 |
| `has_phone` is false | −10 |
| Fewer than 3 bullet lines (line starts with `•`, `●`, `▪`, `◦`, `-`, `*`, `–`, or `>` followed by a space) | −10 |
| Mean line length of non-empty lines > 160 characters (text blocks or merged columns) | −10 |
| More than 5 % of non-whitespace characters are outside printable ASCII and are not letters (`\p{L}`) — icons, broken glyphs | −15 |

#### 3.5.4 Contact detection regexes (before redaction)

- Email: `/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g`
- Phone: `/(?<![\w])\+?\d[\d\s\-().]{8,18}\d(?![\w])/g`, accepted only if it contains 10–13 digits in total.
- URL: `/\b(?:https?:\/\/|www\.)\S+|\b(?:linkedin\.com|github\.com)\/\S*/gi`

Redaction replaces matches in this order: URL, email, phone.

#### 3.5.5 Explanation templates for code-scored dimensions

| Dimension | Template (English) |
|---|---|
| `keyword_match` | `Found {matched} of {total} common keywords for {roleName}.` |
| `skills_coverage` | `Found {matchedCore} of {totalCore} core and {matchedPreferred} of {totalPreferred} preferred skills for {roleName}.` |
| `structure_completeness` | `Found {n} of 5 standard sections; missing: {list}.` or, when none missing, `All 5 standard sections found.` |
| `formatting_parseability` | `{n} formatting issue(s) may confuse ATS parsers: {list}.` or `No formatting issues found.` where list items are: `too short`, `too long`, `too many pages`, `no email`, `no phone`, `few bullet points`, `very long lines`, `unusual symbols` |

### 3.6 Web screens

| ID | Route | Contents and behaviour | FRs |
|---|---|---|---|
| W-01 | `/resume-optimizer` | Header with `QuotaBadge` (API-002). If API-003 says no valid consent: `ConsentDialog` (modal, notice text from `messages/*.json` key `consent.body`, checkbox "I agree", button "Continue" → API-004). `UploadForm`: role dropdown (API-001), level radio group (5 options with year ranges), file input (`accept=".pdf,.docx"`; client-side pre-check of size `K-04` shows the same message as `FILE_TOO_LARGE` without uploading), submit. On API errors, the localized `error.message` shows under the form. History list (API-006) with status chip, role, score, date, and actions "View", "Download PDF" (completed only), "Delete" (confirm dialog). Link "Delete all my data" (FR-016). | FR-001, FR-004, FR-005, FR-014, FR-015, FR-016, FR-020 |
| W-02 | `/resume-optimizer/analyses/[analysisId]` | Progress state while polling (FR-021). When completed: `ScoreCard` (overall score, band, role, level, date), six `DimensionBar`s with explanation, `MissingSkills` (two groups), `RewriteList` (Before/After/Why; locked cards for free users), "Download PDF" button, "Delete" button. When failed: message for `failureCode` and "Try again". Unknown/other user's ID: "Analysis not found" page. | FR-009 – FR-013, FR-018, FR-021 |
| W-03 | `/resume-optimizer/admin/job-roles` | Visible only if the token has `ro_admin` (the page calls API-011; on 403 shows "Not allowed"). Table of roles with active toggle, edit names, and a terms editor (rows of term / type / aliases, comma-separated) that saves through API-014; validation errors are shown per row using `details[].field` paths such as `terms[3].aliases[0]`. | FR-019 |

### 3.7 Test fixtures

Synthetic files in `apps/api/test/fixtures/resumes/` (no real people). Each is created by `npm run fixtures:build` from source text in `fixtures/src/` so they can be regenerated.

| File | Content | Expected result |
|---|---|---|
| `valid-backend-mid.pdf` | 2-page English resume for a backend developer, all 5 sections, email + phone, 14 bullets, ~650 words, contains the unique marker `ZXQ-MARKER-7731` | `202`; with role `backend-developer`, level `mid`: `structure_completeness = 100`, `formatting_parseability = 100` |
| `valid-frontend-junior.docx` | 1-page DOCX, no summary section, no phone | `202`; `structure_completeness = 80`, `formatting_parseability = 90` |
| `large-4900kb.pdf` | valid resume padded with an embedded image to 4,900 KB | `202` |
| `six-mb.pdf` | 6 MiB | `413 FILE_TOO_LARGE` |
| `png-renamed.pdf` | PNG image with `.pdf` extension | `415 UNSUPPORTED_FILE_TYPE` |
| `legacy.doc` | Word 97 file | `415 UNSUPPORTED_FILE_TYPE` |
| `encrypted.pdf` | Password `test` | `422 FILE_ENCRYPTED` |
| `truncated.pdf` | First 50 % of a valid PDF | `422 FILE_UNREADABLE` |
| `zip-bomb.docx` | DOCX whose `word/document.xml` expands to 100 MiB | `422 FILE_UNREADABLE` |
| `eleven-pages.pdf` | 11 pages of text | `422 TOO_MANY_PAGES` |
| `scanned.pdf` | 1 page, image only | `422 NOT_ENOUGH_TEXT` |
| `hindi.docx` | Resume written in Devanagari | `422 RESUME_LANGUAGE_UNSUPPORTED` |
| `too-long.docx` | 45,000 characters of text | `422 RESUME_TOO_LONG` |

Recorded AI responses in `apps/api/test/fixtures/ai/` (`ok.json`, `invented-number.json` — one suggestion adds "40%" → dropped, `bad-schema.json`, `refusal.json`) are used by `AI_MODE=fake` and by unit tests.

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
| v0.2 | 2026-10-20 | Kakali Dinda | Section 3 written: FRs, NFRs, assumptions, configuration constants, scoring rules, web screens, test fixtures | First draft | FR-001 … FR-022, NFR-001 … NFR-020, A-01 … A-12, K-01 … K-61 |
