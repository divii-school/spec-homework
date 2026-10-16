# SurfingBear Resume Optimizer — Specification

| Field | Value |
|---|---|
| Author | Pranesh Ghosh |
| Version | v1.0 |
| Last updated | 2026-10-16 |
| Status | Draft |
| Backend stack | Java 21 + Spring Boot 3.5 |
| Frontend stack | Next.js 15 (App Router, TypeScript) |
| Data stores | PostgreSQL 16, Redis 7 |

## 0. How to read this document

- This file is the **only** input needed to build the system. When this document and any other source disagree, this document wins.
- **MUST / MUST NOT** are hard rules. **SHOULD** is a strong default that may be broken only with a reason written in the PR. Anything not marked is descriptive.
- IDs are permanent and never reused: `CR-` common requirement, `FR-` functional, `NFR-` non-functional, `CMP-` component, `TBL-` table, `API-` endpoint, `ERR-` error code, `A-` assumption, `OOS-` out of scope.
- Every number that appears in more than one section is defined **once**, in [§3.4 Business constants](#34-business-constants), and referenced by name elsewhere. If a value changes, it changes there.
- Glossary:

| Term | Meaning |
|---|---|
| Platform | The main SurfingBear web platform. It owns sign-up, login, sessions and the user's subscription plan. |
| Module | This system, the Resume Optimizer ("RO"). |
| User | A person logged in to the Platform who uses the Module. |
| Resume | One uploaded PDF or DOCX file, stored by the Module only as redacted text plus metadata (the file itself is never stored, see CR-025). |
| Analysis | One AI evaluation of one Resume against one job role and one experience level. |
| Job role | An entry in the Module's job-role catalogue (e.g. "Backend Developer"), with a list of skills. |
| Suggestion | One proposed rewrite of one weak line of the Resume. |
| Plan | `FREE` or `PREMIUM`, taken from the Platform token on every request. |
| Quota period | One calendar month in the `Asia/Kolkata` time zone, written `YYYY-MM`. |
| DPDP Act | India's Digital Personal Data Protection Act, 2023, and the DPDP Rules, 2025. |

---

## 1. Company-wide common requirements

Rules every Divii / Hanriver project must follow, written for this Module. Each rule has an ID and a way to check it.

### 1.1 Authentication

The Module does **not** sign users up, log them in, or issue tokens. It only verifies the token the Platform has already issued.

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-001 | Authentication | Every path under `/api/ro/v1/**` requires a valid Platform JWT. The only exceptions are `POST /api/ro/v1/dev/token` (exists only under the `local` profile, see API-017) and `GET /actuator/health/**`. A request without a valid token gets `401 UNAUTHENTICATED` (ERR-001). | Integration test `SecurityCoverageIT` calls every mapped endpoint with no cookie and asserts 401 + `UNAUTHENTICATED`. |
| CR-002 | Authentication | The token is read **only** from the httpOnly cookie named `sb_at`. An `Authorization` header is ignored. The Module never sets, refreshes or deletes `sb_at` (except API-017 under `local`). | Test: a valid token sent only as `Authorization: Bearer …` gets 401. Code review: no `Set-Cookie` outside `DevTokenController`. |
| CR-003 | Authentication | A token is valid only if **all** hold: signature is RS256 and verifies against the Platform JWKS (`ro.auth.jwks-uri`, keys cached 10 minutes, refetched once on an unknown `kid`); `iss` equals `ro.auth.issuer`; `aud` contains `resume-optimizer`; `exp` is later than now minus 30 s clock skew; `sub` is a UUID string. Any failure → 401 `UNAUTHENTICATED`. | Unit tests `JwtValidationTest`: one test per rule (bad signature, wrong `iss`, wrong `aud`, expired, non-UUID `sub`). |
| CR-004 | Authentication | When any API call returns 401, the frontend redirects the browser to `https://app.surfingbear.com/login?returnTo=<URL-encoded current URL>` and shows nothing else. | Playwright test with an expired cookie asserts the redirect URL. |
| CR-005 | Authentication | CSRF: every `POST`, `PUT`, `PATCH` and `DELETE` request must carry the header `X-Requested-By: ro-web`. Without it → `403 CSRF_HEADER_MISSING` (ERR-003), checked **after** authentication. The backend sends **no** CORS headers for any origin (frontend and API share one origin, see §4.1), so a cross-site page cannot add the header. | Integration test: authenticated `POST /api/ro/v1/analyses` without the header → 403 `CSRF_HEADER_MISSING`; an `OPTIONS` preflight from `https://evil.example` gets no `Access-Control-Allow-Origin`. |

The Platform token claims the Module relies on (assumption A-01):

| Claim | Type | Example | Used for |
|---|---|---|---|
| `sub` | string (UUID) | `"7d0c2f1e-5b8a-4c39-9a51-0f1f2c3d4e5f"` | `user_id` in every table. |
| `iss` | string | `"https://auth.surfingbear.com"` | CR-003. |
| `aud` | string or string[] | `["platform","resume-optimizer"]` | CR-003. |
| `exp` | number (epoch s) | `1792130000` | CR-003. |
| `sb_plan` | string | `"FREE"` or `"PREMIUM"` | Plan, CR-006. |
| `roles` | string[] (optional) | `["RO_ADMIN"]` | Admin role, CR-006. |

Other claims (name, email, phone) are **ignored and never read**, so they can never be stored or logged.

### 1.2 Roles and permissions

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-006 | Roles | Roles are derived from the token on every request, never stored: `sb_plan == "PREMIUM"` → role `PREMIUM`; any other value or a missing claim → role `FREE`. If `roles` contains `"RO_ADMIN"`, the user also has role `ADMIN`. An admin is still `FREE` or `PREMIUM` for their own analyses. | Unit test `RoleMapperTest` covers `PREMIUM`, `FREE`, `"premium"` (lower case → `FREE`), missing claim, and `RO_ADMIN`. |
| CR-007 | Ownership | A user can read, download or delete only rows whose `user_id` equals their own `sub`. A request for someone else's resume or analysis returns `404 RESOURCE_NOT_FOUND` (ERR-006), exactly like a missing id, so the existence of other users' data is never revealed. `ADMIN` gives **no** access to other users' resumes or analyses. | Integration test `OwnershipIT`: user B requests user A's resume, analysis and report → 404 each; an ADMIN token gets 404 too. |
| CR-008 | Roles | Admin endpoints (`/api/ro/v1/admin/**`) require role `ADMIN`. Without it → `403 FORBIDDEN` (ERR-002). | Integration test: FREE token on API-014 → 403 `FORBIDDEN`. |

Role × permission table (✔ allowed, ✖ refused with the code in brackets):

| Permission | FREE | PREMIUM | ADMIN (in addition to own plan) |
|---|---|---|---|
| Read own profile, usage and consent (API-001) | ✔ | ✔ | ✔ |
| Accept privacy notice (API-002) | ✔ | ✔ | ✔ |
| List active job roles (API-003) | ✔ | ✔ | ✔ |
| Upload / list / delete own resumes (API-004…006) | ✔ | ✔ | ✔ |
| Create analysis (API-007) | ✔ up to `FREE_MONTHLY_ANALYSES` per quota period | ✔ up to `PREMIUM_MONTHLY_ANALYSES` | (own plan's limit) |
| See all suggestions of an analysis (API-009) | ✖ only the first `FREE_VISIBLE_SUGGESTIONS`; the rest are counted as locked | ✔ | (own plan) |
| Download PDF report (API-010) | ✔ (contains only visible suggestions) | ✔ (all suggestions) | (own plan) |
| Export / erase own data (API-012, API-013) | ✔ | ✔ | ✔ |
| Read anyone else's resumes or analyses | ✖ (404) | ✖ (404) | ✖ (404) |
| Manage job-role catalogue (API-014…016) | ✖ (403 `FORBIDDEN`) | ✖ (403 `FORBIDDEN`) | ✔ |

### 1.3 Logging

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-009 | Logging | All backend logs are JSON, one object per line, written to stdout with `logstash-logback-encoder`. Every line has `timestamp` (ISO-8601 UTC, ms), `level`, `logger`, `thread`, `message`, `requestId`, and `userId` when the request is authenticated. Worker lines carry `analysisId` instead of `requestId`. | Test `LogFormatTest` captures one request's log lines and asserts each parses as JSON with the fields. |
| CR-010 | Logging | The following are **never** logged at any level: file bytes, extracted or redacted resume text, original filenames, AI prompts and AI responses, suggestion or rationale text, cookie and JWT values, the Anthropic API key, and any token claim other than `sub`. | Test `NoPersonalDataInLogsIT` uploads a resume whose text and filename contain the marker `CANARY-7Q2X`, runs a full analysis with the stub AI, and asserts no captured log line contains `CANARY-7Q2X`. |
| CR-011 | Logging | `requestId`: taken from the `X-Request-Id` request header when it is a valid UUID, otherwise a new random UUID. It is put in the SLF4J MDC, returned in the `X-Request-Id` response header, and included in every error body (CR-013). | Integration test sends `X-Request-Id: <uuid>` and asserts the same value in the response header and in a 404 error body. |
| CR-012 | Logging | Levels: `ERROR` = a person must act (5xx, AI configuration error, retention job failure). `WARN` = degraded but handled (AI retry, Redis unavailable, dropped AI suggestion). `INFO` = one line per HTTP request (`method`, `pathTemplate`, `status`, `durationMs`) and one line per analysis state change. `DEBUG` is off in `staging` and `prod`. | Code review checklist item; `application-prod.yml` sets `logging.level.root=INFO`. |
| CR-040 | Audit | These actions write one row to `audit_events` (TBL-011) in the same transaction as the action: consent accepted, consent withdrawn, data exported, data erased, job role created, job role updated. The row never contains personal data (only ids, counts and enum values). | Integration tests per action assert exactly one audit row with the expected `action`. |

### 1.4 Error handling

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-013 | Errors | Every non-2xx response from `/api/ro/v1/**` uses exactly the format below, with `Content-Type: application/json`. This includes unknown paths (404), wrong methods (405), oversized uploads (413), validation errors and 500s. No other error shape exists. | Contract test `ErrorFormatIT` triggers one error of each HTTP status listed in §6.3 and validates the body against the JSON schema below. |
| CR-014 | Errors | `error.code` is always one of the codes in §6.3. `error.message` is the English text from the same table. Bodies never contain stack traces, SQL, class names or exception messages. 5xx bodies always use the fixed generic message. | `ErrorFormatIT` asserts `code` ∈ the §6.3 list; code review. |
| CR-015 | Errors | Every exception not mapped to a §6.3 code becomes `500 INTERNAL_ERROR` (ERR-029) and is logged once at `ERROR` with the stack trace and `requestId`. A database or Redis-for-required-path outage becomes `503 SERVICE_UNAVAILABLE` (ERR-030). Spring's own exceptions are mapped explicitly (unknown path → `ENDPOINT_NOT_FOUND`, wrong method → `METHOD_NOT_ALLOWED` with an `Allow` header, unreadable JSON → `MALFORMED_REQUEST`, missing parameter / type mismatch → `VALIDATION_FAILED`, upload too large → `FILE_TOO_LARGE`), so a catch-all handler can never turn them into 500. | `ErrorFormatIT` covers each Spring exception listed. |

**The one standard error format** (identical to §6.1):

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "The file is larger than 5 MB.",
    "requestId": "3f6d9c1e-2a47-4b8e-9d0f-6c1b2a3e4f50",
    "timestamp": "2026-10-16T12:03:44.120Z",
    "details": { "maxBytes": 5242880 },
    "fieldErrors": []
  }
}
```

| Field | Type | Always present | Meaning |
|---|---|---|---|
| `error.code` | string | yes | Stable machine code from §6.3. Clients branch on this. |
| `error.message` | string | yes | English, human-readable, from §6.3. |
| `error.requestId` | string (UUID) | yes | Same as the `X-Request-Id` response header. |
| `error.timestamp` | string (ISO-8601 UTC) | yes | Server time of the error. |
| `error.details` | object | yes (may be `{}`) | Code-specific extra data; the keys per code are listed in §6.3. |
| `error.fieldErrors` | array | yes (may be `[]`) | Only for `VALIDATION_FAILED`: `[{ "field": "jobRoleId", "code": "REQUIRED", "message": "must not be null" }]`. `field` is the JSON property, query parameter or header name. `code` is one of `REQUIRED`, `INVALID_FORMAT`, `OUT_OF_RANGE`, `TOO_LONG`, `TOO_SHORT`, `INVALID_VALUE`. The rejected value is never echoed. |

### 1.5 Security

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-016 | Secrets | Secrets (DB password, Redis auth token, Anthropic API key) exist only as environment variables injected by ECS from AWS Secrets Manager. No secret is in the repository, in `application*.yml` (they reference `${RO_DB_PASSWORD}` etc.), in Docker images, or in logs. | CI runs `gitleaks detect` on every PR and fails on a finding; code review. |
| CR-017 | Transport | Public traffic is HTTPS only (TLS 1.2 or higher) terminated at the AWS ALB; HTTP is redirected to HTTPS. Responses carry `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Backend → PostgreSQL uses TLS (`sslmode=verify-full`); backend → Redis uses TLS. | `curl -I http://…` returns 301; staging config review; `testssl.sh` scan before go-live shows no TLS 1.0/1.1. |
| CR-018 | Input validation | Every request body, path variable, query parameter and header used by a controller is validated with Jakarta Bean Validation before the service runs. JSON with unknown properties or wrong types is rejected with `400 MALFORMED_REQUEST` (ERR-005) (`spring.jackson.deserialization.fail-on-unknown-properties=true`). | Integration test posts `{"jobRoleId":"…","extra":1}` → 400 `MALFORMED_REQUEST`. |
| CR-019 | File safety | Uploaded files are identified by their **content** (magic bytes and container structure), never by filename extension or the `Content-Type` part header. Parsing runs with the limits in §3.4 (`MAX_FILE_BYTES`, `MAX_PDF_PAGES`, `PARSE_TIMEOUT`) and with Apache POI zip-bomb protection (`ZipSecureFile.setMinInflateRatio(0.01)`, `ZipSecureFile.setMaxEntrySize(20 MB)`). The file is processed in memory and discarded when the request ends. | Tests: a `.pdf`-named DOCX is accepted as DOCX; a `.docx`-named PNG gets `UNSUPPORTED_FILE_TYPE`; a zip bomb DOCX gets `FILE_CORRUPT` within `PARSE_TIMEOUT`. |
| CR-020 | Rate limiting | Per-user fixed-window limits in Redis (table below). Over the limit → `429 RATE_LIMITED` (ERR-027) with header `Retry-After: <seconds until the window ends>`. If Redis is unreachable the request is **allowed** (fail-open), a `WARN` is logged at most once per minute, and metric `ro_ratelimit_redis_errors_total` is incremented. | Integration test (Testcontainers Redis) sends 11 uploads in an hour → the 11th is 429 with `Retry-After`; test with Redis stopped → request succeeds. |
| CR-021 | Response headers | Every API response carries `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`. The frontend sends `Content-Security-Policy: default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'`. | Integration test asserts the four backend headers on a 200 and a 404. |
| CR-022 | Retention | Data is kept no longer than the table below and deleted by the retention job (FR-022), which runs every hour. | Integration test `RetentionJobIT` with a fixed clock. |
| CR-028 | Dependencies | CI fails if any backend dependency has a known vulnerability with CVSS ≥ 7.0 (`dependency-check-maven`, `failBuildOnCVSS=7`) or if `npm audit --audit-level=high` reports a finding in the frontend. | CI job result. |

Rate limits (CR-020). Key format `ro:rl:<bucket>:<userId>:<windowStartEpochSeconds>`, `INCR` then `EXPIRE <window>` on first increment:

| Bucket | Applies to | Limit | Window |
|---|---|---|---|
| `all` | every authenticated `/api/ro/v1/**` request | 120 requests | 60 s |
| `upload` | API-004 | 10 requests | 3600 s |
| `analysis` | API-007 | 10 requests | 3600 s |
| `report` | API-010 | 30 requests | 3600 s |
| `export` | API-012 | 5 requests | 86400 s |

The `all` bucket is checked first; a request counts against `all` and against its own bucket. The rate limit is separate from the monthly quota (FR-009): a rate-limited request does not use quota.

Retention (CR-022). All periods are in §3.4:

| Data | Kept for | Deleted how |
|---|---|---|
| Original uploaded file | Not kept: discarded when the upload request ends | Never written to disk or storage (CR-025). |
| Resume metadata and redacted text (TBL-002) | `RESUME_RETENTION` (30 days) after upload | Retention job; or the user (FR-007, FR-021). |
| Analysis and its scores, skills, suggestions (TBL-003…006) | `ANALYSIS_RETENTION` (180 days) after creation | Retention job; or the user (FR-017, FR-021). |
| Consent records (TBL-001) | While active; `AUDIT_RETENTION` (365 days) after withdrawal | Retention job. |
| Erasure records (TBL-010) and audit events (TBL-011) | `AUDIT_RETENTION` (365 days) | Retention job. |
| Monthly usage counters (TBL-007) | 13 months after the period ends | Retention job. |
| Application logs (CloudWatch) | 30 days | CloudWatch log-group retention setting. |
| Database backups (RDS) | 7 days | RDS automated-backup retention; manual snapshots are not allowed in `prod`. |
| Data sent to the AI provider | Per the provider's API retention terms (A-07) | Not controllable by the Module; disclosed in the privacy notice. |

### 1.5a Personal data and the DPDP Act

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-023 | DPDP — notice and consent | Before a user's first upload, the user must accept the current privacy notice (`CONSENT_NOTICE_VERSION`, FR-002). The notice states, in plain English: what is collected (resume content, file name, file type and size, the Platform user id), the single purpose (scoring the resume and producing suggestions), the processors (AWS ap-south-1 for hosting, Anthropic for AI analysis, with processing outside India), the retention periods of CR-022, the rights in CR-024, and the grievance contact `grievance@surfingbear.com`. Uploading or analysing without an active consent for the current version → `403 CONSENT_REQUIRED` (ERR-009). | Integration test: user without consent → API-004 and API-007 return 403 `CONSENT_REQUIRED`. Content review of the notice page against this list. |
| CR-024 | DPDP — rights | The user can, without contacting support: see a summary of their data (API-001), export all of it as JSON (API-012, right to access), delete any single resume or analysis (API-006, API-011), and erase everything and withdraw consent in one action (API-013, right to erasure). Erasure of database data is complete when API-013 returns. | Integration tests for API-012 and API-013 (FR-020, FR-021). |
| CR-025 | DPDP — minimisation | The original file is never stored. Only the normalised text, after redaction of e-mail addresses, phone numbers, URLs and Indian ID numbers (Aadhaar, PAN) by the rules in §4.6.3, is stored and sent to the AI. The Module never reads the name, e-mail or phone claims of the token. | Unit test `PiiRedactorTest` (one case per pattern); code review that no `S3`/file write exists. |
| CR-026 | DPDP — breach | A personal-data breach is handled by the incident runbook: the on-call engineer informs the Data Protection Officer immediately; SurfingBear informs the Data Protection Board of India and each affected user without delay, and sends the Board the detailed report within 72 hours, as the DPDP Rules require. | Runbook `docs/runbooks/personal-data-breach.md` exists and is reviewed before go-live. |
| CR-027 | DPDP — age | Only users the Platform has verified as 18 or older reach the Module (assumption A-03). The Module does not process children's data and has no parental-consent flow. | Assumption confirmed in writing by the Platform team before go-live. |

### 1.6 Internationalization (i18n)

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-029 | i18n | v1 ships one locale, `en-IN`. Every user-facing frontend string comes from `frontend/messages/en-IN.json` through `next-intl`; there are no hard-coded strings in components. Adding a locale must need only a new messages file. | ESLint rule `i18next/no-literal-string` (plugin `eslint-plugin-i18next`) on `src/**/*.tsx` fails the build on a literal UI string. |
| CR-030 | i18n | The frontend shows an error by looking up `errors.<code>` in the messages file; that text equals the Message column of §6.3. If the code is unknown, it shows `errors.UNKNOWN` = "Something went wrong. Please try again." It never shows `error.message` from the server directly. | Unit test: every §6.3 code has a key in `en-IN.json`. |
| CR-031 | Time | Timestamps are stored as `timestamptz` and returned by the API as ISO-8601 UTC with `Z` and milliseconds (`2026-10-16T12:03:44.120Z`). The UI shows them in `Asia/Kolkata` as `dd MMM yyyy, hh:mm a` (e.g. `16 Oct 2026, 05:33 PM`). The quota period and its reset time are computed in `Asia/Kolkata` (FR-009). Numbers use `en-IN` grouping. | Unit test of the date formatter; FR-009 tests at 23:59 and 00:00 IST on the 1st. |
| CR-032 | Content language | Only English resumes are supported, because ATS screening for the target roles is in English and the AI rubric is written for English. The detection rule is in §4.6.2; failing it → `422 RESUME_LANGUAGE_UNSUPPORTED` (ERR-020). AI output and the PDF report are English only. | Unit test `LanguageDetectorTest`. |

### 1.7 Environments

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-033 | Environments | Four environments exist with the differences in the table below. All differences are in `application-<profile>.yml` and environment variables; the same Docker image runs in `dev`, `staging` and `prod`. | Deployment review; the image digest in `prod` equals the one tested in `staging`. |
| CR-034 | Environments | Production data is never copied to `local`, `dev` or `staging`. Test data is synthetic only (the `fixtures/` resumes, §2.6). | Code review; RDS snapshot sharing is disabled in the `prod` account. |

| Aspect | local | dev | staging | prod |
|---|---|---|---|---|
| Spring profile | `local` | `dev` | `staging` | `prod` |
| URL | `http://localhost:3000/resume-optimizer` (frontend), API proxied to `http://localhost:8080` | `https://dev.app.surfingbear.com` | `https://staging.app.surfingbear.com` | `https://app.surfingbear.com` |
| Auth | In-memory dev key + API-017 dev token (no Platform) | Platform dev JWKS | Platform staging JWKS | Platform prod JWKS |
| AI client (`ro.ai.mode`) | `stub` (deterministic, no network, §4.6.6) | `live` | `live` | `live` |
| PostgreSQL | Docker `postgres:16` via `docker compose` | RDS single-AZ `db.t4g.small` | RDS single-AZ `db.t4g.medium` | RDS Multi-AZ `db.m7g.large` |
| Redis | Docker `redis:7` | ElastiCache single node | ElastiCache single node | ElastiCache primary + 1 replica |
| Backend tasks (ECS Fargate) | 1 process | 1 | 2 | 2 minimum, autoscale to 6 |
| Swagger UI `/api/ro/swagger-ui.html` | on | on | off | off |
| Log level root | `DEBUG` for `com.surfingbear` | `INFO` | `INFO` | `INFO` |
| Data | Synthetic | Synthetic | Synthetic | Real |

### 1.8 Deployment

| ID | Area | Rule | How it is checked |
|---|---|---|---|
| CR-035 | CI | Every pull request runs GitHub Actions workflow `ci.yml`: backend `./mvnw -B verify` (Spotless check, Checkstyle, unit tests, Testcontainers integration tests, JaCoCo gate, dependency-check), frontend `npm ci && npm run lint && npm run typecheck && npm test && npm run build`, and `gitleaks`. A PR cannot be merged unless every job is green (branch protection on `main`). | Branch protection settings; workflow file. |
| CR-036 | CD | A merge to `main` builds Docker images `ro-backend:<git-sha>` and `ro-frontend:<git-sha>`, pushes them to Amazon ECR, and deploys them to `dev` automatically. Pushing a tag `vMAJOR.MINOR.PATCH` deploys **the same image digests** to `staging`. `prod` deploys the staging digests after a manual approval in the GitHub environment `prod` (approver: tech lead or engineering manager). | Workflow `cd.yml`; ECS task definitions reference digests. |
| CR-037 | Release | Deployments are rolling with zero downtime: ECS `minimumHealthyPercent=100`, `maximumPercent=200`; a task receives traffic only after `GET /actuator/health/readiness` returns 200. On shutdown the backend stops claiming analyses, finishes or releases in-flight work within 30 s (`server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`); unfinished analyses are picked up again via the lease (FR-019). | Staging deploy during a load test shows no 5xx. |
| CR-038 | Database changes | Schema changes are Flyway migrations in `backend/src/main/resources/db/migration`, applied at application start (`spring.flyway.enabled=true`; `spring.jpa.hibernate.ddl-auto=validate`). A migration must keep the previous application version working (expand/contract): columns are added nullable or with a default, and a column is dropped only one release after the code stopped using it. | Code review checklist; CI boots the previous release image against the new schema in the `migration-compat` job. |
| CR-039 | Rollback | Rollback = redeploy the previous ECS task-definition revision (`cd.yml` job `rollback`, input: target environment). It must complete within 15 minutes. Migrations are never rolled back; because of CR-038 the previous version runs on the new schema. | Rollback drill in `staging` before go-live, timed. |

---

## 2. Coding conventions

### 2.1 Naming

| Item | Convention | Example |
|---|---|---|
| Java base package | `com.surfingbear.resumeoptimizer` | `com.surfingbear.resumeoptimizer.analysis` |
| Java packages | lower case, one word per feature | `resume`, `jobrole`, `analysis` |
| Classes | `PascalCase`, noun; role suffix for Spring beans | `AnalysisService`, `ResumeController`, `AnalysisRepository`, `AnthropicAiClient` |
| JPA entities | `PascalCase` singular noun, **no** suffix | `Analysis`, `Resume`, `JobRole` |
| Request / response DTOs | Java `record`, `<Action><Thing>Request` / `<Thing>Response` | `CreateAnalysisRequest`, `AnalysisResponse` |
| Interfaces | `PascalCase`, no `I` prefix; implementation named by technology | `AiClient` → `AnthropicAiClient`, `StubAiClient` |
| Methods and variables | `camelCase`, methods start with a verb | `claimNextBatch()`, `remainingQuota` |
| Constants | `UPPER_SNAKE_CASE`, `static final` | `MAX_SUGGESTIONS` |
| Enums | type `PascalCase`, constants `UPPER_SNAKE_CASE`; serialised to JSON and DB as the constant name | `AnalysisStatus.PROCESSING` → `"PROCESSING"` |
| Test classes | `<ClassUnderTest>Test` (unit, Surefire), `<Feature>IT` (integration, Failsafe) | `ScoreCalculatorTest`, `CreateAnalysisIT` |
| Test methods | `<method>_<condition>_<expected>` | `create_quotaUsedUp_throwsQuotaExceeded` |
| DB tables | `snake_case`, plural | `analyses`, `job_role_skills` |
| DB columns | `snake_case`; PK `id`; FK `<singular>_id`; time `<verb>_at`; boolean `is_<adjective>`; count `<thing>_count` | `resume_id`, `completed_at`, `is_active`, `attempt_count` |
| DB constraints and indexes | `pk_<table>`, `fk_<table>_<column>`, `ux_<table>_<cols>` (unique), `ix_<table>_<cols>`, `ck_<table>_<rule>` | `ux_analyses_user_id_idempotency_key`, `ck_analyses_score_range` |
| Flyway files | `V<n>__<snake_case_description>.sql`, `n` increments by 1 | `V1__create_schema.sql`, `V2__seed_job_roles.sql` |
| API paths | `/api/ro/v1/` + plural kebab-case nouns; ids as path variables in `camelCase` | `/api/ro/v1/job-roles`, `/api/ro/v1/analyses/{analysisId}/report` |
| JSON properties and query parameters | `camelCase` | `experienceLevel`, `?page=2&size=20` |
| Error codes | `UPPER_SNAKE_CASE` | `QUOTA_EXCEEDED` |
| Config properties | `ro.<area>.<kebab-name>` | `ro.quota.free-monthly-analyses` |
| Environment variables | `RO_<AREA>_<NAME>` | `RO_AI_API_KEY`, `RO_DB_PASSWORD` |
| Metrics | `ro_<noun>_<unit or total>`, snake_case | `ro_analysis_duration_seconds` |
| Frontend components | `PascalCase.tsx`, one exported component per file | `ResumeUploadForm.tsx` |
| Frontend hooks | `use<Thing>.ts` | `useAnalysisPolling.ts` |
| Frontend routes (folders) | kebab-case | `app/analyses/[analysisId]/page.tsx` |
| Frontend message keys | `<area>.<camelCaseKey>` | `upload.dropHint`, `errors.FILE_TOO_LARGE` |

### 2.2 Folder / module structure

One Git repository, `surfingbear-resume-optimizer`, with two deployables:

```text
surfingbear-resume-optimizer/
├── backend/
│   ├── pom.xml
│   ├── mvnw, mvnw.cmd, .mvn/
│   ├── checkstyle.xml
│   └── src/
│       ├── main/java/com/surfingbear/resumeoptimizer/
│       │   ├── ResumeOptimizerApplication.java
│       │   ├── common/
│       │   │   ├── config/       # Clock, Jackson, scheduling, ShedLock, Resilience4j beans
│       │   │   ├── error/        # ErrorCode enum (§6.3), ApiException, GlobalExceptionHandler, ErrorResponse
│       │   │   ├── security/     # SecurityConfig, JwtConfig, CurrentUser, RoleMapper, CsrfHeaderFilter
│       │   │   ├── web/          # RequestIdFilter, SecurityHeadersFilter, PageResponse, PageRequestParams
│       │   │   ├── ratelimit/    # RateLimitFilter, RateLimitBucket
│       │   │   └── time/         # QuotaPeriod (Asia/Kolkata month logic)
│       │   ├── consent/          # ConsentController, ConsentService, Consent, ConsentRepository, dto/
│       │   ├── jobrole/          # JobRoleController, AdminJobRoleController, JobRoleService, JobRole, JobRoleSkill, repositories, dto/
│       │   ├── resume/
│       │   │   ├── ResumeController, ResumeService, Resume, ResumeRepository, dto/
│       │   │   └── parsing/      # FileTypeDetector, PdfTextExtractor, DocxTextExtractor, TextNormaliser,
│       │   │                     # PiiRedactor, LanguageDetector, ParsedResume
│       │   ├── analysis/
│       │   │   ├── AnalysisController, AnalysisService, Analysis, child entities, repositories, dto/
│       │   │   ├── worker/       # AnalysisWorker, AnalysisClaimer, AnalysisProcessor
│       │   │   └── scoring/      # ScoreCalculator, AiOutputValidator, ScoreBand
│       │   ├── ai/               # AiClient, AnthropicAiClient, StubAiClient, PromptBuilder, AiAnalysisOutput, AiFailure
│       │   ├── usage/            # QuotaService, UsagePeriod, UsagePeriodRepository
│       │   ├── report/           # ReportController, ReportService, PdfRenderer
│       │   ├── privacy/          # PrivacyController, DataExportService, DataErasureService, RetentionJob, DataErasureRequest
│       │   ├── audit/            # AuditService, AuditEvent, AuditEventRepository
│       │   └── dev/              # DevTokenController, DevJwtKeyConfig  (@Profile("local") only)
│       ├── main/resources/
│       │   ├── application.yml, application-local.yml, application-dev.yml,
│       │   │   application-staging.yml, application-prod.yml
│       │   ├── logback-spring.xml
│       │   ├── db/migration/     # V1__create_schema.sql, V2__seed_job_roles.sql
│       │   ├── prompts/analysis-system-prompt.txt   # §4.6.4, byte-for-byte
│       │   ├── templates/report/analysis-report.html # Thymeleaf, §4.8
│       │   └── fonts/NotoSans-Regular.ttf, NotoSans-Bold.ttf
│       └── test/
│           ├── java/com/surfingbear/resumeoptimizer/   # mirrors main; a test sits in its subject's package
│           └── resources/fixtures/                      # synthetic resumes, §2.6
├── frontend/
│   ├── package.json, next.config.ts, tsconfig.json, eslint.config.mjs, .prettierrc.json
│   ├── messages/en-IN.json
│   ├── src/
│   │   ├── app/          # routes, §4.7 (basePath /resume-optimizer)
│   │   ├── components/   # shared UI components
│   │   ├── features/     # consent/, upload/, analysis/, history/, privacy/, admin/ — screen-specific components + hooks
│   │   ├── lib/          # apiClient.ts, errors.ts, formatDate.ts, queryClient.ts
│   │   └── types/        # api.ts — TypeScript types matching §6 exactly
│   └── tests/            # Vitest unit tests, Playwright e2e in tests/e2e
├── docker-compose.yml    # local PostgreSQL 16 + Redis 7
├── docs/runbooks/
└── .github/workflows/ci.yml, cd.yml
```

Layering rules (backend):

| Rule | Checked by |
|---|---|
| Controllers call services only; they never call repositories, `AiClient` or other controllers. | ArchUnit test `LayeringArchTest`. |
| `@Transactional` is placed on service methods only; read-only methods use `@Transactional(readOnly = true)`. | ArchUnit test. |
| Entities never leave the service layer; controllers return DTO records only. Mapping is manual in the service (no MapStruct). | ArchUnit test: no controller method returns a type annotated `@Entity`. |
| A feature package may call another feature's **service**, never its repository. `common` must not import any feature package. | ArchUnit test. |
| Services throw `ApiException(ErrorCode, details)`; only `GlobalExceptionHandler` builds error responses. No `try/catch` that swallows an exception. | Code review; ArchUnit forbids `ResponseEntity` construction in services. |
| Time is read only from an injected `java.time.Clock` (bean `Clock.systemUTC()`), never `Instant.now()` without it, so tests can fix time. | Checkstyle `RegexpSinglelineJava` forbidding `Instant.now()` and `LocalDateTime.now()` outside `common/config`. |
| Constructor injection only; no field `@Autowired`. | Checkstyle rule. |
| Logging via SLF4J `private static final Logger log = LoggerFactory.getLogger(X.class);`; no `System.out`/`printStackTrace`. | Checkstyle rule. |

Frontend rules: server state only through TanStack Query hooks in `features/*/`; all HTTP calls only through `lib/apiClient.ts` (which adds `X-Requested-By: ro-web` to non-GET requests, sends `credentials: "same-origin"`, parses the §1.4 error format and performs the CR-004 redirect on 401); no `any` (`@typescript-eslint/no-explicit-any: error`).

### 2.3 Branching and commits

- Trunk-based: `main` is always deployable. Work happens on short-lived branches cut from `main`, merged within 3 working days.
- Branch names: `<type>/<ID>-<short-kebab-description>`, where `<type>` is `feature`, `fix`, `chore`, `docs` or `refactor`, and `<ID>` is the FR/NFR/ERR/CR id the change is for. Examples: `feature/FR-004-resume-upload`, `fix/ERR-015-corrupt-docx`, `chore/CR-028-bump-pdfbox`.
- Commit messages follow Conventional Commits: `<type>(<scope>): <subject>`.
  - `type` ∈ `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`, `chore`.
  - `scope` = backend feature package or frontend feature folder (`analysis`, `resume`, `upload`, `ci`…).
  - `subject`: imperative, lower case, no full stop, at most 72 characters.
  - Footer `Refs: FR-008, ERR-024` lists the affected ids.
  - Example: `feat(analysis): reserve quota inside the creation transaction` / `Refs: FR-009`.
- Pull requests are **squash-merged** into `main`; the PR title must itself be a valid commit message (checked by the `commitlint` job). `main` is protected: no direct pushes, no force pushes, linear history.
- Releases are tags `vMAJOR.MINOR.PATCH` on `main` (SemVer: MAJOR = breaking API change, MINOR = new feature, PATCH = fix).

### 2.4 Code review rules

| Rule | Detail |
|---|---|
| Approvals | 1 approval from a team member other than the author. **2** approvals when the PR touches `common/security/`, `privacy/`, `ai/`, or `db/migration/`. |
| Green CI | All CR-035 jobs green; reviewers do not review a red PR. |
| Size | A PR SHOULD change at most 400 lines (excluding generated files and fixtures); larger PRs explain why in the description. |
| Response time | First review within 1 working day. |
| Description | States the ids it implements (`Implements: FR-008`), how it was tested, and any change to §3.4 constants, API contract or schema. |
| Spec first | A change that alters behaviour described in this spec updates this spec in the same PR (and adds a §8 change-log row). |
| Checklist the reviewer ticks | (1) every new endpoint is in §6 and the traceability matrix; (2) every new error uses a §6.3 code; (3) no personal data logged (CR-010); (4) ownership checked in the query (CR-007); (5) new columns are nullable or defaulted (CR-038); (6) tests cover the acceptance criteria named in the PR. |

### 2.5 Lint and format

| Tool | Scope | Config | When it runs | Fails the build when |
|---|---|---|---|---|
| Spotless `2.44.x` with google-java-format (AOSP style: 4-space indent, 100 columns) | backend `*.java` | `pom.xml` `<spotless>` block | `./mvnw spotless:apply` locally; `spotless:check` in `verify` (CI) | Any file is not formatted |
| Checkstyle `10.x` (`maven-checkstyle-plugin`) | backend | `backend/checkstyle.xml`: no star imports, no unused imports, no `System.out`/`System.err`, no `printStackTrace`, no field `@Autowired`, `Instant.now()` ban (§2.2), method length ≤ 60 lines, max 7 parameters | `validate` phase (CI and local `verify`) | Any violation (severity `error`) |
| ArchUnit `1.3.x` | backend | `LayeringArchTest` | `test` phase | Any rule in §2.2 is broken |
| JaCoCo `0.8.x` | backend | `pom.xml` | `verify` | Line coverage < 80%, measured over classes whose names end in `Service`, `Processor`, `Calculator`, `Validator`, `Redactor`, `Detector` or `Extractor` (NFR-014) |
| ESLint 9 (flat config, `eslint-config-next`, `@typescript-eslint`, `eslint-plugin-i18next`) | frontend `src/**/*.{ts,tsx}` | `frontend/eslint.config.mjs` | `npm run lint` (CI) and lint-staged pre-commit | Any error |
| Prettier 3 | frontend | `.prettierrc.json`: `{"printWidth":100,"singleQuote":true,"trailingComma":"all","semi":true}` | lint-staged pre-commit; `prettier --check` in CI | Any file not formatted |
| TypeScript `strict: true` | frontend | `tsconfig.json` | `npm run typecheck` (`tsc --noEmit`) | Any type error |
| Husky + lint-staged | frontend | `package.json` | `git commit` | Pre-commit lint/format fails |
| markdownlint | `docs/**`, this spec | `.markdownlint.json` (MD013 off) | CI | Any error |

### 2.6 Testing conventions

| Level | Tooling | What it covers | Where |
|---|---|---|---|
| Unit | JUnit 5, AssertJ, Mockito | Pure logic: parsing, redaction, language detection, score calculation, AI-output validation, quota period maths, role mapping | `backend/src/test/java/**/*Test.java` |
| Integration | Spring Boot Test, MockMvc, Testcontainers (`postgres:16`, `redis:7`), WireMock for the Anthropic HTTP API | Every endpoint's happy path and every error row of §6.2; worker; retention; security | `**/*IT.java`, run by Failsafe in `verify` |
| Frontend unit | Vitest + React Testing Library | Components and hooks, including every error code mapping | `frontend/tests/**/*.test.tsx` |
| End-to-end | Playwright against `docker compose` + backend `local` profile + stub AI | The journeys in §4.3 | `frontend/tests/e2e/` |
| AI quality | Script `backend/src/test/eval/run-eval.sh` against the live model (manual, before changing the prompt or model) | NFR-017 consistency | Not in CI (costs money) |

Fixtures are synthetic and committed under `backend/src/test/resources/fixtures/`: `valid-2page.pdf`, `valid.docx`, `scanned-image-only.pdf` (no text layer), `encrypted.pdf` (password `test`), `corrupt.pdf` (truncated), `eleven-pages.pdf`, `hindi.pdf` (Devanagari text), `tiny.pdf` (< 300 characters), `huge-text.docx` (> 50,000 characters), `zip-bomb.docx`, `png-renamed.docx`, `legacy.doc`. No fixture contains a real person's data.

---

## 3. Requirements definition

_Being written._

---

## 4. Architecture

_Being written._

---

## 5. ERD and database design

_Being written._

---

## 6. API design

_Being written._

---

## 7. Traceability matrix

_Being written._

---

## 8. Change log

_Being written._
