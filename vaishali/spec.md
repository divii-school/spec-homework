# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Vaishali Naruka |
| Version | v0.1 |
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

_Section 3 is written in the next revision._

---

## 4. Architecture

_Written in a later revision._

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
