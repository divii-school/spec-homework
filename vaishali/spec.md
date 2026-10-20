# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Vaishali Naruka |
| Version | v0.3 |
| Last updated | 2026-10-20 |
| Status | Draft |

**What this document is.** The single source for building the SurfingBear Resume Optimizer ("RO") module: a Next.js web client, a NestJS API, a NestJS background worker, PostgreSQL and Redis. A developer must be able to build, test and deploy RO from this file alone. If this file is silent on something, that is a defect in this file: raise it, do not guess.

**Normative words.** **MUST** / **MUST NOT** = mandatory; **SHOULD** = expected unless a reason is written in the PR; **MAY** = free choice of the implementer.

**ID rules**

| Prefix | Used for | Example |
|---|---|---|
| `CR-` | Company-wide common requirement | CR-001 |
| `FR-` | Functional requirement | FR-001 |
| `NFR-` | Non-functional requirement | NFR-001 |
| `CFG-` | Configuration constant (single source for every number) | CFG-01 |
| `CMP-` | Architecture component | CMP-001 |
| `TBL-` | Database table | TBL-001 |
| `API-` | API endpoint | API-001 |
| `ERR-` | Error code | ERR-001 |
| `SCR-` | Web screen | SCR-01 |

IDs are permanent. An ID is never renumbered or reused; a removed item is marked **Deprecated** and its row stays.

**Rule for numbers.** Every limit, timeout and threshold is defined once in §3.4 as a `CFG-` constant. Other sections quote the constant ID together with its value, e.g. "max `CFG-01` (5 MiB)". If the two ever disagree, §3.4 wins.

---

## 1. Company-wide common requirements

### 1.1 Authentication

RO has no sign-up or login of its own. The main SurfingBear platform ("the platform") logs the user in and issues the access token. RO only **verifies** it.

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-001 | Authentication | The platform sets the access token as a cookie named `sb_access` on domain `.surfingbear.com` with `HttpOnly; Secure; SameSite=Lax; Path=/`. RO reads the token **only** from this cookie. RO never reads tokens from `localStorage`, query strings or the `Authorization` header in deployed environments (`dev`, `staging`, `prod`). In `local` and `test`, `Authorization: Bearer <jwt>` is also accepted so e2e tests can call the API directly. | e2e test: request with valid token in `Authorization` header to `staging` returns 401; same token in cookie returns 200 |
| CR-002 | Authentication | The token is a JWT signed with **RS256**. RO verifies it with the platform's public keys from `AUTH_JWKS_URL` (§4.6). Keys are cached in memory for `CFG-21` (10 min); on an unknown `kid` the cache is refreshed at most once per `CFG-22` (60 s). | Unit test of `JwtVerifier` with rotated key fixture |
| CR-003 | Authentication | A token is valid only if: signature is valid; `alg` is `RS256` (any other value, including `none`, is rejected); `iss` equals `AUTH_ISSUER`; `aud` contains `AUTH_AUDIENCE`; `exp` is in the future and `nbf`/`iat` are not in the future, with a clock skew tolerance of `CFG-23` (30 s); `sub` is a UUID. Any failure → `401 UNAUTHENTICATED` (ERR-001). | Unit tests, one per rule |
| CR-004 | Authentication | Claims RO uses: `sub` (platform user ID, UUID) → `userId`; `plan` (`"free"` or `"pro"`) → plan; `roles` (array of strings) → roles. A missing or unknown `plan` value is treated as `"free"`. A missing `roles` claim is treated as `[]`. RO never stores the token. | Unit test with claim fixtures |
| CR-005 | Authentication | On `401` the web client redirects the browser to `${PLATFORM_LOGIN_URL}?returnTo=<url-encoded current RO URL>`. Token refresh is the platform's job; RO does not implement refresh. | Playwright test with expired token fixture |
| CR-006 | Authentication | RO keeps no user table. The platform owns the user profile (name, email). RO stores only the `userId` UUID as the owner key of its own rows. | Schema review: no `users` table, no name/email columns |

### 1.2 Roles and permissions

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-007 | Roles | RO has three effective roles, derived from token claims on every request: **Free user** (`plan` = `free`, no `ro_admin` in `roles`), **Pro user** (`plan` = `pro`, no `ro_admin`), **Admin** (`roles` contains `"ro_admin"`; plan is irrelevant for admin endpoints). An admin also uses the user endpoints with their own plan. | Unit test of `RolesGuard` |
| CR-008 | Ownership | A user can read, change or delete **only their own** resumes, analyses and consents. A request for another user's resource returns `404 NOT_FOUND` (ERR-004), never `403`, so the existence of other users' data is not revealed. This applies to admins too: admins have **no** access to any user's resume text or analysis. | e2e test: user B gets 404 on user A's analysis ID |
| CR-009 | Roles | Plan is read from the token on every request, never cached across requests. An upgrade from `free` to `pro` takes effect on the first request with the new token. | e2e test |

Permission table (✔ = allowed, ✗ = denied with the error shown).

| Action | API | Free user | Pro user | Admin | No/invalid token |
|---|---|---|---|---|---|
| Read own context (plan, quota, consent) | API-001 | ✔ | ✔ | ✔ | ✗ 401 |
| Read privacy notice | API-002 | ✔ | ✔ | ✔ | ✗ 401 |
| Give consent | API-003 | ✔ | ✔ | ✔ | ✗ 401 |
| Upload / list / view / delete own resume | API-004 – API-007 | ✔ | ✔ | ✔ | ✗ 401 |
| List active job roles | API-008 | ✔ | ✔ | ✔ | ✗ 401 |
| Start analysis | API-009 | ✔ up to `CFG-05` (3) / month | ✔ up to `CFG-06` (50) / month | ✔ by own plan | ✗ 401 |
| List / view / delete own analyses | API-010 – API-012 | ✔ | ✔ | ✔ | ✗ 401 |
| See rewrite suggestions | API-011, API-013 | first `CFG-07` (3) only | all | by own plan | ✗ 401 |
| Download PDF report | API-013 | ✔ (gated content) | ✔ (full) | ✔ by own plan | ✗ 401 |
| Data summary / erase all my data | API-014, API-015 | ✔ | ✔ | ✔ | ✗ 401 |
| Manage job-role catalogue | API-016 – API-019 | ✗ 403 | ✗ 403 | ✔ | ✗ 401 |
| Health checks | API-020, API-021 | public | public | public | public |

### 1.3 Logging

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-010 | Logging | All services log JSON, one object per line, to stdout, via `pino` (`nestjs-pino` in API and worker; `pino` in Next.js server code). Fields on every line: `time` (ISO-8601 UTC with ms), `level` (`debug`/`info`/`warn`/`error`/`fatal`), `service` (`ro-web`, `ro-api`, `ro-worker`), `env`, `version` (git short SHA), `requestId`, `msg`. Added when known: `userId`, `analysisId`, `resumeId`, `jobId`, `route`, `method`, `status`, `durationMs`, `errorCode`. | Unit test of logger config; log sample review on `staging` |
| CR-011 | Logging | **Personal data is never logged.** Never log: resume text (raw or redacted), file names, file bytes, AI prompts, AI responses, rewrite text, email, phone, the JWT or the `Cookie` header. `pino` `redact` paths MUST include `req.headers.cookie`, `req.headers.authorization`, `res.headers["set-cookie"]`, `*.text`, `*.fileName`, `*.prompt`, `*.completion`. `userId` (an opaque UUID) MAY be logged. | Unit test asserting redact paths; CI step `scripts/check-log-pii.ts` greps test-run logs for the fixture resume's email, phone and name and fails if found |
| CR-012 | Logging | Every HTTP request gets a `requestId`: the incoming `X-Request-Id` header if it is a valid UUID, otherwise a new UUID v4. The value is echoed in the `X-Request-Id` response header and in every error body (§1.4). The API passes it to the worker in the job payload, so worker logs for that analysis carry the same `requestId`. | e2e test |
| CR-013 | Logging | Log levels: `prod` = `info`, `staging` = `info`, `dev` = `debug`, `local` = `debug`, `test` = `warn`. Each request is logged once at completion (`info` for 2xx–4xx, `error` for 5xx). | Config review |

### 1.4 Error handling

Every API error uses this **one** response body, with `Content-Type: application/json`:

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "The file is larger than 5 MB.",
    "requestId": "8f14e45f-ea3b-4c1e-9b1a-2f6d2c1e7a10",
    "details": [
      { "field": "file", "issue": "max_size", "limit": 5242880 }
    ]
  }
}
```

| Field | Type | Required | Rule |
|---|---|---|---|
| `error.code` | string | yes | One of the codes in §6.3. UPPER_SNAKE_CASE. Stable: clients branch on it. |
| `error.message` | string | yes | English, one sentence, safe to show to the user. Exact text per code in §6.3. Never contains stack traces, SQL, file content or other users' data. |
| `error.requestId` | string (UUID) | yes | Same as the `X-Request-Id` response header. |
| `error.details` | array | no | Present only for `VALIDATION_FAILED` and the file errors. Each item: `field` (string, request field name), `issue` (string, one of `required`, `type`, `min`, `max`, `min_length`, `max_length`, `pattern`, `enum`, `max_size`, `unknown_field`, `body_too_large`), optional `limit` (number). |

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-014 | Errors | A single global NestJS exception filter (`AppExceptionFilter`) produces every error body. Controllers and services throw `AppError(code, details?)`; the filter maps `code` → HTTP status and message from the table in §6.3. | Unit test: every code in §6.3 maps to the listed status |
| CR-015 | Errors | Framework errors are mapped: class-validator failure → 400 `VALIDATION_FAILED`; unknown route → 404 `NOT_FOUND`; JSON body over `CFG-24` (100 KB) or malformed JSON → 400 `VALIDATION_FAILED` with `issue: "body_too_large"` or `issue: "type"`; Multer file over `CFG-01` → 413 `FILE_TOO_LARGE`; throttler → 429 `RATE_LIMITED`; anything else → 500 `INTERNAL_ERROR` with the generic message, and the stack is logged at `error`. | e2e tests |
| CR-016 | Errors | `429` responses carry `Retry-After` (seconds, integer). `503` responses carry `Retry-After: 30`. | e2e test |
| CR-017 | Errors | Background failures are not HTTP errors. A failed analysis is stored with `status = "failed"` and a `failureCode` from §6.3 (ERR-022 – ERR-024), which the client reads from API-011. | e2e test with AI mock in failure mode |

### 1.5 Security

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-018 | Secrets | Secrets (DB password, Redis password, `RESUME_TEXT_KEY`) live in AWS Secrets Manager and reach containers as environment variables injected by ECS. No secret is committed to git; `.env*` files except `.env.example` are git-ignored. CI runs `gitleaks` on every PR and fails on a finding. | CI job `secrets-scan` |
| CR-019 | Transport | HTTPS only, TLS 1.2+ at the AWS ALB; HTTP → HTTPS 301. Response header `Strict-Transport-Security: max-age=31536000; includeSubDomains`. | `curl -I` check in deploy smoke test |
| CR-020 | CORS | Web and API are served from the **same origin** (`https://resume.surfingbear.com`, `/api/*` routed to the API, §4.1). The API therefore enables **no** CORS: it sends no `Access-Control-Allow-Origin` header, so cross-origin browser calls fail. | e2e test: preflight from other origin gets no ACAO header |
| CR-021 | CSRF | For every `POST`, `PUT`, `PATCH`, `DELETE`, the API requires the `Origin` header to equal `WEB_ORIGIN` (§4.6). Missing or different → `403 FORBIDDEN` (ERR-002). Combined with `SameSite=Lax` on the cookie (CR-001), this blocks cross-site form posts. | e2e test |
| CR-022 | Input validation | Every request body, query and path parameter is validated with `class-validator` DTOs and `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`. Unknown fields → 400 `VALIDATION_FAILED` with `issue: "unknown_field"`. Path IDs must be UUIDs, else 404 `NOT_FOUND`. | Unit tests per DTO |
| CR-023 | File validation | Uploaded files are checked by **content**, not by name or `Content-Type`: magic bytes `%PDF-` (PDF) or ZIP header `PK\x03\x04` **and** a `word/document.xml` entry (DOCX). Files are processed in memory only and never written to disk or object storage (FR-004). DOCX unzip is limited to `CFG-25` (20 MB) total uncompressed size and `CFG-26` (500) entries to stop zip bombs. | Unit tests with fixtures in §3.7 |
| CR-024 | Rate limiting | Per-user limits, counted in Redis (fixed window): all API routes `CFG-15` (120 requests / 60 s); `POST /resumes` `CFG-16` (10 / hour); `POST /analyses` `CFG-17` (20 / hour). Unauthenticated requests are limited per client IP at 60 / 60 s. Over the limit → 429 `RATE_LIMITED` (ERR-018). | e2e test |
| CR-025 | Headers | The web app sends `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-<per-request>'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'`, plus `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`. The API uses `helmet()` defaults. | Deploy smoke test checks headers |
| CR-026 | Data at rest | RDS and ElastiCache use AWS KMS encryption at rest. In addition, resume text is encrypted in the application with AES-256-GCM before it is written to PostgreSQL (§5.2, TBL-004). | Schema + code review |
| CR-027 | Data retention | Retention is fixed (DPDP purpose limitation): a resume (its redacted text and metadata) is deleted `CFG-09` (30 days) after upload; an analysis and its children are deleted `CFG-10` (180 days) after creation; audit events are kept `CFG-11` (365 days). The deletion is a hard delete (FR-021). The user can delete earlier at any time (FR-006, FR-016, FR-020). | Purge job test with clock fixture |
| CR-028 | DPDP | RO processes personal data only for the stated purpose (resume analysis), only after recorded consent (FR-002), stores data only in AWS `ap-south-1` (Mumbai), sends only **redacted** text to the AI provider (FR-005), and offers erasure (FR-020). The privacy notice names the Grievance Officer contact `grievance@surfingbear.com`. | Checklist review before release |

### 1.6 Internationalization (i18n)

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-029 | Language | v1 UI language is English (`en-IN`) only. All user-facing strings in the web app come from `apps/web/messages/en.json` through `next-intl`; no literal text in JSX. Adding a language later MUST need only a new messages file. | ESLint rule `i18next/no-literal-string` on `apps/web/src/**/*.tsx` |
| CR-030 | Time | The API stores and returns all timestamps in UTC, ISO-8601 with `Z` and milliseconds (`2026-10-20T09:15:00.000Z`). The web displays them in `Asia/Kolkata` as `20 Oct 2026, 2:45 pm`. The quota month is the calendar month in `Asia/Kolkata` (§3.5.6). | Unit test of date formatter and quota period function |
| CR-031 | Numbers | Scores are integers 0–100, displayed without decimals. Numbers use `en-IN` grouping (`1,00,000`). | Unit test |
| CR-032 | Error text | The API `message` is English. The web shows its own translated text keyed by `error.code` (`errors.<CODE>` in `en.json`), falling back to the API `message` for unknown codes. | Unit test of error mapper |
| CR-033 | Text input | Resume text is handled as UTF-8 and normalised to Unicode NFC (§3.5.1). Non-Latin scripts (e.g. Hindi) are accepted, but keyword matching only works for the English skill names in the catalogue (see §3.3 assumption A-07). | Unit test with Devanagari fixture |

### 1.7 Environments

| Env | Purpose | URL | Data | AI | Auth | Deploy trigger |
|---|---|---|---|---|---|---|
| `local` | Developer machine | `http://localhost:3000` (web), `:4000` (API) | Docker Compose Postgres + Redis, seeded | `AI_MODE=mock` | Mock platform (§4.8) issues tokens; `Authorization` header allowed | `pnpm dev` |
| `test` | CI unit + e2e | in CI container | Fresh Postgres + Redis per run | `AI_MODE=mock` | Mock platform; `Authorization` header allowed | every push |
| `dev` | Shared integration | `https://resume.dev.surfingbear.com` | Synthetic data only | `AI_MODE=bedrock`, `CFG-27` daily AI call cap 200 | Platform dev | merge to `main` |
| `staging` | Release candidate, QA | `https://resume.staging.surfingbear.com` | Synthetic data only; no real resumes | `AI_MODE=bedrock` | Platform staging | git tag `vX.Y.Z-rc.N` |
| `prod` | Real users | `https://resume.surfingbear.com` | Real personal data | `AI_MODE=bedrock` | Platform prod | git tag `vX.Y.Z` + manual approval |

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-034 | Environments | Real resumes exist only in `prod`. Copying prod data to any other environment is forbidden. | Access policy: no IAM role can read prod RDS snapshots from other accounts |
| CR-035 | Environments | `dev`, `staging` and `prod` run in separate AWS accounts with identical Terraform (`infra/`), differing only by the variables in `infra/envs/<env>.tfvars` (instance sizes, desired counts, domains). | Terraform plan review |

### 1.8 Deployment

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-036 | CI | GitHub Actions workflow `.github/workflows/ci.yml` runs on every PR and push: `pnpm install --frozen-lockfile` → `pnpm lint` → `pnpm typecheck` → `pnpm test` (unit) → `pnpm test:e2e` (API e2e against Postgres 16 + Redis 7 service containers, `AI_MODE=mock`) → `pnpm build` → `gitleaks`. A PR cannot merge unless all are green. | Branch protection on `main` |
| CR-037 | Build | One Docker image per app: `ro-web`, `ro-api` (the worker uses the `ro-api` image with command `node dist/worker.js`). Images are tagged with the git SHA and pushed to ECR. | Workflow review |
| CR-038 | Release | `deploy.yml`: merge to `main` → deploy to `dev`; tag `vX.Y.Z-rc.N` → `staging`; tag `vX.Y.Z` → `prod` after a manual approval in the GitHub `prod` environment by one of the 2 named release approvers. Deploy order: run DB migrations (one-off ECS task `node dist/migrate.js`) → update `ro-worker` → update `ro-api` → update `ro-web`. ECS rolling update with `minimumHealthyPercent=100`, `maximumPercent=200`. | Workflow review |
| CR-039 | Migrations | Migrations are forward-only and **backward compatible with the previous release** (expand → migrate → contract over two releases). A column is never renamed or dropped in the same release that stops using it. | Code review checklist |
| CR-040 | Rollback | Rollback = redeploy the previous image tag (`deploy.yml` with input `image_tag`). Because migrations are backward compatible (CR-039), no down-migration is needed. ECS deployment circuit breaker with auto-rollback is enabled: if new tasks fail `/health/ready` for 5 minutes, ECS rolls back by itself. | Quarterly rollback drill on `staging` |
| CR-041 | Smoke test | After every deploy, `deploy.yml` runs `scripts/smoke.ts` against the environment: `GET /api/health/ready` = 200, web `/` = 200, security headers present (CR-019, CR-025). On failure the deploy job fails and alerts `#ro-alerts`. | Workflow review |

---

## 2. Coding conventions

### 2.1 Naming

| Item | Convention | Example |
|---|---|---|
| Repo / packages | pnpm monorepo, package names `@ro/<name>` | `@ro/api`, `@ro/web`, `@ro/shared` |
| Folders and files (TS) | `kebab-case`; NestJS suffixes `.module.ts`, `.controller.ts`, `.service.ts`, `.dto.ts`, `.guard.ts`, `.processor.ts`, `.spec.ts`, `.e2e-spec.ts` | `analyses.controller.ts`, `create-analysis.dto.ts` |
| React components | `PascalCase` file and export, one component per file | `ScoreGauge.tsx` exports `ScoreGauge` |
| React hooks | `camelCase` starting with `use` | `useAnalysisPolling.ts` |
| Classes / interfaces / types | `PascalCase`; no `I` prefix | `AnalysisService`, `AnalysisResult` |
| Variables / functions | `camelCase`; booleans start with `is`/`has`/`can` | `remainingQuota`, `hasConsent()` |
| Constants | `UPPER_SNAKE_CASE`, defined in `packages/shared/src/config.ts` with the `CFG-` ID in a comment | `export const MAX_FILE_BYTES = 5_242_880; // CFG-01` |
| Enums | TS string union types, values `snake_case` (same as DB and API) | `type AnalysisStatus = 'queued' \| 'processing' \| 'completed' \| 'failed'` |
| DB tables | `snake_case`, plural | `analysis_suggestions` |
| DB columns | `snake_case`; FK = `<singular_table>_id`; timestamps end in `_at` | `job_role_id`, `created_at` |
| DB indexes / constraints | `ix_<table>_<cols>`, `ux_<table>_<cols>` (unique), `ck_<table>_<rule>`, `fk_<table>_<ref>` | `ix_analyses_user_id_created_at` |
| API paths | lowercase, plural nouns, `kebab-case`, no verbs, no trailing slash | `/api/v1/job-roles`, `/api/v1/analyses/{analysisId}/report.pdf` |
| API JSON fields | `camelCase` | `overallScore`, `createdAt` |
| Query parameters | `camelCase` | `?pageSize=20&cursor=...` |
| Error codes | `UPPER_SNAKE_CASE` | `QUOTA_EXCEEDED` |
| Env variables | `UPPER_SNAKE_CASE`, grouped by prefix | `DATABASE_URL`, `AI_MODE`, `AUTH_JWKS_URL` |
| BullMQ queues / jobs | `kebab-case` | queue `analysis`, job `run-analysis` |
| CSS | Tailwind utility classes; no custom class names except in `globals.css` (`ro-` prefix) | `ro-print-only` |
| Test IDs | `data-testid` in `kebab-case` | `data-testid="upload-dropzone"` |

### 2.2 Folder / module structure

```text
resume-optimizer/
├── apps/
│   ├── api/                         # NestJS 11 — API (main.ts) and worker (worker.ts)
│   │   ├── src/
│   │   │   ├── main.ts              # HTTP entrypoint
│   │   │   ├── worker.ts            # BullMQ worker + scheduler entrypoint (no HTTP)
│   │   │   ├── migrate.ts           # runs `prisma migrate deploy`, then exits
│   │   │   ├── app.module.ts
│   │   │   ├── worker.module.ts
│   │   │   ├── common/              # AppError, AppExceptionFilter, guards, interceptors, request-id
│   │   │   ├── config/              # env schema (zod), typed ConfigService
│   │   │   ├── auth/                # CMP-003 JwtVerifier, AuthGuard, RolesGuard, OriginGuard
│   │   │   ├── consents/            # CMP-004
│   │   │   ├── resumes/             # CMP-005, CMP-006 extractor/, CMP-007 redactor/
│   │   │   ├── job-roles/           # CMP-015 (+ admin controller)
│   │   │   ├── analyses/            # CMP-008 API side, CMP-009 quota/
│   │   │   ├── worker/              # CMP-010 processor, CMP-011 scoring/, CMP-012 ai/
│   │   │   ├── reports/             # CMP-013 PDF
│   │   │   ├── privacy/             # CMP-014 erasure + retention purge, data summary
│   │   │   ├── health/
│   │   │   └── prisma/              # PrismaService
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/          # SQL migrations (hand-edited for CHECK / partial indexes)
│   │   │   └── seed.ts              # job-role catalogue §5.5
│   │   └── test/                    # e2e specs + fixtures/ (§3.7)
│   └── web/                         # Next.js 15 App Router
│       ├── src/
│       │   ├── app/                 # routes, one folder per screen (§3.6)
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx                     # SCR-01 dashboard
│       │   │   ├── consent/page.tsx             # SCR-02
│       │   │   ├── analyses/new/page.tsx        # SCR-03
│       │   │   ├── analyses/[analysisId]/page.tsx  # SCR-04
│       │   │   ├── history/page.tsx             # SCR-05
│       │   │   ├── privacy/page.tsx             # SCR-06
│       │   │   └── admin/job-roles/page.tsx     # SCR-07
│       │   ├── components/          # shared UI (PascalCase files)
│       │   ├── features/            # feature folders: upload/, analysis/, history/, privacy/, admin/
│       │   ├── lib/api/             # typed fetch client, error mapper
│       │   ├── lib/format/          # date / number formatters (CR-030, CR-031)
│       │   └── middleware.ts        # CSP nonce (CR-025)
│       └── messages/en.json
├── packages/
│   └── shared/                      # @ro/shared: config constants, API types, enums, zod schemas
├── tools/
│   └── mock-platform/               # §4.8 local JWT issuer + JWKS
├── infra/                           # Terraform; envs/<env>.tfvars
├── scripts/                         # smoke.ts, check-log-pii.ts, check-ids.ts
├── docker-compose.yml               # postgres:16, redis:7, mock-platform
├── .github/workflows/               # ci.yml, deploy.yml
└── pnpm-workspace.yaml
```

Rules:
- A NestJS feature module MUST NOT import another feature module's repository or Prisma model directly; it calls the other module's exported service.
- `packages/shared` holds every type that crosses the API boundary. The web app MUST import API types from `@ro/shared`, never redefine them.

### 2.3 Branching and commits

- Trunk-based: one long-lived branch `main`, protected (no direct push, 1 approval, CI green, linear history via squash merge).
- Branch names: `feature/FR-008-start-analysis`, `fix/ERR-011-scanned-pdf-message`, `chore/<short-name>`, `docs/<short-name>`. Every feature/fix branch names an FR, NFR, CR or ERR ID.
- Commit format (Conventional Commits, enforced by `commitlint` with `@commitlint/config-conventional`): `<type>(<scope>): <message>`; types `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`; scope = module name (`api-analyses`, `web-upload`, …). Message in imperative, ≤ 72 chars. Footer `Refs: FR-008`.
  Example: `feat(api-analyses): reserve quota in same transaction as insert` / `Refs: FR-008, FR-009`.
- A PR title follows the same format; the squash commit uses the PR title.

### 2.4 Code review rules

- Every PR needs 1 approval from a code owner (`.github/CODEOWNERS`: `apps/api/` → backend team, `apps/web/` → frontend team, `apps/api/src/privacy/` and `apps/api/src/resumes/redactor/` → backend lead **and** one more reviewer).
- PR size SHOULD be ≤ 400 changed lines (excluding lockfiles and generated files).
- PR description template (`.github/pull_request_template.md`): what changed, requirement IDs, how it was tested, screenshots for UI, migration yes/no.
- Reviewer checklist (MUST be ticked): (1) no personal data in logs (CR-011); (2) new endpoints have DTO validation and an auth decorator; (3) new numbers come from `@ro/shared` config, not literals; (4) migrations are backward compatible (CR-039); (5) tests cover each acceptance criterion touched; (6) user-facing strings are in `en.json`.
- Review turnaround: within 1 working day.

### 2.5 Lint and format

| Tool | Config file | Rules | When it runs |
|---|---|---|---|
| ESLint 9 (flat config) | `eslint.config.mjs` at root | `@eslint/js` recommended, `typescript-eslint` `strictTypeChecked`, `eslint-plugin-import` (`import/order` alphabetised, groups builtin/external/internal), `next/core-web-vitals` for `apps/web`, `eslint-plugin-jsx-a11y` recommended for `apps/web`, `i18next/no-literal-string` for `apps/web/src/**/*.tsx`; `no-console: error` (use the logger); `@typescript-eslint/no-explicit-any: error`. | pre-commit (staged files), CI `pnpm lint` |
| Prettier 3 | `.prettierrc` | `{ "singleQuote": true, "semi": true, "trailingComma": "all", "printWidth": 100, "tabWidth": 2 }` + `prettier-plugin-tailwindcss` | pre-commit, CI `pnpm format:check` |
| TypeScript 5 | `tsconfig.base.json` | `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`, `noImplicitOverride: true` | CI `pnpm typecheck` |
| Husky 9 + lint-staged | `.husky/pre-commit`, `.husky/commit-msg`, `lint-staged` key in root `package.json` | pre-commit: `eslint --fix` + `prettier --write` on staged files; commit-msg: `commitlint --edit` | local git hooks |
| commitlint | `commitlint.config.mjs` | `extends: ['@commitlint/config-conventional']`, `scope-empty: [2, 'never']` | commit-msg hook, CI on PR title |
| Prisma | — | `prisma format` and `prisma validate` | CI |

Package manager: **pnpm 9**; Node.js **22 LTS** pinned in `.nvmrc` and `package.json#engines`.

---

## 3. Requirements definition

### 3.0 Glossary

| Term | Meaning |
|---|---|
| Resume | One uploaded file after text extraction. RO stores only its **redacted, encrypted text** and metadata, never the file (FR-004). |
| Analysis | One run of scoring + missing skills + suggestions for one resume, one job role and one experience level. |
| Job role | An entry in the job-role catalogue (TBL-001) with its skill list (TBL-002), e.g. "Frontend Developer". |
| Experience level | One of `entry` (0–2 years), `mid` (3–5), `senior` (6–9), `lead` (10+). Ordered `entry < mid < senior < lead`. |
| Dimension | One of the 5 ATS score parts in §3.5.3. |
| Suggestion | A proposed rewrite of one weak line of the resume (§3.5.5). |
| Quota period | Calendar month in `Asia/Kolkata`, written `YYYY-MM` (§3.5.6). |
| In-flight analysis | An analysis with status `queued` or `processing`. |

### 3.1 Functional requirements

| ID | Title | Description | Priority | Acceptance criteria |
|---|---|---|---|---|
| FR-001 | Authenticated access | Every RO screen and API (except health checks) requires a valid platform token. | Must | 1. A request without the `sb_access` cookie, or with a token failing any CR-003 rule, returns `401 UNAUTHENTICATED` (ERR-001). 2. The web app on any 401 redirects to `PLATFORM_LOGIN_URL?returnTo=<current URL>` (CR-005). 3. `GET /api/v1/me` (API-001) with a valid token returns `userId` equal to the token `sub`. 4. A token with `plan` missing is treated as `free` (CR-004). |
| FR-002 | Privacy notice and consent | Before the first upload, the user reads the privacy notice (§3.5.7) and gives explicit consent. Consent is recorded with the notice version. | Must | 1. `POST /resumes` (API-004) without a consent row for the current notice version `CFG-30` returns `403 CONSENT_REQUIRED` (ERR-005). 2. The consent screen (SCR-02) shows the notice text and an unticked checkbox "I agree…"; the "Continue" button is disabled until it is ticked. 3. `POST /consents` (API-003) stores one TBL-003 row with `notice_version`, `accepted_at`; repeating it for the same version returns 200 with the existing row, not a duplicate. 4. When `CFG-30` changes, every user must consent again before the next upload (criterion 1 applies). Existing resumes and analyses stay readable. 5. API-001 returns `consent.required = true` while there is no consent for the current version. |
| FR-003 | Upload resume | The user uploads one PDF or DOCX file. | Must | 1. Accepted types are decided by content (CR-023): PDF or DOCX. Anything else → `415 UNSUPPORTED_FILE_TYPE` (ERR-008). 2. Max size `CFG-01` (5 MiB = 5,242,880 bytes); larger → `413 FILE_TOO_LARGE` (ERR-007); the web app checks size before upload and shows the same message without calling the API. 3. A PDF with more than `CFG-02` (5) pages → `422 TOO_MANY_PAGES` (ERR-010). 4. Password-protected PDF or encrypted DOCX → `422 FILE_ENCRYPTED` (ERR-009). 5. Request without a `file` part → `400 FILE_REQUIRED` (ERR-006). 6. A user with `CFG-18` (10) stored resumes gets `409 RESUME_LIMIT_REACHED` (ERR-013). 7. On success the API returns `201` with the resume summary (API-004), within `NFR-002`. 8. Uploading a file whose normalised text has the same SHA-256 as one of the user's existing resumes returns `200` with that existing resume and `duplicate: true`; no new row is created. 9. The checks run in this order and the first failure is returned: consent (FR-002) → file present → size → type → resume limit → encryption → page count → extraction (FR-004). |
| FR-004 | Extract and store text | The API extracts plain text from the file in memory, normalises it, collects layout signals, redacts it (FR-005), encrypts it and stores it. The file itself is discarded. | Must | 1. PDF text is extracted with `pdfjs-dist`; DOCX with `mammoth` (`extractRawText`) plus a scan of `word/document.xml` for signals (§3.5.1). 2. Extraction is aborted after `CFG-19` (10 s) → `422 TEXT_NOT_EXTRACTABLE` (ERR-011). 3. Normalised text shorter than `CFG-03` (400) characters (e.g. a scanned PDF with no text layer) → `422 TEXT_NOT_EXTRACTABLE` (ERR-011) with message suggesting a text-based file. 4. Normalised text longer than `CFG-04` (30,000) characters → `422 TEXT_TOO_LONG` (ERR-012). 5. The original bytes are never written to disk, object storage, logs or the database; after the request the buffer is released. 6. TBL-004 stores: redacted text encrypted (AES-256-GCM, §5.2), `text_sha256` of the **normalised un-redacted** text, signals JSON, page count, file type, size, display name, `expires_at = created_at + CFG-09 (30 days)`. |
| FR-005 | Redact personal data | Contact details and identity numbers are removed from the text before it is stored or sent to the AI. | Must | 1. The redaction rules of §3.5.2 are applied in the listed order. 2. Before redaction, the presence of an email or phone is saved as signal `hasContact` (used by §3.5.3 D2). 3. The fixture `resume-contact.pdf` (§3.7) after redaction contains none of its email, phone, LinkedIn URL, PAN or Aadhaar strings and contains `[EMAIL]`, `[PHONE]`, `[URL]`, `[ID]` once each. 4. Text sent to the AI is always the stored redacted text; there is no code path that sends un-redacted text (checked by unit test that spies on the AI client). |
| FR-006 | Manage resumes | The user lists, views and deletes their stored resumes. | Must | 1. `GET /resumes` (API-005) returns the user's resumes, newest first, without the text. 2. `GET /resumes/{id}` (API-006) returns metadata and `expiresAt`; RO does not return the text to the client in v1. 3. `DELETE /resumes/{id}` (API-007) hard-deletes the row and returns 204; analyses made from it stay, with `resumeId = null` and `resumeDisplayName` kept. 4. Deleting a resume that has an in-flight analysis returns `409 ANALYSIS_IN_PROGRESS` (ERR-015). 5. Another user's resume ID → 404 (CR-008). |
| FR-007 | Choose job role and experience level | The user picks a target job role from the catalogue and an experience level. | Must | 1. `GET /job-roles` (API-008) returns all **active** roles ordered by `sort_order`, then `name`. 2. Each role shows `name`, `slug`, `description`. 3. The four experience levels and their year ranges are fixed (§3.0) and shown in SCR-03 as radio buttons; default selection is none (user must choose). |
| FR-008 | Start analysis | The user starts an analysis for one stored resume, one active job role and one level. | Must | 1. `POST /analyses` (API-009) with valid input returns `202` with `status = "queued"` within `NFR-001`. 2. A user with an in-flight analysis gets `409 ANALYSIS_IN_PROGRESS` (ERR-015); enforced by a partial unique index (TBL-005) so it also holds under concurrent requests. 3. An unknown or inactive `jobRoleId` → `422 INVALID_JOB_ROLE` (ERR-016). 4. Unknown `resumeId` or one owned by another user → `404 NOT_FOUND` (ERR-004). 5. Quota is checked and reserved **in the same DB transaction** as the insert (FR-009). 6. A job `run-analysis` with `{ analysisId, requestId }` is added to queue `analysis` after the transaction commits. If adding the job fails, the analysis is set to `failed` with `AI_UNAVAILABLE` (ERR-022), quota is released, and the API returns `503 SERVICE_UNAVAILABLE` (ERR-019). |
| FR-009 | Monthly quota | Free users get `CFG-05` (3) analyses per quota period, Pro users `CFG-06` (50). | Must | 1. Quota counts analyses created in the current quota period (§3.5.6) that did not end `failed`. 2. When `used >= limit` for the plan **at request time**, API-009 returns `403 QUOTA_EXCEEDED` (ERR-014) with `details: [{ field: "quota", issue: "max", limit: <limit> }]`. 3. A failed analysis releases its reservation (FR-018). 4. API-001 returns `quota: { period, limit, used, remaining, resetsAt }`; `resetsAt` = first instant of the next month in `Asia/Kolkata`, in UTC. 5. A user who upgrades mid-month to Pro gets limit 50 with the already-used count kept (e.g. used 3 → remaining 47). 6. 10 parallel `POST /analyses` from a Free user with `used = 2` result in exactly one accepted request (the others get 409 or 403). |
| FR-010 | ATS score | The system scores the resume out of 100 on 5 dimensions for the chosen role and level. | Must | 1. Dimensions, weights and formulas are exactly those in §3.5.3. 2. Each dimension score is an integer 0–100; `overallScore = round(Σ weight × score / 100)` (half up). 3. The band is set from `overallScore` per §3.5.3. 4. Each dimension has a one-sentence explanation (templates in §3.5.3; D4 from the AI). 5. Running the same analysis twice with `AI_MODE=mock` gives identical scores (D1, D2, D3, D5 are deterministic). 6. Fixture `resume-frontend-good.pdf` for role `frontend-developer`, level `mid` scores D1 ≥ 70 (§3.7). |
| FR-011 | Missing skills | The system lists the role's skills not found in the resume. | Must | 1. The skill set and matching rules are in §3.5.4. 2. Each missing skill shows `name` and `importance` (`must_have` / `nice_to_have`). 3. Order: `must_have` first, then `nice_to_have`; within each, catalogue `sort_order`. 4. Missing skills come **only** from the catalogue; the AI does not add skills. 5. Free and Pro users see the full list. |
| FR-012 | Rewrite suggestions | The AI proposes rewrites for up to `CFG-08` (10) weak lines. | Must | 1. Candidate lines, AI output contract and validation are in §3.5.5 and §4.7. 2. Every stored suggestion's `originalText` is exactly a line of the redacted resume text. 3. A rewrite never contains a number that is not in the original line; a missing figure is shown as a placeholder `[number]`, `[percent]`, `[amount]` or `[count]` for the user to fill. 4. Suggestions are ranked `severity` desc, then `lineIndex` asc, `rank` = 1..n. 5. If the AI returns 0 valid suggestions, the analysis still completes with an empty list and the UI shows "No weak lines found." |
| FR-013 | Plan gating of suggestions | Free users see only part of the suggestions. | Must | 1. For a Free user, API-011 returns suggestions with `rank <= CFG-07 (3)` in full; for the others it returns only `rank`, `issue` and `locked: true` (no `originalText`, `rewriteText`, `rationale`). 2. `suggestionsTotal` and `suggestionsLocked` counts are returned. 3. Gating uses the plan of the **current request** (CR-009): after upgrading to Pro, the same analysis shows all suggestions without re-running. 4. The PDF report (FR-017) applies the same gating. 5. All suggestions are always stored (gating is at read time). |
| FR-014 | View analysis status and result | The user sees progress and then the result. | Must | 1. `GET /analyses/{id}` (API-011) returns `status`; when `completed` it also returns scores, missing skills and suggestions; when `failed` it returns `failureCode` and `failureMessage` (§6.3). 2. SCR-04 polls API-011 every `CFG-20` (2 s) while status is `queued` or `processing`, and stops on `completed`/`failed` or after 5 minutes (then shows "This is taking longer than usual. We'll keep working — check History later."). 3. Polling pauses while the browser tab is hidden (`document.visibilityState = "hidden"`) and resumes when visible. |
| FR-015 | Analysis history | The user sees past analyses. | Must | 1. `GET /analyses` (API-010) returns the user's analyses newest first, cursor-paginated (§6.1), default page size 20, max 50. 2. Each item shows role name, level, status, overall score (null unless completed), resume display name, created date. 3. Optional filter `status`. |
| FR-016 | Delete analysis | The user deletes one analysis. | Must | 1. `DELETE /analyses/{id}` (API-012) hard-deletes the analysis and its scores, missing skills and suggestions; returns 204. 2. An in-flight analysis cannot be deleted → `409 ANALYSIS_IN_PROGRESS` (ERR-015). 3. Deleting does **not** give quota back (quota counts analyses run, FR-009). |
| FR-017 | Download PDF report | The user downloads a completed analysis as a PDF. | Must | 1. `GET /analyses/{id}/report.pdf` (API-013) returns `200`, `Content-Type: application/pdf`, `Content-Disposition: attachment; filename="resume-analysis-<YYYY-MM-DD>-<roleSlug>.pdf"` (date = analysis `completedAt` in IST). 2. Layout and content per §4.9. 3. Analysis not `completed` → `409 ANALYSIS_NOT_READY` (ERR-017). 4. The PDF is generated on request and never stored. 5. Generated within `NFR-003`. 6. Free user's PDF shows only the first `CFG-07` (3) suggestions and the line "N more suggestions are available with SurfingBear Pro." |
| FR-018 | AI failure handling | AI timeouts, errors and invalid output do not leave analyses stuck and do not cost quota. | Must | 1. Each AI call has a timeout `CFG-12` (45 s). 2. A call that times out, returns HTTP 429/5xx, or returns output failing the §4.7 schema is retried up to `CFG-13` (3 attempts total) with delays 2 s and 8 s. 3. After the last failed attempt the analysis becomes `failed` with `AI_UNAVAILABLE` (ERR-022) for timeouts/429/5xx or `AI_INVALID_OUTPUT` (ERR-023) for schema failures. 4. On `failed`, the quota reservation is released in the same transaction (`quota_released = true`). 5. While the AI circuit breaker is open (§4.7.4), new analyses are still accepted and queued; the worker delays jobs until it closes. An analysis that is still not finished `CFG-14` (10 min) after creation is failed by the sweep (FR-019) with `PROCESSING_TIMEOUT` and its quota is released. 6. SCR-04 for a failed analysis shows the failure message and a "Try again" button that starts a new analysis with the same inputs. |
| FR-019 | Stuck analysis recovery | Analyses that never finish are failed automatically. | Must | 1. Every 1 minute the scheduler (CMP-014) sets every analysis in `queued` or `processing` for longer than `CFG-14` (10 min) since `created_at` to `failed` with `PROCESSING_TIMEOUT` (ERR-024) and releases quota. 2. A worker that later finishes such an analysis does not overwrite it: the final update has `WHERE status = 'processing'` and its result is discarded if 0 rows change. |
| FR-020 | Erase all my data | The user withdraws consent and erases all their RO data. | Must | 1. SCR-06 "Delete all my data" asks the user to type `DELETE` to confirm. 2. `DELETE /me/data` (API-015) deletes, in one transaction, all the user's TBL-003, TBL-004, TBL-005 (with children) and TBL-008 rows, and returns `204`. 3. In-flight analyses are deleted too; the worker treats a missing analysis as "drop the job" without error. 4. The user's Redis keys (`rl:*:<userId>`) are deleted. 5. One TBL-009 audit row `data_erased` is written with `user_hash` only (§5.2). 6. After erasure, API-001 shows `consent.required = true` and `quota.used = 0` — note: quota resets because the counter row is erased (accepted, see A-09). |
| FR-021 | Automatic retention purge | Data is deleted when its retention ends. | Must | 1. Daily at 02:00 Asia/Kolkata the purge job hard-deletes: resumes with `expires_at <= now()` that have no in-flight analysis (those are picked up the next day); analyses with `created_at <= now() - CFG-10 (180 days)` (children cascade); audit rows older than `CFG-11` (365 days); usage counters for periods older than 13 months. 2. Deletes run in batches of 1,000 rows. 3. The job logs counts only (no IDs of personal data rows other than UUIDs). 4. A resume past `expires_at` but not yet purged is treated as deleted by every API (404). |
| FR-022 | Admin: manage job-role catalogue | An admin adds and edits job roles and their skills. | Should | 1. API-016 lists all roles including inactive. 2. API-017 creates a role; `slug` must match `^[a-z0-9]+(-[a-z0-9]+)*$`, 3–60 chars, unique → else `409 ROLE_SLUG_TAKEN` (ERR-021). 3. API-018 edits name, description, sort order, active flag; slug cannot change. 4. API-019 replaces a role's skill list (1–40 skills). 5. Roles are never deleted (analyses reference them); deactivate instead. 6. Every change writes a TBL-009 audit row `job_role_changed` with the admin's `user_hash`. 7. Non-admin → `403 FORBIDDEN` (ERR-002). |
| FR-023 | My data summary | The user sees what RO stores about them (DPDP right to information). | Should | 1. `GET /me/data-summary` (API-014) returns counts of resumes and analyses, the consent version and date, the retention rules (`CFG-09`, `CFG-10`), and the Grievance Officer email. 2. SCR-06 shows it. |
| FR-024 | Dashboard | A landing screen shows the user's quota and recent activity. | Should | 1. SCR-01 shows remaining analyses for this month, plan badge, the 5 latest analyses (API-010 `pageSize=5`) and the "New analysis" button. 2. When `remaining = 0`, the button is disabled and the text "You have used all N analyses for this month. Resets on <date>." is shown; Free users also see "Upgrade to Pro" linking to `PLATFORM_UPGRADE_URL`. |

### 3.2 Non-functional requirements

| ID | Category | Requirement | Measurable target | How it is verified |
|---|---|---|---|---|
| NFR-001 | Performance | API latency for JSON endpoints (all except API-004 and API-013) | p95 < 300 ms, p99 < 800 ms at 50 req/s | k6 script `perf/api.js` on `staging` before each prod release |
| NFR-002 | Performance | Upload + extraction + redaction (API-004) | p95 < 4 s for a 5 MiB, 5-page PDF | k6 with fixture `resume-5mb.pdf` |
| NFR-003 | Performance | PDF report generation (API-013) | p95 < 3 s | k6 |
| NFR-004 | Performance | End-to-end analysis time (`completed_at - created_at`) | p95 < 45 s, p50 < 20 s, measured over 7 days in `prod` | Prometheus histogram `ro_analysis_duration_seconds` |
| NFR-005 | Availability | Monthly availability of API (`/api/health/ready` + API-001 synthetic check every 60 s) | ≥ 99.5 % per calendar month (≤ 3 h 39 m downtime) | CloudWatch Synthetics canary |
| NFR-006 | Availability | Degraded mode when the AI provider is down | Upload, history, view and PDF of past analyses keep working (0 errors on those endpoints); analyses created during the outage complete after recovery if the outage ends within `CFG-14` (10 min) of their creation, otherwise they end `failed` / `PROCESSING_TIMEOUT` with quota released (FR-018 AC 5) | Outage test: block Bedrock egress on `staging` for 5 min, then for 15 min |
| NFR-007 | Scalability | Load the design must carry without code change | 2,000 analyses/day, 100 concurrent in-flight analyses, 500 concurrent web users; worker scales horizontally (ECS auto-scaling on queue depth > 20 waiting jobs for 2 min → +1 task, max 6) | k6 soak test 1 h on `staging` |
| NFR-008 | Security | OWASP ASVS L2 controls for auth, session, input, files | 0 high/critical findings in OWASP ZAP baseline scan and `pnpm audit --prod` | CI weekly job + pre-release |
| NFR-009 | Security | Dependencies | No dependency with a known critical CVE in prod images | Trivy scan in CI fails on CRITICAL |
| NFR-010 | Privacy | No personal data in logs, metrics or traces | 0 matches of fixture PII in CI log scan (CR-011) | `scripts/check-log-pii.ts` in CI |
| NFR-011 | Privacy | Data residency | All RO data stores and the AI endpoint are in AWS `ap-south-1` | Terraform review; AWS Config rule `allowed-regions` |
| NFR-012 | Privacy | Retention enforced | No resume row older than 31 days and no analysis older than 181 days in `prod` at any time | Daily CloudWatch alarm query |
| NFR-013 | Reliability | No lost analyses | 100 % of analyses reach `completed` or `failed` within `CFG-14` (10 min) | Alarm on count of in-flight analyses older than 10 min > 0 |
| NFR-014 | Backup / DR | Database recovery | RPO ≤ 5 min (RDS PITR), RTO ≤ 4 h; backups retained 7 days (short on purpose, CR-027) | Quarterly restore drill on `staging` |
| NFR-015 | Observability | Metrics, traces, alerts | Metrics listed in §4.10 exported; alerts in §4.10 fire to `#ro-alerts` within 5 min | Alert test on `staging` |
| NFR-016 | Accessibility | Web UI | WCAG 2.2 AA; 0 serious/critical `axe-core` violations on every screen; full keyboard path for upload → result | `@axe-core/playwright` in e2e; manual keyboard pass before release |
| NFR-017 | Compatibility | Browsers and widths | Latest 2 versions of Chrome, Edge, Firefox, Safari (macOS + iOS); layouts work from 360 px to 1920 px wide | Playwright projects chromium/firefox/webkit; screenshots at 360, 768, 1280, 1920 |
| NFR-018 | Web performance | Web vitals on SCR-01 and SCR-04 | LCP < 2.5 s, INP < 200 ms, CLS < 0.1 at p75 on 4G mobile profile | Lighthouse CI in PR (`lighthouserc.json`, budgets) |
| NFR-019 | Cost | AI cost per analysis | ≤ USD 0.03 average (≈ 12k input + 2k output tokens) | Monthly Bedrock bill ÷ analyses count |
| NFR-020 | Maintainability | Test coverage | ≥ 80 % lines in `apps/api/src`, ≥ 70 % in `apps/web/src/features`; every FR acceptance criterion has at least one test whose name starts with its ID (e.g. `FR-003.2 rejects 6 MB file`) | Jest coverage gate; `scripts/check-ids.ts` |

### 3.3 Assumptions and out of scope

**Assumptions** (decisions not given in the brief, with the reason):

| ID | Assumption | Reason |
|---|---|---|
| A-01 | The platform issues an RS256 JWT in cookie `sb_access` on `.surfingbear.com`, publishes a JWKS, and puts `sub`, `plan`, `roles` in the token. | "Users are already logged in through the main platform" — RO must only verify. JWKS avoids a network call to the platform per request. |
| A-02 | RO is served under `resume.surfingbear.com`, so the platform cookie is sent to it. | Same parent domain is required for the cookie to reach RO without a token exchange. |
| A-03 | Plans are only `free` and `pro`; limits are 3 and 50 analyses per month. | 3 is enough to try the product before paying; 50 is a fair-use cap that stops abuse and caps AI cost (NFR-019). |
| A-04 | The original file is not kept. | DPDP data minimisation: the analysis needs only the text. Removes object storage and its breach surface. Trade-off: the user cannot download their original file from RO, and re-extraction needs a re-upload. |
| A-05 | Scanned PDFs (no text layer) are rejected, not OCR'd. | OCR adds cost, latency and error; ATS systems also fail on such files, so telling the user is itself useful. |
| A-06 | The AI provider is Claude on **AWS Bedrock in `ap-south-1`**. | Data stays in India and inside the AWS account (NFR-011); Bedrock does not use prompts for training; IAM auth means no extra API key secret. |
| A-07 | Skill matching is English-only. | The catalogue is in English, as are almost all Indian tech resumes. |
| A-08 | Free users see all scores and missing skills; only suggestions are gated. | The brief gates "some of the suggestions" only. Showing the score shows the value of Pro. |
| A-09 | Erasing all data also erases the month's usage counter, so a user could reset quota by erasing. | DPDP erasure must be complete; the abuse cost is low (they lose all data and must re-upload). Revisit if abuse is seen. |
| A-10 | One in-flight analysis per user. | Keeps AI cost and queue fairness simple; an analysis takes < 45 s (NFR-004). |
| A-11 | RO does not send email. Notifications are on-screen only. | Not in the brief; avoids handling email addresses at all. |

**Out of scope (v1)**: sign-up, login, logout, password reset, payments and plan purchase (platform); editing the resume inside RO or exporting an edited resume; cover letters; job matching / job search; OCR; languages other than English in UI and matching; mobile apps; sharing analyses with other users; admin access to user content; email notifications; analytics dashboards for admins.

### 3.4 Configuration constants

All values live in `packages/shared/src/config.ts`. Values marked _env_ can be overridden per environment by the env variable shown; all others are code constants (change = release).

| ID | Name | Value | Env override | Used by |
|---|---|---|---|---|
| CFG-01 | `MAX_FILE_BYTES` | 5,242,880 (5 MiB) | — | FR-003, API-004, ERR-007, SCR-03 |
| CFG-02 | `MAX_PDF_PAGES` | 5 | — | FR-003, ERR-010 |
| CFG-03 | `MIN_TEXT_CHARS` | 400 | — | FR-004, ERR-011 |
| CFG-04 | `MAX_TEXT_CHARS` | 30,000 | — | FR-004, ERR-012 |
| CFG-05 | `FREE_MONTHLY_ANALYSES` | 3 | `FREE_MONTHLY_ANALYSES` | FR-009, API-001 |
| CFG-06 | `PRO_MONTHLY_ANALYSES` | 50 | `PRO_MONTHLY_ANALYSES` | FR-009, API-001 |
| CFG-07 | `FREE_VISIBLE_SUGGESTIONS` | 3 | — | FR-013, FR-017 |
| CFG-08 | `MAX_SUGGESTIONS` | 10 | — | FR-012, §4.7 |
| CFG-09 | `RESUME_RETENTION_DAYS` | 30 | — | CR-027, FR-004, FR-021 |
| CFG-10 | `ANALYSIS_RETENTION_DAYS` | 180 | — | CR-027, FR-021 |
| CFG-11 | `AUDIT_RETENTION_DAYS` | 365 | — | CR-027, FR-021 |
| CFG-12 | `AI_TIMEOUT_MS` | 45,000 | `AI_TIMEOUT_MS` | FR-018, §4.7 |
| CFG-13 | `AI_MAX_ATTEMPTS` | 3 (delays 2,000 ms then 8,000 ms) | — | FR-018 |
| CFG-14 | `ANALYSIS_STUCK_MINUTES` | 10 | — | FR-019, NFR-013 |
| CFG-15 | `RATE_LIMIT_DEFAULT` | 120 requests / 60 s / user | — | CR-024 |
| CFG-16 | `RATE_LIMIT_UPLOAD` | 10 requests / 3,600 s / user | — | CR-024 |
| CFG-17 | `RATE_LIMIT_ANALYSIS` | 20 requests / 3,600 s / user | — | CR-024 |
| CFG-18 | `MAX_STORED_RESUMES` | 10 per user | — | FR-003, ERR-013 |
| CFG-19 | `EXTRACT_TIMEOUT_MS` | 10,000 | — | FR-004 |
| CFG-20 | `POLL_INTERVAL_MS` | 2,000 (stop after 300,000) | — | FR-014 |
| CFG-21 | `JWKS_CACHE_MS` | 600,000 (10 min) | — | CR-002 |
| CFG-22 | `JWKS_MIN_REFRESH_MS` | 60,000 | — | CR-002 |
| CFG-23 | `JWT_CLOCK_SKEW_S` | 30 | — | CR-003 |
| CFG-24 | `MAX_JSON_BODY_BYTES` | 102,400 (100 KB) | — | CR-015 |
| CFG-25 | `DOCX_MAX_UNCOMPRESSED_BYTES` | 20,971,520 (20 MiB) | — | CR-023 |
| CFG-26 | `DOCX_MAX_ENTRIES` | 500 | — | CR-023 |
| CFG-27 | `AI_DAILY_CALL_CAP` | unlimited in `prod`/`staging`; 200 in `dev` | `AI_DAILY_CALL_CAP` (0 = unlimited) | §1.7, §4.7.4 |
| CFG-28 | `AI_BREAKER` | opens after 5 consecutive failed calls; stays open 60 s; then 1 trial call | — | FR-018, §4.7.4 |
| CFG-29 | `MAX_AI_CANDIDATE_LINES` | 80 | — | §3.5.5 |
| CFG-30 | `CONSENT_NOTICE_VERSION` | `"2026-10-v1"` | — | FR-002, §3.5.7 |
| CFG-31 | `MAX_DISPLAY_NAME_CHARS` | 120 | — | FR-004, TBL-004 |
| CFG-32 | `PAGE_SIZE` | default 20, max 50 | — | §6.1 |

### 3.5 Processing and scoring rules

#### 3.5.1 Text extraction, normalisation and signals

1. **PDF** (`pdfjs-dist`, legacy Node build, `disableFontFace: true`, `isEvalSupported: false`): if `getDocument` throws `PasswordException` → ERR-009. Pages > `CFG-02` → ERR-010 (checked from `numPages` before reading text). For each page, join `textContent.items[].str`; insert `\n` when an item has `hasEOL = true` or its y-position differs from the previous item by more than half its height. Pages joined with `\n\n`. Signal `imageCount` = count of `paintImageXObject` ops over all pages (`getOperatorList`).
2. **DOCX**: zip limits per CR-023. If the zip contains `EncryptionInfo` or is an OLE compound file (`D0 CF 11 E0` magic) → ERR-009. Text = `mammoth.extractRawText`. Signals from `word/document.xml`: `hasTables` = contains `<w:tbl>`; `hasTextBoxes` = contains `<w:txbxContent>`; `imageCount` = number of `<w:drawing>` elements. `pageCount` = `<Pages>` from `docProps/app.xml` if present, else `ceil(chars / 3000)`.
3. **Normalisation** (both types, in order): Unicode NFC → replace `\r\n` and `\r` with `\n` → replace tabs and non-breaking spaces with a space → collapse runs of spaces to one → trim each line → remove lines that are empty after trim **only** when more than 2 in a row (keep at most one blank line) → trim the whole text.
4. **Lines.** After normalisation, the text is split on `\n`; `lineIndex` is the 0-based index in this array. The same array is used by §3.5.3 and §3.5.5.
5. **Signals JSON** stored in TBL-004 `signals`: `{ "hasContact": bool, "hasTables": bool, "hasTextBoxes": bool, "imageCount": int, "charCount": int, "lineCount": int }` (for PDF `hasTables` and `hasTextBoxes` are always `false`).

#### 3.5.2 Redaction (FR-005)

Applied to the normalised text, in this order (regexes are JavaScript, flag `g`; `i` where marked):

| # | Data | Regex | Replace with |
|---|---|---|---|
| 1 | Email | `/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g` | `[EMAIL]` |
| 2 | URL | `/\b(?:https?:\/\/|www\.)[^\s]+|\b(?:linkedin\.com|github\.com|behance\.net)\/[^\s]+/gi` | `[URL]` |
| 3 | Aadhaar | `/\b\d{4}[ -]?\d{4}[ -]?\d{4}\b/g` | `[ID]` |
| 4 | PAN | `/\b[A-Z]{5}\d{4}[A-Z]\b/g` | `[ID]` |
| 5 | Phone | `/(?:\+?\d{1,3}[ -]?)?(?:\(?\d{2,5}\)?[ -]?)?\d{3,5}[ -]?\d{4,5}\b/g` — only when the match has 10–13 digits in total | `[PHONE]` |
| 6 | Name | The first non-empty line, if it has 2–4 words, no digits, and no `[` | `[NAME]` |

`hasContact` = rule 1 or rule 5 matched at least once (computed before replacing). The un-redacted text exists only in memory during the upload request.

#### 3.5.3 ATS dimensions and score (FR-010)

**Section detection.** A line is a **heading** if it has ≤ 40 characters and, lowercased with trailing `:` removed, equals one of these synonyms:

| Section key | Synonyms |
|---|---|
| `summary` | summary, profile, professional summary, about me, objective, career objective |
| `experience` | experience, work experience, professional experience, employment, employment history, work history |
| `education` | education, academic background, qualifications, academics |
| `skills` | skills, technical skills, key skills, core competencies, skills & tools |
| `projects` | projects, personal projects, key projects |
| `certifications` | certifications, certificates, licenses & certifications |

A section runs from the line after its heading to the line before the next heading (or the end). If a key appears twice, both ranges count. A **bullet line** is a non-empty, non-heading line in the `experience` or `projects` section with at least 4 words.

| Dim | Key | Weight | Scored by | Formula (result is an integer 0–100) |
|---|---|---|---|---|
| D1 | `keyword_match` | 30 | code | Skill set S = role skills with `min_level <= chosen level` (TBL-002). Weight `must_have` = 2, `nice_to_have` = 1. Matched per §3.5.4. `round(100 × Σ weight(matched) / Σ weight(S))`. |
| D2 | `section_structure` | 15 | code | +20 each for `experience`, `education`, `skills` found; +20 if `hasContact`; +20 if `summary` found. |
| D3 | `parseability` | 15 | code | Start 100, subtract: `hasTables` −25; `hasTextBoxes` −25; `imageCount > 2` −10; average chars per page < 500 −30; page count above the level limit (entry/mid: 2, senior: 3, lead: 4) −15. Floor 0. |
| D4 | `experience_relevance` | 20 | AI | `experienceRelevance.score` from the AI (§4.7), integer 0–100. |
| D5 | `impact` | 20 | code | B = bullet lines; Q = bullet lines matching `/\d|%|₹|\$|\b(lakh|crore|million|thousand)\b/i`. If B = 0 → 0. Else `round(min(1, (Q/B) / 0.5) × 100)` (50 % quantified bullets = full score). |

`overallScore = round((30·D1 + 15·D2 + 15·D3 + 20·D4 + 20·D5) / 100)`, half up.

| Band | Range | Label |
|---|---|---|
| `needs_work` | 0–49 | Needs work |
| `fair` | 50–69 | Fair |
| `good` | 70–84 | Good |
| `excellent` | 85–100 | Excellent |

**Explanation templates** for code-scored dimensions (stored in TBL-006 `explanation`):
- D1: `"Found {matched} of {total} skills expected for a {level} {roleName}."`
- D2: `"Found: {comma-separated found parts}. Missing: {comma-separated missing parts or 'none'}."` — parts are named Experience, Education, Skills, Contact details, Summary.
- D3: `"No layout problems found."` if 100, else `"Possible ATS problems: {comma-separated reasons}."` with reasons: tables, text boxes, many images, little selectable text, longer than {limit} pages.
- D5: `"{Q} of {B} experience lines include a number or measurable result."`

#### 3.5.4 Skill matching and missing skills (FR-011)

- Each skill has a `name` and `aliases` (TBL-002). Terms = `name` + aliases, lowercased.
- A term matches if the lowercased resume text contains it as a whole term: regex `(?<![a-z0-9+#.])` + escaped term + `(?![a-z0-9+#])`. (So `java` does not match `javascript`, `c` is never used as a term, `c++` and `c#` work.)
- A skill is **matched** if any of its terms matches anywhere in the text (all sections).
- Missing skills = S minus matched, ordered per FR-011 AC 3.

#### 3.5.5 Rewrite suggestions (FR-012)

1. **Candidate lines** = bullet lines (§3.5.3) in order; if there are more than `CFG-29` (80), keep the first 80. If there are none, the AI call is still made (for D4) with an empty candidate list and suggestions are empty.
2. Each candidate is sent to the AI as `{ "i": lineIndex, "t": lineText }` (§4.7).
3. **Validation** of each returned suggestion; a suggestion failing any rule is dropped (not an error):
   1. `lineIndex` is one of the sent candidates; at most one suggestion per `lineIndex` (keep the first).
   2. `issue` ∈ `weak_verb`, `no_metric`, `vague`, `too_long`, `passive`, `jargon`.
   3. `rewrite` is 10–300 characters and differs from the original line (case-insensitive, whitespace-trimmed).
   4. Every digit run (`/\d+/g`) in `rewrite`, outside placeholders, also appears in the original line. Allowed placeholders: `[number]`, `[percent]`, `[amount]`, `[count]`.
   5. `rewrite` does not contain `[EMAIL]`, `[PHONE]`, `[URL]`, `[ID]` or `[NAME]`.
   6. `severity` ∈ 1, 2, 3 (3 = most important).
4. Keep at most `CFG-08` (10) after validation, ranked per FR-012 AC 4. `originalText` is set by RO from the line array, not taken from the AI.

#### 3.5.6 Quota period (FR-009)

`period = format(toZonedTime(now, 'Asia/Kolkata'), 'yyyy-MM')` (`date-fns-tz`). The quota row is TBL-008 `(user_id, period)`. Reservation, in the API-009 transaction:

```sql
INSERT INTO usage_counters (user_id, period, used) VALUES ($1, $2, 0)
  ON CONFLICT (user_id, period) DO NOTHING;
SELECT used FROM usage_counters WHERE user_id = $1 AND period = $2 FOR UPDATE;
-- if used >= limit(plan): ROLLBACK, return ERR-014
UPDATE usage_counters SET used = used + 1, updated_at = now() WHERE user_id = $1 AND period = $2;
INSERT INTO analyses (...) VALUES (...);   -- partial unique index may raise → ERR-015
```

Release (on `failed`): `UPDATE usage_counters SET used = GREATEST(used - 1, 0) WHERE user_id = $1 AND period = <analysis.quota_period>` and `UPDATE analyses SET quota_released = true`, in the same transaction as the status change. The period of the analysis (`quota_period` column), not the current one, is released.

#### 3.5.7 Privacy notice text (version `2026-10-v1`)

Shown on SCR-02 and returned by API-002 (`noticeMarkdown`). Stored in `apps/api/src/consents/notice/2026-10-v1.md`.

> **How Resume Optimizer uses your resume**
>
> - **What we collect:** the text of the resume you upload, and the job role and experience level you choose. We do not keep the file itself.
> - **Why:** only to score your resume, list missing skills and suggest better wording.
> - **What we remove first:** your name (first line), email, phone number, links, PAN and Aadhaar numbers are replaced before the text is saved or analysed.
> - **Who processes it:** the text is analysed by an AI model running on Amazon Web Services in Mumbai, India. It is not used to train any AI model.
> - **How long we keep it:** resume text for 30 days, analysis results for 180 days. You can delete any item, or all your data, at any time from Privacy settings.
> - **Your rights:** you may withdraw consent at any time by deleting all your data. For questions or complaints, contact our Grievance Officer at grievance@surfingbear.com.
>
> ☐ I have read this notice and I agree to SurfingBear processing my resume for this purpose.

### 3.6 Web screens

All screens use the root layout: header with "Resume Optimizer" logo (links to SCR-01), plan badge (`Free` / `Pro`), links "History" (SCR-05) and "Privacy" (SCR-06), and "Admin" (SCR-07) only when the user is admin (from API-001 `isAdmin`). On first load the layout calls API-001; on 401 it redirects (CR-005).

| ID | Route | Purpose | Data | States and behaviour |
|---|---|---|---|---|
| SCR-01 | `/` | Dashboard (FR-024) | API-001, API-010 `pageSize=5` | Loading skeleton; empty state "No analyses yet" + "New analysis" button; quota card "{remaining} of {limit} analyses left this month · resets {resetsAt date}". |
| SCR-02 | `/consent?next=<path>` | Privacy notice + consent (FR-002) | API-002, API-003 | Shown automatically when the user goes to SCR-03 and API-001 `consent.required = true`. After API-003 succeeds, navigate to `next` (only relative paths starting with `/` are accepted; otherwise `/`). |
| SCR-03 | `/analyses/new` | Upload or pick resume, choose role and level, start (FR-003, FR-007, FR-008) | API-005, API-004, API-008, API-009 | Step 1: drop zone (`data-testid="upload-dropzone"`, accepts `.pdf,.docx`, also keyboard-operable "Choose file" button) or a list of stored resumes as radio buttons. Client-side size check (CFG-01). Upload shows a progress bar. Step 2: role `<select>` (API-008) and level radio group. Step 3: "Analyse" button → API-009 → navigate to SCR-04. Each error code shows its `errors.<CODE>` text inline under the relevant control. |
| SCR-04 | `/analyses/[analysisId]` | Progress and result (FR-010 – FR-014, FR-017) | API-011 polling, API-013 | `queued`/`processing`: spinner with "Analysing your resume… usually under a minute." (`aria-live="polite"`). `completed`: score gauge (overall + band), 5 dimension bars with explanations, missing skills chips grouped by importance, suggestion cards (original → rewrite, issue tag, rationale; placeholders highlighted), locked cards for Free users with "Unlock all suggestions with Pro" (link `PLATFORM_UPGRADE_URL`), "Download PDF" button. `failed`: message for `failureCode` and "Try again". |
| SCR-05 | `/history` | History (FR-015, FR-016) | API-010, API-012 | Table: date, role, level, score, status, resume name, actions (View, Delete with confirm dialog). "Load more" button for the next cursor. Status filter. |
| SCR-06 | `/privacy` | My data (FR-006, FR-020, FR-023) | API-014, API-005, API-007, API-015 | Data summary; stored resumes with Delete; "Delete all my data" danger zone (type `DELETE` to enable). After API-015 success, show "All your data has been deleted." and link to `/`. |
| SCR-07 | `/admin/job-roles` | Catalogue admin (FR-022) | API-016 – API-019 | Only reachable when `isAdmin`; otherwise render a 403 page. Table of roles; edit drawer with name, description, sort order, active toggle, and a skills editor (name, aliases comma-separated, importance, min level). |

Web data fetching uses TanStack Query 5. The API client (`lib/api`) sends `credentials: "same-origin"`, parses the §1.4 error body, and throws `ApiError { code, message, status, details }`.

### 3.7 Test fixtures

Stored in `apps/api/test/fixtures/` and generated by `pnpm fixtures` (`scripts/make-fixtures.ts`, using `pdfkit` and `docx`), so their content is exactly known.

| File | Content | Expected result |
|---|---|---|
| `resume-frontend-good.pdf` | 2 pages; sections Summary, Experience (8 bullets, 5 with numbers), Skills (React, TypeScript, Next.js, HTML, CSS, Redux, Jest, Git, REST, Tailwind), Education; email + phone | 201; D1 ≥ 70 for `frontend-developer`/`mid`; D2 = 100; D3 = 100; D5 = 100 |
| `resume-contact.pdf` | Line 1 `Asha Verma`; contains `asha.verma@example.com`, `+91 98765 43210`, `linkedin.com/in/ashaverma`, `ABCDE1234F`, `1234 5678 9012` | Redaction per FR-005 AC 3 |
| `resume-docx-tables.docx` | Skills in a table, one text box, 3 images | 201; D3 = 100 − 25 − 25 − 10 = 40 |
| `resume-6mb.pdf` | 6,291,456 bytes | 413 FILE_TOO_LARGE |
| `resume-7-pages.pdf` | 7 pages | 422 TOO_MANY_PAGES |
| `resume-encrypted.pdf` | Password `test` | 422 FILE_ENCRYPTED |
| `resume-scanned.pdf` | 1 page, one image, no text | 422 TEXT_NOT_EXTRACTABLE |
| `resume-too-long.docx` | 35,000 characters | 422 TEXT_TOO_LONG |
| `not-a-resume.png` | PNG renamed to `.pdf` | 415 UNSUPPORTED_FILE_TYPE |
| `zip-bomb.docx` | Expands to > 20 MiB | 415 UNSUPPORTED_FILE_TYPE (zip limits exceeded) |
| `resume-5mb.pdf` | 5,242,000 bytes, 5 pages | 201, used by NFR-002 |
| `resume-hindi.docx` | Devanagari summary + English skills | 201; English skills matched |

---

## 4. Architecture

### 4.1 Component diagram

```mermaid
flowchart LR
  subgraph Browser
    U[User] --> WEB_UI[Next.js pages<br/>CMP-001]
  end
  subgraph AWS ap-south-1
    ALB[ALB<br/>resume.surfingbear.com]
    WEB[ro-web<br/>Next.js server<br/>CMP-001]
    API[ro-api<br/>NestJS HTTP<br/>CMP-002 .. CMP-009, CMP-013, CMP-015]
    WK[ro-worker<br/>NestJS BullMQ<br/>CMP-010 .. CMP-012, CMP-014]
    PG[(PostgreSQL 16<br/>RDS<br/>CMP-016)]
    RD[(Redis 7<br/>ElastiCache<br/>CMP-017)]
    BR[AWS Bedrock<br/>Claude]
    OBS[CloudWatch + AMP<br/>CMP-018]
  end
  PLAT[SurfingBear platform<br/>login + JWKS]

  WEB_UI -->|HTTPS, cookie sb_access| ALB
  ALB -->|/api/*| API
  ALB -->|everything else| WEB
  API -->|verify JWT, JWKS cached| PLAT
  API --> PG
  API -->|enqueue, rate limit| RD
  WK -->|consume jobs, breaker| RD
  WK --> PG
  WK -->|Converse API, IAM| BR
  API -.logs/metrics.-> OBS
  WK -.logs/metrics.-> OBS
  WEB -.logs.-> OBS
  WEB_UI -->|401 → redirect| PLAT
```

The browser only talks to `resume.surfingbear.com`. The ALB routes by path, so web and API share one origin (no CORS, CR-020). The Next.js server renders pages only; it never calls the API on behalf of the user and never sees resume data (all data calls are made from the browser to `/api/v1`).

### 4.2 Components

| ID | Component | Responsibility | Technology |
|---|---|---|---|
| CMP-001 | Web app (`ro-web`) | Screens SCR-01 – SCR-07, client-side file checks, polling, error display, CSP nonce | Next.js 15 App Router, React 19, TanStack Query 5, Tailwind CSS 4, next-intl |
| CMP-002 | API app (`ro-api`) | HTTP entrypoint: global prefix `/api`, versioned routes `/api/v1`, validation pipe, exception filter, request-id, logging, rate limiting, health | NestJS 11, Express adapter, `@nestjs/throttler` with Redis storage |
| CMP-003 | Auth | Read `sb_access`, verify JWT (CR-001 – CR-004), build `RequestUser { userId, plan, roles, isAdmin }`; `AuthGuard` (global), `RolesGuard` (`@Roles('ro_admin')`), `OriginGuard` (CR-021) | `jose` (JWKS + verify) |
| CMP-004 | Consents | Serve notice, record consent, `ConsentService.hasCurrentConsent(userId)` | NestJS module, TBL-003 |
| CMP-005 | Resumes | Upload pipeline (FR-003 order), dedupe, encrypt, list/get/delete | NestJS module, Multer memory storage, Node `crypto` (AES-256-GCM), TBL-004 |
| CMP-006 | Text extractor | PDF/DOCX → normalised text + signals (§3.5.1), with timeout `CFG-19` | `pdfjs-dist` 4 (legacy build), `mammoth`, `yauzl` (zip limits) |
| CMP-007 | Redactor | Apply §3.5.2 rules, compute `hasContact` | Pure TS functions |
| CMP-008 | Analyses (API side) | Create (quota + insert + enqueue), list, get with gating (FR-013), delete | NestJS module, BullMQ `Queue`, TBL-005 – TBL-007, TBL-010 |
| CMP-009 | Quota | Period function, reserve/release (§3.5.6), `getQuota(userId, plan)` | NestJS service, TBL-008 |
| CMP-010 | Analysis processor | Consume `run-analysis` jobs, run the pipeline (§4.3.2), write results, fail + release on error | NestJS standalone app (`worker.ts`), BullMQ `Worker`, concurrency 5 per task |
| CMP-011 | Scoring engine | Section detection, D1/D2/D3/D5, missing skills (§3.5.3, §3.5.4), candidate lines | Pure TS, no I/O |
| CMP-012 | AI client | Build prompt, call Bedrock, retries, circuit breaker, validate output (§4.7) | `@aws-sdk/client-bedrock-runtime` (`ConverseCommand`), `zod` |
| CMP-013 | Report generator | Render the PDF (§4.9) as a stream | `pdfkit` |
| CMP-014 | Privacy & maintenance | Erase all data (FR-020), data summary (FR-023), stuck sweeper (FR-019), retention purge (FR-021) | API module (erase, summary) + worker repeatable jobs on queue `maintenance` |
| CMP-015 | Job roles | Public list, admin CRUD, skill replace, seed | NestJS module, TBL-001, TBL-002 |
| CMP-016 | PostgreSQL | System of record for all data | Amazon RDS PostgreSQL 16, Multi-AZ, `db.t4g.medium` in prod |
| CMP-017 | Redis | BullMQ queues, rate-limit counters, AI breaker state, JWKS is **not** stored here (in-memory) | Amazon ElastiCache Redis 7, 1 primary + 1 replica, `noeviction` policy |
| CMP-018 | Observability | Logs (CloudWatch Logs), metrics (Prometheus format at `/api/metrics` on an internal port, scraped by Amazon Managed Prometheus), traces (OpenTelemetry → AWS X-Ray), alerts | `nestjs-pino`, `prom-client`, `@opentelemetry/sdk-node` |
| CMP-019 | Mock platform | Local/test only: issues JWTs and serves a JWKS (§4.8) | Small Node 22 HTTP server in `tools/mock-platform`, `jose` |

### 4.3 Main flows

#### 4.3.1 Consent and upload (FR-002, FR-003, FR-004, FR-005)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser (CMP-001)
  participant A as ro-api
  participant X as Extractor (CMP-006)
  participant R as Redactor (CMP-007)
  participant DB as PostgreSQL
  B->>A: GET /api/v1/me
  A-->>B: 200 { consent.required: true, quota, plan }
  B->>A: GET /api/v1/consents/notice
  A-->>B: 200 { version, noticeMarkdown }
  B->>A: POST /api/v1/consents { noticeVersion }
  A->>DB: INSERT consents ON CONFLICT DO NOTHING
  A-->>B: 201 consent
  B->>B: check size ≤ CFG-01 and extension .pdf/.docx
  B->>A: POST /api/v1/resumes (multipart, file)
  A->>DB: has consent for CFG-30?
  alt no consent
    A-->>B: 403 CONSENT_REQUIRED
  end
  A->>A: size, magic bytes, resume count < CFG-18
  A->>X: extract(buffer, type) with 10 s timeout
  alt encrypted / too many pages / no text / too long / timeout
    X-->>A: ExtractError(code)
    A-->>B: 422 FILE_ENCRYPTED | TOO_MANY_PAGES | TEXT_NOT_EXTRACTABLE | TEXT_TOO_LONG
  end
  X-->>A: { text, signals, pageCount }
  A->>A: sha256(text); look up (user_id, text_sha256)
  alt duplicate
    A-->>B: 200 { resume, duplicate: true }
  end
  A->>R: redact(text) → { redacted, hasContact }
  A->>A: AES-256-GCM encrypt(redacted); drop buffer + raw text
  A->>DB: INSERT resumes
  A-->>B: 201 { resume, duplicate: false }
```

#### 4.3.2 Start and process an analysis — success (FR-008 – FR-013)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant A as ro-api
  participant DB as PostgreSQL
  participant Q as Redis / BullMQ
  participant W as ro-worker
  participant AI as Bedrock
  B->>A: POST /api/v1/analyses { resumeId, jobRoleId, experienceLevel }
  A->>DB: BEGIN; check resume owner + not expired; role active
  A->>DB: reserve quota (§3.5.6); INSERT analyses status=queued
  A->>DB: COMMIT
  A->>Q: add job run-analysis { analysisId, requestId }, jobId = analysisId
  A-->>B: 202 { id, status: queued }
  loop every 2 s (CFG-20)
    B->>A: GET /api/v1/analyses/{id}
    A-->>B: 200 { status }
  end
  Q->>W: run-analysis
  W->>DB: UPDATE status=processing WHERE status=queued
  W->>DB: load resume (decrypt), role skills for level
  W->>W: sections, D1, D2, D3, D5, missing skills, candidate lines
  W->>Q: breaker closed?
  W->>AI: Converse(prompt with redacted lines), timeout 45 s
  AI-->>W: toolUse submit_analysis { experienceRelevance, suggestions }
  W->>W: zod validate; §3.5.5 validation; rank
  W->>DB: BEGIN; INSERT scores, missing skills, suggestions; UPDATE analyses status=completed WHERE status=processing; COMMIT
  B->>A: GET /api/v1/analyses/{id}
  A->>A: apply gating for current plan (FR-013)
  A-->>B: 200 { status: completed, result }
```

#### 4.3.3 AI timeout, error or invalid output (FR-018)

```mermaid
sequenceDiagram
  autonumber
  participant W as ro-worker
  participant RB as Redis breaker
  participant AI as Bedrock
  participant DB as PostgreSQL
  W->>RB: GET ai:breaker
  alt breaker open (openUntil > now)
    W->>W: job.moveToDelayed(openUntil + jitter 0–5 s); stop (no attempt used)
  end
  loop attempt 1..3 (CFG-13)
    W->>AI: Converse (timeout 45 s)
    alt success + valid schema
      AI-->>W: result
      W->>RB: reset failures
    else timeout / 429 / 5xx / invalid schema
      W->>RB: INCR failures (open for 60 s at 5, CFG-28)
      W->>W: wait 2 s, then 8 s
    end
  end
  Note over W: all 3 attempts failed
  W->>DB: BEGIN; UPDATE analyses SET status=failed, failure_code=AI_UNAVAILABLE|AI_INVALID_OUTPUT WHERE status=processing; release quota; COMMIT
```

#### 4.3.4 Stuck analysis sweep (FR-019) and retention purge (FR-021)

```mermaid
sequenceDiagram
  autonumber
  participant S as ro-worker scheduler (queue maintenance)
  participant DB as PostgreSQL
  Note over S: every 1 min: sweep-stuck
  S->>DB: SELECT id FROM analyses WHERE status IN (queued, processing) AND created_at < now() - 10 min FOR UPDATE SKIP LOCKED
  loop each analysis
    S->>DB: UPDATE status=failed, failure_code=PROCESSING_TIMEOUT; release quota (same tx)
  end
  Note over S: daily 02:00 Asia/Kolkata: purge-retention
  S->>DB: DELETE resumes WHERE expires_at <= now() AND no in-flight analysis (batches of 1,000)
  S->>DB: DELETE analyses WHERE created_at <= now() - 180 days (children cascade)
  S->>DB: DELETE audit_events older than 365 days; usage_counters older than 13 months
  S->>S: log counts
```

#### 4.3.5 Download PDF report (FR-017)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant A as ro-api
  participant DB as PostgreSQL
  participant P as Report generator (CMP-013)
  B->>A: GET /api/v1/analyses/{id}/report.pdf
  A->>DB: load analysis + children (owner check)
  alt not completed
    A-->>B: 409 ANALYSIS_NOT_READY
  end
  A->>A: apply gating for current plan
  A->>P: render(viewModel)
  P-->>B: 200 application/pdf (streamed)
```

#### 4.3.6 Erase all my data (FR-020)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant A as ro-api
  participant DB as PostgreSQL
  participant RD as Redis
  B->>A: DELETE /api/v1/me/data (Origin checked)
  A->>DB: BEGIN
  A->>DB: DELETE analyses WHERE user_id (children cascade)
  A->>DB: DELETE resumes, consents, usage_counters WHERE user_id
  A->>DB: INSERT audit_events (data_erased, user_hash)
  A->>DB: COMMIT
  A->>RD: DEL rl:*:<userId> (SCAN + DEL)
  A-->>B: 204
  Note over A: a queued job for a deleted analysis finds no row and ends without error
```

#### 4.3.7 Analysis state machine

```mermaid
stateDiagram-v2
  [*] --> queued: API-009 (quota reserved)
  queued --> processing: worker picks job
  processing --> completed: results saved
  processing --> failed: AI_UNAVAILABLE / AI_INVALID_OUTPUT
  queued --> failed: PROCESSING_TIMEOUT (sweep) / AI_UNAVAILABLE (enqueue failed)
  processing --> failed: PROCESSING_TIMEOUT (sweep)
  completed --> [*]
  failed --> [*]: quota released
```

Only these transitions are allowed. Every update uses `WHERE status = <expected from-state>`; 0 updated rows means another actor won and the current actor stops.

### 4.4 External integrations

| System | Purpose | Protocol / auth | Failure handling |
|---|---|---|---|
| SurfingBear platform — JWKS | Public keys to verify `sb_access` | HTTPS `GET AUTH_JWKS_URL`, no auth, timeout 3 s | Keys cached in memory `CFG-21`. If a refresh fails, the cached keys are used (no TTL expiry while the platform is unreachable). If there are no cached keys at all (fresh task start) → requests get `503 SERVICE_UNAVAILABLE` (ERR-019) and `/health/ready` reports not ready. |
| SurfingBear platform — login / upgrade pages | Redirect targets | Browser navigation to `PLATFORM_LOGIN_URL`, `PLATFORM_UPGRADE_URL` | None needed (outside RO). |
| AWS Bedrock (Claude) | D4 score + rewrite suggestions | `ConverseCommand` over HTTPS, SigV4 with the ECS task IAM role (`bedrock:InvokeModel` on the one model ARN only), region `ap-south-1` | Timeout `CFG-12`, 3 attempts (FR-018), circuit breaker (§4.7.4), daily cap `CFG-27`. Analyses fail cleanly with quota released. Rest of RO keeps working (NFR-006). |
| PostgreSQL (RDS) | All data | TLS, password auth (Secrets Manager), Prisma pool `connection_limit=10` per task | API: Prisma errors of connection type → `503 SERVICE_UNAVAILABLE`; `/health/ready` fails → ALB stops routing. Worker: job throws, BullMQ retries the job up to 3 times with exponential backoff 5 s (job-level), then the sweep fails it. |
| Redis (ElastiCache) | Queues, rate limits, breaker | TLS, AUTH token | API: if Redis is down, rate limiting **fails open** (requests are allowed, a `warn` log is written); `POST /analyses` returns `503` because the job cannot be queued (FR-008 AC 6). Worker: BullMQ reconnects automatically. |

### 4.5 Tech stack and reasons

| Layer | Choice | Why this choice (for this system) | Alternatives considered |
|---|---|---|---|
| Backend | NestJS 11 on Node.js 22 LTS, TypeScript 5 | My team's main stack. One language end to end means DTO types and constants are shared with the web (`@ro/shared`), so a limit like `CFG-01` can't drift between client and server. Nest modules map 1:1 to the components above. | Spring Boot (stronger PDF tooling but a second language for the frontend team), Go (fast, but no shared types with Next.js). |
| API + worker split | One codebase, two entrypoints (`main.ts`, `worker.ts`), same image | AI calls take up to 45 s; keeping them off the HTTP tasks keeps API p95 < 300 ms (NFR-001) and lets the worker scale on queue depth (NFR-007) separately. One codebase avoids duplicating the Prisma schema. | Separate worker repo (duplicate models); doing AI calls in the request (blocks connections, no retry). |
| Queue | BullMQ 5 on Redis | Redis is already required; BullMQ gives delayed jobs (breaker), repeatable jobs with timezone (purge at 02:00 IST), stalled-job recovery, and `jobId` dedupe (one job per analysis). | SQS (another service, no repeatable cron), pg-boss (fine, but Redis already present). |
| Frontend | Next.js 15 App Router, React 19 | Required by the brief. Server components render the shell fast (NFR-018); data screens are client components because they poll. `output: "standalone"` gives a small Docker image. | — |
| Client data | TanStack Query 5 | Built-in `refetchInterval` with a function (stop on final status) and `refetchIntervalInBackground: false` gives exactly FR-014's polling rules. | SWR (similar, less control of mutations); WebSockets/SSE (needs sticky connections through ALB for a < 1 min wait — not worth it). |
| Styling | Tailwind CSS 4 + Radix UI primitives | Radix gives accessible dialogs, radio groups, select (NFR-016) without a heavy design system. | MUI (large bundle), PrimeReact (theme overrides needed). |
| ORM | Prisma 6 with hand-edited SQL migrations | Typed client shared by API and worker; migrations are plain SQL so CHECK constraints and partial unique indexes (needed by FR-008) are written directly. | TypeORM (weaker typing), raw `pg` (more boilerplate). |
| Database | PostgreSQL 16 | Given. Row locks (`FOR UPDATE`) make quota exact under concurrency (FR-009 AC 6); partial unique index enforces one in-flight analysis; `jsonb` for signals. | — |
| Cache | Redis 7 | Given. Rate-limit counters with TTL, BullMQ, breaker state shared by all workers. | — |
| AI | Claude via AWS Bedrock, `ap-south-1`, model from `BEDROCK_MODEL_ID` | Data stays in India in our AWS account (DPDP, NFR-011); no prompt retention for training; IAM role auth (no API key to leak); tool-use forces structured JSON output (§4.7). | Direct provider API (data leaves India; key management); self-hosted open model (GPU cost, quality). |
| File storage | **None** | The file is parsed in memory and discarded (A-04): nothing to secure, expire or erase. | S3 with 30-day lifecycle (adds breach surface and an erase path). |
| PDF extraction | `pdfjs-dist` | Maintained by Mozilla, gives text positions (`hasEOL`) and operator lists for image counts (D3). | `pdf-parse` (unmaintained wrapper over old pdf.js). |
| DOCX extraction | `mammoth` + `yauzl` | `mammoth` gives clean raw text; `yauzl` lets us enforce zip-bomb limits before parsing. | `docx4js` (less used). |
| PDF output | `pdfkit` | Streams a PDF directly with no browser; small image; fast (NFR-003). Report layout is simple text + bars. | Headless Chromium (large image, slow cold start), `@react-pdf/renderer` (heavier for this layout). |
| Hosting | AWS ECS Fargate, ALB, RDS, ElastiCache, `ap-south-1` | Mumbai region for residency; Fargate needs no server patching; ALB path routing gives the single origin. | EKS (too much ops for 3 services). |
| IaC / CI | Terraform 1.9, GitHub Actions | Same infra in 3 accounts (CR-035); repo is on GitHub. | CDK. |

### 4.6 Configuration (environment variables)

Validated at start-up with a `zod` schema in `apps/api/src/config/env.schema.ts`; an invalid or missing required variable stops the process with exit code 1 and a log line naming the variable (never its value).

| Variable | Used by | Required | Example / default | Notes |
|---|---|---|---|---|
| `NODE_ENV` | all | yes | `production` | |
| `APP_ENV` | all | yes | `local` \| `test` \| `dev` \| `staging` \| `prod` | Drives CR-001 header rule, CR-013 log level |
| `PORT` | api | no | `4000` | web uses `3000` |
| `METRICS_PORT` | api, worker | no | `9464` | Not exposed through the ALB |
| `DATABASE_URL` | api, worker | yes | `postgresql://ro:***@host:5432/ro?sslmode=require&connection_limit=10` | |
| `REDIS_URL` | api, worker | yes | `rediss://:***@host:6379` | |
| `AUTH_JWKS_URL` | api | yes | `https://auth.surfingbear.com/.well-known/jwks.json` | |
| `AUTH_ISSUER` | api | yes | `https://auth.surfingbear.com` | |
| `AUTH_AUDIENCE` | api | yes | `resume-optimizer` | |
| `WEB_ORIGIN` | api | yes | `https://resume.surfingbear.com` | CR-021 |
| `PLATFORM_LOGIN_URL` | web | yes | `https://surfingbear.com/login` | exposed as `NEXT_PUBLIC_PLATFORM_LOGIN_URL` |
| `PLATFORM_UPGRADE_URL` | web | yes | `https://surfingbear.com/pricing` | exposed as `NEXT_PUBLIC_PLATFORM_UPGRADE_URL` |
| `RESUME_TEXT_KEY` | api, worker | yes | 32 random bytes, base64 | AES-256-GCM key (TBL-004) |
| `RESUME_TEXT_KEY_ID` | api, worker | yes | `k1` | Stored with each row; a rotation adds `RESUME_TEXT_KEY_PREVIOUS` + `_ID` for reading old rows |
| `AI_MODE` | worker | yes | `bedrock` \| `mock` | `mock` only allowed when `APP_ENV` is `local` or `test` (start-up check) |
| `AI_MOCK_FAILURE` | worker | no | `none` \| `timeout` \| `error` \| `invalid` | Only in `mock` mode (§4.7.5) |
| `AWS_REGION` | worker | yes | `ap-south-1` | Start-up check: must be `ap-south-1` when `APP_ENV=prod` |
| `BEDROCK_MODEL_ID` | worker | yes in `bedrock` mode | the Claude Sonnet model ID / inference profile ID enabled in the account | Recorded per analysis (`ai_model`) |
| `AI_TIMEOUT_MS` | worker | no | `45000` | CFG-12 |
| `AI_DAILY_CALL_CAP` | worker | no | `0` (unlimited) | CFG-27; `200` in `dev` |
| `FREE_MONTHLY_ANALYSES` | api | no | `3` | CFG-05 |
| `PRO_MONTHLY_ANALYSES` | api | no | `50` | CFG-06 |
| `WORKER_CONCURRENCY` | worker | no | `5` | |
| `LOG_LEVEL` | all | no | per CR-013 | |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | api, worker | no | `http://localhost:4318` | ADOT sidecar |

`.env.example` in each app lists every variable with the `local` value.

### 4.7 AI call contract (CMP-012)

#### 4.7.1 Request

`ConverseCommand` with:

- `modelId`: `BEDROCK_MODEL_ID`
- `inferenceConfig`: `{ maxTokens: 4000, temperature: 0 }`
- `system`: the system prompt below (`PROMPT_VERSION = "p1"`, stored in `apps/api/src/worker/ai/prompts/p1.ts`)
- `messages`: one user message whose text is the JSON input below
- `toolConfig`: one tool `submit_analysis` with the input schema in §4.7.2, and `toolChoice: { tool: { name: "submit_analysis" } }` so the model must answer through the tool

System prompt (`p1`):

```text
You are an expert technical recruiter reviewing a resume for an Applicant Tracking System (ATS) report.
Personal details in the resume have been replaced with tokens such as [NAME], [EMAIL], [PHONE], [URL], [ID]. Never try to guess them and never output them.

You will receive JSON with: targetRole (name and description), experienceLevel (entry = 0-2 years, mid = 3-5, senior = 6-9, lead = 10+), resumeText, and candidateLines (each {i, t}).

Do two things and answer only by calling the tool submit_analysis:

1. experienceRelevance: score 0-100 for how well the work experience in resumeText matches the targetRole at the experienceLevel. 90-100 = directly relevant roles and responsibilities at the right seniority; 60-89 = mostly relevant or seniority slightly off; 30-59 = partly relevant or clearly different seniority; 0-29 = unrelated. Give a one-sentence explanation (max 300 characters) that does not quote personal details.

2. suggestions: pick up to 10 of the weakest candidateLines and rewrite each one. For each give: lineIndex (the i value), issue (one of weak_verb, no_metric, vague, too_long, passive, jargon), severity (3 = hurts the resume most, 1 = minor), rewrite (10-300 characters, starts with a strong past-tense action verb, keeps every fact of the original), rationale (max 200 characters).
Rules for rewrite: do not invent facts, employers, tools or numbers. If a number would make the line stronger but the original has none, write a placeholder exactly as [number], [percent], [amount] or [count] for the candidate to fill in. Only use digits that already appear in the original line.
```

User message JSON:

```json
{
  "targetRole": { "name": "Frontend Developer", "description": "Builds user interfaces for web applications." },
  "experienceLevel": "mid",
  "resumeText": "<redacted resume text, full>",
  "candidateLines": [ { "i": 12, "t": "Worked on the dashboard using React." } ]
}
```

#### 4.7.2 Output schema (tool input), validated with `zod`

```json
{
  "type": "object",
  "required": ["experienceRelevance", "suggestions"],
  "additionalProperties": false,
  "properties": {
    "experienceRelevance": {
      "type": "object",
      "required": ["score", "explanation"],
      "additionalProperties": false,
      "properties": {
        "score": { "type": "integer", "minimum": 0, "maximum": 100 },
        "explanation": { "type": "string", "minLength": 1, "maxLength": 300 }
      }
    },
    "suggestions": {
      "type": "array",
      "maxItems": 10,
      "items": {
        "type": "object",
        "required": ["lineIndex", "issue", "severity", "rewrite", "rationale"],
        "additionalProperties": false,
        "properties": {
          "lineIndex": { "type": "integer", "minimum": 0 },
          "issue": { "enum": ["weak_verb", "no_metric", "vague", "too_long", "passive", "jargon"] },
          "severity": { "type": "integer", "minimum": 1, "maximum": 3 },
          "rewrite": { "type": "string", "maxLength": 400 },
          "rationale": { "type": "string", "maxLength": 300 }
        }
      }
    }
  }
}
```

Validation is two-level:
1. **Schema level** (zod, the schema above): the response has no `toolUse` block, or the tool input fails the schema, or `experienceRelevance` fails → the **attempt** failed with reason `invalid` (counts toward `CFG-13`).
2. **Item level** (§3.5.5 rules 1–6, which are stricter than the schema, e.g. rewrite 10–300 chars, rationale truncated to 200 chars): failing items are dropped silently; the attempt still succeeds. Metric `ro_ai_suggestions_dropped_total{reason}` counts drops.

The worker stores `ai_model` (= `BEDROCK_MODEL_ID`), `prompt_version` (= `p1`) and the token counts from `usage` on the analysis. The prompt and the response body are **not** stored or logged (CR-011).

#### 4.7.3 Error classification

| Outcome | Classified as | Retry? |
|---|---|---|
| No response within `CFG-12` (SDK `abortSignal` via `AbortSignal.timeout`) | `timeout` | yes |
| `ThrottlingException`, `ServiceUnavailableException`, `InternalServerException`, `ModelNotReadyException`, network error | `error` | yes |
| `ValidationException`, `AccessDeniedException`, `ResourceNotFoundException` | `config` | no — fail immediately with `AI_UNAVAILABLE`, log at `error`, alert |
| Schema failure (§4.7.2 level 1) | `invalid` | yes |

Final failure code: if the last attempt was `invalid` → `AI_INVALID_OUTPUT` (ERR-023); otherwise → `AI_UNAVAILABLE` (ERR-022).

#### 4.7.4 Circuit breaker and daily cap

- Redis hash `ai:breaker` with fields `failures` (int) and `openUntil` (epoch ms).
- After a call classified `timeout`/`error`: `HINCRBY failures 1`; if the result ≥ 5 → `HSET openUntil now+60000`. After a successful call: `HSET failures 0 openUntil 0`. `invalid` does not count (the provider is up).
- Before each attempt: if `openUntil > now`, the job is moved to delayed until `openUntil` + random 0–5 s (`job.moveToDelayed`, then throw BullMQ `DelayedError`); no attempt is used. When `openUntil` has passed, the first worker to do `SET ai:breaker:trial 1 NX PX 60000` makes one trial call; other workers wait as if the breaker were open.
- Daily cap (`CFG-27`, non-zero only in `dev`): Redis key `ai:calls:<YYYY-MM-DD IST>` `INCR` with 48 h TTL; when over the cap, the analysis fails with `AI_UNAVAILABLE` without calling Bedrock.

#### 4.7.5 Mock AI (`AI_MODE=mock`)

Deterministic, no network:
- `experienceRelevance = { score: 70, explanation: "Mock: experience is mostly relevant to the target role." }`.
- `suggestions`: for the first 3 candidate lines (in order) that contain no digit: `{ lineIndex, issue: "no_metric", severity: 2, rewrite: "<line without trailing period>, improving results by [percent].", rationale: "Mock: adding a measurable result makes the impact clear." }`.
- `AI_MOCK_FAILURE=timeout` → every call waits `AI_TIMEOUT_MS` then is classified `timeout`; `error` → throws `ServiceUnavailableException`; `invalid` → returns `{ "foo": 1 }`.

### 4.8 Local setup and mock platform

```bash
corepack enable && pnpm install
docker compose up -d                 # postgres:16 on 5432, redis:7 on 6379, mock-platform on 4010
pnpm --filter @ro/api prisma migrate deploy && pnpm --filter @ro/api prisma db seed
pnpm dev                             # web :3000, api :4000, worker (AI_MODE=mock)
open http://localhost:4010/login?userId=11111111-1111-4111-8111-111111111111&plan=free
```

Mock platform (CMP-019, `tools/mock-platform/server.ts`):
- On start, generates an RS256 key pair (`kid` = `mock-1`) unless `MOCK_KEY_PATH` points to a saved JWK.
- `GET /.well-known/jwks.json` → public JWKS.
- `GET /login?userId=<uuid>&plan=free|pro&admin=0|1&returnTo=<url>` → signs a JWT (`iss=http://localhost:4010`, `aud=resume-optimizer`, `exp` = now + 15 min, `roles` = `["ro_admin"]` if `admin=1`), sets cookie `sb_access` (`HttpOnly; SameSite=Lax; Path=/`, domain `localhost`), and redirects to `returnTo` (default `http://localhost:3000/`).
- `POST /token` with JSON `{ userId, plan, admin, expiresInSec }` → `{ token }` for e2e tests (sent as `Authorization: Bearer`, allowed in `local`/`test` per CR-001).
- Local `.env`: `AUTH_JWKS_URL=http://localhost:4010/.well-known/jwks.json`, `AUTH_ISSUER=http://localhost:4010`, `WEB_ORIGIN=http://localhost:3000`, `PLATFORM_LOGIN_URL=http://localhost:4010/login`.
- Next.js `next.config.ts` (local only, `APP_ENV=local`): `rewrites: [{ source: '/api/:path*', destination: 'http://localhost:4000/api/:path*' }]` so the browser sees one origin, as in production.

### 4.9 PDF report layout (CMP-013, FR-017)

A4 portrait, margins 48 pt, font Helvetica (built into pdfkit; no font files), black text on white, accent colour `#1E5EFF` for bars.

1. **Header:** "SurfingBear Resume Optimizer — ATS Report" (18 pt bold); line below: "{roleName} · {level label} · {completedAt as `20 Oct 2026, 2:45 pm IST`}" (10 pt); resume display name (10 pt, grey `#555555`).
2. **Overall score:** large number "{overallScore}/100" (36 pt bold) and band label.
3. **Dimensions table:** one row per D1–D5 in that order: label (`Keyword match`, `Section structure`, `ATS parseability`, `Experience relevance`, `Impact`), weight "30 %", score, a horizontal bar 200 pt wide filled proportionally, and the explanation below in 9 pt.
4. **Missing skills:** two lists "Must-have" and "Nice-to-have"; "None — great coverage!" when empty.
5. **Suggestions:** numbered by `rank`; each shows "Original:" line, "Suggested:" line (placeholders in bold), issue label, rationale. Free users: first 3 only, then the sentence from FR-017 AC 6. "No weak lines found." when empty.
6. **Footer on every page:** "Generated {now IST} · Page {n} of {N} · Scores are guidance, not a hiring decision."
7. The PDF metadata `Title` = "ATS Report — {roleName}"; `Author` = "SurfingBear"; no personal data in metadata.

### 4.10 Observability (CMP-018)

**Metrics** (Prometheus, prefix `ro_`):

| Metric | Type | Labels |
|---|---|---|
| `ro_http_request_duration_seconds` | histogram | `route`, `method`, `status` |
| `ro_upload_rejected_total` | counter | `code` |
| `ro_analysis_created_total` | counter | `plan` |
| `ro_analysis_finished_total` | counter | `status`, `failure_code` |
| `ro_analysis_duration_seconds` | histogram (buckets 5, 10, 20, 30, 45, 60, 120, 300) | — |
| `ro_ai_call_duration_seconds` | histogram | `outcome` (`ok`, `timeout`, `error`, `invalid`, `config`) |
| `ro_ai_tokens_total` | counter | `direction` (`input`, `output`) |
| `ro_ai_suggestions_dropped_total` | counter | `reason` (rule number 1–6) |
| `ro_ai_breaker_open` | gauge (0/1) | — |
| `ro_queue_waiting_jobs` | gauge | `queue` |
| `ro_purge_deleted_rows_total` | counter | `table` |

**Alerts** (to Slack `#ro-alerts`, prod and staging):

| Alert | Condition |
|---|---|
| API error rate | 5xx > 2 % of requests over 5 min |
| API latency | p95 of `ro_http_request_duration_seconds` (excluding API-004, API-013) > 300 ms for 10 min |
| Analyses failing | `failed` > 10 % of finished analyses over 15 min |
| Breaker open | `ro_ai_breaker_open = 1` for > 5 min |
| Stuck analyses | any in-flight analysis older than 10 min (NFR-013) |
| Queue backlog | `ro_queue_waiting_jobs{queue="analysis"} > 100` for 5 min |
| Purge did not run | no `purge-retention` success log in the last 26 h |
| AI config error | any AI call with outcome `config` |

**Traces:** OpenTelemetry auto-instrumentation for HTTP, Prisma and ioredis; the API injects the trace context into the BullMQ job data so the worker span links to the request span. Span attributes MUST NOT contain resume text (CR-011).

### 4.11 Queue and worker rules (CMP-010, CMP-014)

| Queue | Job name | Added by | Job options | Payload |
|---|---|---|---|---|
| `analysis` | `run-analysis` | API-009 | `jobId = analysisId` (dedupe), `attempts: 3`, `backoff: { type: "exponential", delay: 5000 }`, `removeOnComplete: { age: 86400 }`, `removeOnFail: { age: 604800 }` | `{ analysisId: string, requestId: string, traceContext: object }` — no personal data |
| `maintenance` | `sweep-stuck` | worker start-up (`upsertJobScheduler`) | every 60,000 ms | `{}` |
| `maintenance` | `purge-retention` | worker start-up | cron `0 2 * * *`, `tz: "Asia/Kolkata"` | `{}` |

`run-analysis` processor steps (each step's failure behaviour in brackets):

1. Load the analysis by ID. Not found → return (erased, FR-020 AC 3). Status `completed`/`failed` → return.
2. Claim: `UPDATE analyses SET status='processing', started_at=COALESCE(started_at, now()) WHERE id=$1 AND status IN ('queued','processing')`. 0 rows → return. (`processing` is allowed so a BullMQ retry or stalled-job recovery of the **same** job can continue; BullMQ's job lock guarantees one active run per `jobId`.)
3. Load the resume. Not found (deleted by erasure) → return.
4. Decrypt the text, split lines (§3.5.1 step 4), run CMP-011 (D1, D2, D3, D5, missing skills, candidate lines).
5. Call CMP-012 (§4.7) with its own 3 attempts. AI failure → step 7.
6. Save in one transaction: delete any existing child rows for this analysis (idempotent retry), insert TBL-006/TBL-007/TBL-010 rows, `UPDATE analyses SET status='completed', overall_score, band, completed_at=now(), ai_model, prompt_version, ai_input_tokens, ai_output_tokens WHERE id=$1 AND status='processing'`. 0 rows → rollback, return (the sweep already failed it, FR-019 AC 2).
7. Fail: in one transaction `UPDATE analyses SET status='failed', failure_code=$2, completed_at=now() WHERE id=$1 AND status='processing'` and, if 1 row changed, release quota (§3.5.6).
8. Any other exception (DB, Redis, decrypt) is thrown, so BullMQ retries the job (max 3 job attempts). After the last job attempt, the processor's `failed` event handler runs step 7 with `AI_UNAVAILABLE`; if that also fails, the stuck sweep (FR-019) fails it within 10 min.

Worker graceful shutdown: on `SIGTERM`, `worker.close()` waits up to 60 s for active jobs (ECS `stopTimeout: 90`).

---

## 5. ERD and database design

_Written in a later revision._

---

## 6. API design

_Written in a later revision._

---

## 7. Traceability matrix

_Written in a later revision._

---

## 8. Change log

| Version | Date | Author | What changed | Why | Affected IDs |
|---|---|---|---|---|---|
| v0.1 | 2026-10-20 | Vaishali Naruka | Sections 1–2: common requirements and coding conventions | First draft | CR-001 – CR-041 |
| v0.2 | 2026-10-20 | Vaishali Naruka | Section 3: glossary, FR-001 – FR-024, NFR-001 – NFR-020, assumptions A-01 – A-11, constants CFG-01 – CFG-32, scoring and redaction rules, privacy notice, screens SCR-01 – SCR-07, test fixtures | First draft | FR-001 – FR-024, NFR-001 – NFR-020 |
| v0.3 | 2026-10-20 | Vaishali Naruka | Section 4: components, flows, integrations, stack reasons, env config, AI contract, PDF layout, observability, local setup | First draft | CMP-001 – CMP-019 |
