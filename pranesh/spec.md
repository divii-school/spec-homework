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

Priorities: **Must** = needed for go-live; **Should** = planned for go-live, may slip one release with product-owner approval; **Could** = nice to have. Every FR below is **Must** unless stated.

### 3.1 Functional requirements

| ID | Title | Description | Priority | Acceptance criteria |
|---|---|---|---|---|
| FR-001 | My context | A logged-in user opens the Module and sees their plan, this month's usage and whether they have accepted the current privacy notice. | Must | 1. API-001 returns `userId` (= token `sub`), `plan`, `isAdmin`, `consent` and `usage` as in §6.2. 2. `usage.limit` is `FREE_MONTHLY_ANALYSES` for FREE and `PREMIUM_MONTHLY_ANALYSES` for PREMIUM; `usage.used` is the counter of the current quota period (FR-009), `0` if no row exists; `usage.remaining = max(0, limit − used)`. 3. `usage.resetsAt` is 00:00 `Asia/Kolkata` on the 1st of the next month, returned in UTC (e.g. period `2026-10` → `2026-10-31T18:30:00.000Z`). 4. Works without consent (it is how the frontend learns consent is missing). |
| FR-002 | Privacy notice and consent | Before the first upload, the user reads the privacy notice (CR-023) and accepts it. A new notice version requires a new acceptance. | Must | 1. The frontend shows the notice page (§4.7 `/consent`) whenever API-001 returns `consent.accepted = false` and the user tries to upload or analyse. 2. API-002 with `noticeVersion = CONSENT_NOTICE_VERSION` creates one TBL-001 row (`accepted_at = now`) and returns 200; calling it again while that consent is active returns 200 with the **same** row and creates nothing. 3. Any other `noticeVersion` → `409 CONSENT_VERSION_MISMATCH` (ERR-010) with `details.currentVersion`. 4. API-004 and API-007 return `403 CONSENT_REQUIRED` (ERR-009) when the user has no active consent row for `CONSENT_NOTICE_VERSION`. 5. Accepting writes audit event `CONSENT_ACCEPTED` (CR-040). 6. Existing resumes and analyses stay readable when the notice version changes; only new uploads and analyses are blocked until re-acceptance. |
| FR-003 | Job-role catalogue | The user picks the target job role from a list maintained by admins (FR-023). Free-text roles are not supported (OOS-05). | Must | 1. API-003 returns all job roles with `isActive = true`, sorted by `name` ascending (case-insensitive), each with its skills in `displayOrder`. 2. Inactive roles are never returned by API-003 and cannot be used for a new analysis (`422 JOB_ROLE_NOT_AVAILABLE`, ERR-022). 3. The v1 catalogue is seeded with the 10 roles of §5.6. 4. The list is not paginated (at most 200 roles, enforced by FR-023). |
| FR-004 | Upload resume | The user uploads one PDF or DOCX file. The Module validates it, extracts and redacts the text (FR-005) and stores the result as a Resume. | Must | Checks run **in this order**; the first failing check decides the error and nothing is stored: 1. No active consent → `403 CONSENT_REQUIRED`. 2. The user already has `MAX_STORED_RESUMES` resumes → `409 RESUME_LIMIT_REACHED` (ERR-021) with `details.maxResumes`. 3. Multipart part `file` missing or 0 bytes → `400 FILE_MISSING` (ERR-011). 4. Size > `MAX_FILE_BYTES` (exactly 5,242,880 bytes is accepted) → `413 FILE_TOO_LARGE` (ERR-012) with `details.maxBytes`. 5. Content is not PDF or DOCX by the §4.6.1 rules (including legacy `.doc`, images, `.txt`, `.odt`) → `415 UNSUPPORTED_FILE_TYPE` (ERR-013). 6. Password-protected PDF or DOCX → `422 FILE_ENCRYPTED` (ERR-014). 7. The parser throws → `422 FILE_CORRUPT` (ERR-015). 8. Parsing takes longer than `PARSE_TIMEOUT` → `422 FILE_PROCESSING_TIMEOUT` (ERR-016). 9. PDF with more than `MAX_PDF_PAGES` pages → `422 RESUME_TOO_MANY_PAGES` (ERR-017) with `details.maxPages`. 10. Normalised text shorter than `MIN_TEXT_CHARS` (this includes scanned, image-only PDFs) → `422 RESUME_TEXT_TOO_SHORT` (ERR-018). 11. Longer than `MAX_TEXT_CHARS` → `422 RESUME_TEXT_TOO_LONG` (ERR-019). 12. Not English by §4.6.2 → `422 RESUME_LANGUAGE_UNSUPPORTED` (ERR-020). 13. On success: one TBL-002 row is stored, `expires_at = created_at + RESUME_RETENTION`, and API-004 returns `201` with the resume (§6.2). 14. The stored filename is the multipart filename with any path removed, control characters removed, trimmed, cut to 255 characters; if empty it becomes `resume.pdf` or `resume.docx`. 15. The original bytes are never written to disk, database or object storage (CR-025). |
| FR-005 | Text extraction and redaction | The Module turns the file into clean text and removes contact details and ID numbers before storing it or sending it anywhere. | Must | 1. PDF text is extracted with PDFBox `PDFTextStripper` (sort by position on); DOCX text with POI `XWPFWordExtractor` (body, tables, headers and footers). 2. Normalisation and the length used in FR-004 checks 10–11 follow §4.6.2 exactly. 3. Redaction follows §4.6.3 exactly and happens **after** the length and language checks, on the normalised text; `redaction_count` stores how many replacements were made. 4. Formatting signals are recorded: `page_count` (PDF only, else null), `image_count` (PDF image XObjects across all pages; DOCX `XWPFPictureData` count), `table_count` (DOCX tables; `0` for PDF because PDF has no table structure). 5. Only the redacted text is stored (`text_redacted`). |
| FR-006 | List my resumes | The user sees their stored resumes, newest first, to reuse one for a new analysis. | Must | 1. API-005 returns only the caller's resumes, sorted by `createdAt` descending then `id` descending, paginated per §6.1. 2. Each item has `id`, `filename`, `fileType`, `fileSizeBytes`, `pageCount`, `createdAt`, `expiresAt`, `analysisCount` (number of the user's analyses with this `resume_id`). 3. Resumes past `expiresAt` that the retention job has not removed yet are **not** returned. 4. The redacted text is not part of the list (it is only in the data export, FR-020). |
| FR-007 | Delete a resume | The user deletes a resume. Its analyses are deleted with it, because they contain quoted lines of the resume. | Must | 1. API-006 deletes the TBL-002 row and every TBL-003 row with that `resume_id` (and their child rows, by cascade) in one transaction, then returns `204`. 2. If any of those analyses is `QUEUED` or `PROCESSING` → `409 ANALYSIS_IN_PROGRESS` (ERR-024) and nothing is deleted. 3. Another user's or a missing id → `404 RESOURCE_NOT_FOUND`. 4. Quota already used by the deleted analyses is **not** returned (FR-009 criterion 7). |
| FR-008 | Request an analysis | The user chooses a resume, a job role and an experience level and starts an analysis. Processing is asynchronous. | Must | Checks run in this order: 1. Header `Idempotency-Key` missing or not a UUID → `400 VALIDATION_FAILED` with `fieldErrors[0].field = "Idempotency-Key"`. 2. Body invalid (missing field, `experienceLevel` not one of `ENTRY`, `MID`, `SENIOR`, `LEAD`) → `400 VALIDATION_FAILED`. 3. An analysis of this user with the same idempotency key exists: if its `resumeId`, `jobRoleId` and `experienceLevel` are equal to the request, return `200` with that analysis' current state (no new row, no quota); otherwise `409 IDEMPOTENCY_KEY_CONFLICT` (ERR-025). 4. No active consent → `403 CONSENT_REQUIRED`. 5. Resume not found, not owned or expired → `404 RESOURCE_NOT_FOUND`. 6. Job role missing or inactive → `422 JOB_ROLE_NOT_AVAILABLE`. 7. The user already has an analysis in `QUEUED` or `PROCESSING` → `409 ANALYSIS_IN_PROGRESS` (at most one at a time, also enforced by index `ux_analyses_user_id_in_progress`). 8. Quota used up → `403 QUOTA_EXCEEDED` (FR-009). 9. On success one TBL-003 row is stored with `status = QUEUED`, the job-role name and resume filename copied in, `plan_at_creation` set, and API-007 returns `202` with the analysis (§6.2). 10. Steps 4–9 run in one database transaction together with the quota reservation, so two simultaneous requests can never both pass step 7 or step 8. |
| FR-009 | Monthly quota | Free users get a limited number of analyses per month; premium users get a fair-use cap. | Must | 1. Limits: FREE = `FREE_MONTHLY_ANALYSES` (3), PREMIUM = `PREMIUM_MONTHLY_ANALYSES` (50), per quota period. 2. The quota period of an analysis is the `YYYY-MM` of its creation time in `Asia/Kolkata` (an analysis created at `2026-10-31T18:30:00Z` belongs to `2026-11`). 3. The plan used is the one in the token at creation time; the limit is that plan's limit. 4. Creation locks the user's TBL-007 row for the period (`INSERT … ON CONFLICT DO NOTHING`, then `SELECT … FOR UPDATE`); if `used_count >= limit` → `403 QUOTA_EXCEEDED` (ERR-023) with `details = {limit, used, resetsAt}`; otherwise `used_count + 1` in the same transaction. 5. An analysis that ends `FAILED` gives its unit back: `used_count − 1` for **its own** period, in the same transaction that sets `FAILED` (never below 0). 6. `COMPLETED` keeps the unit. 7. Deleting an analysis or resume, or erasing data (FR-021), does not give units back, and TBL-007 rows survive erasure so erasure cannot be used to reset the quota. 8. A user downgraded from PREMIUM to FREE mid-month with `used ≥ 3` gets `QUOTA_EXCEEDED` until the next period. |
| FR-010 | AI analysis processing | A background worker analyses each queued resume with the AI provider and stores the result. | Must | 1. The worker (CMP-010) claims queued analyses as in §4.3.3 and §4.6.5, oldest `created_at` first, at most `WORKER_CONCURRENCY` at a time per instance. 2. It builds the prompt from §4.6.4 using the resume's redacted text, the job role and its skills, the experience level and the formatting signals, and calls the AI once with structured output (§4.6.4). 3. It validates the output with §4.6.5; invalid output (rule V1, or unparseable JSON) triggers one more call (`AI_INVALID_OUTPUT_RETRIES`); a second invalid output → `FAILED` / `AI_INVALID_RESPONSE`. 4. On valid output, in one transaction: 6 dimension rows (TBL-004), one skill row per catalogue skill (TBL-005), 0–`MAX_SUGGESTIONS` suggestion rows (TBL-006), `overall_score`, `status = COMPLETED`, `completed_at`, token counts and `ai_model`. 5. The transaction commits only if the analysis is still `PROCESSING` and still leased by this worker (`UPDATE … WHERE id = ? AND status = 'PROCESSING' AND locked_by = ?` affects 1 row); otherwise everything is rolled back and the result discarded (the analysis was erased or re-claimed). |
| FR-011 | ATS score | Each analysis has an overall score out of 100 made of six dimension scores. | Must | 1. The six dimensions, weights and meaning are fixed in §4.6.4 table "Dimensions"; weights sum to 1.00. 2. Each dimension score is an integer 0–100 from the AI. 3. `overallScore = round_half_up(Σ score × weight)`, computed by the backend (CMP-012), never taken from the AI. Example: scores 80, 60, 70, 90, 50, 100 → 24 + 12 + 10.5 + 13.5 + 5 + 10 = 75. 4. `scoreBand`: 0–49 `NEEDS_WORK`, 50–74 `FAIR`, 75–89 `GOOD`, 90–100 `EXCELLENT`. 5. Every dimension has a `rationale` of at most 300 characters. |
| FR-012 | Missing skills | The analysis lists which skills of the chosen job role the resume shows and which it is missing. | Must | 1. The AI only says which catalogue skills are present (`skillsPresent`); the backend computes `missing = catalogue skills − present`. 2. A name returned by the AI counts only if it equals a catalogue skill name ignoring case; other names are ignored and logged as `WARN` (count only, no text). 3. Both lists are ordered `REQUIRED` first, then `PREFERRED`, each by catalogue `displayOrder`. 4. The skills are copied into TBL-005 at completion, so later catalogue edits do not change old analyses. 5. Both lists are shown to FREE and PREMIUM users alike. |
| FR-013 | Rewrite suggestions | The analysis suggests better wording for weak resume lines. | Must | 1. The AI returns 0–10 suggestions ordered by impact. 2. The backend keeps a suggestion only if it passes every rule V3 in §4.6.5 (its `originalText` really occurs in the resume, it differs from `suggestedText`, lengths within limits, not a duplicate). 3. Kept suggestions get `priority` 1, 2, 3… in the AI's order. 4. Zero suggestions is a valid result; the UI then shows "No rewrites needed — your lines are already strong." (key `analysis.noSuggestions`). |
| FR-014 | Plan-based suggestion visibility | Free users see only some suggestions. | Must | 1. Visibility uses the plan in the token **at read time**, so an upgrade unlocks old analyses at once. 2. FREE: `suggestions.visible` holds those with `priority ≤ FREE_VISIBLE_SUGGESTIONS`; `suggestions.lockedCount = total − visible`. The text of locked suggestions is never sent to a FREE user (not in API-009, not in API-010, not in API-008). 3. PREMIUM: all suggestions visible, `lockedCount = 0`. 4. The data export (FR-020) always contains all suggestions, because it is the user's own data (CR-024). |
| FR-015 | View an analysis | The user watches the analysis progress and reads the result. | Must | 1. API-009 returns the analysis in any status; result fields (`overallScore`, `scoreBand`, `dimensions`, `skills`, `suggestions`) are `null` unless `status = COMPLETED`; `failureCode` is non-null only when `FAILED`. 2. The frontend polls API-009 every `CLIENT_POLL_INTERVAL` while the status is `QUEUED` or `PROCESSING`; after 120 s it shows "This is taking longer than usual. You can leave this page — the result will appear in your history." (key `analysis.slow`) and polls every 10 s. Polling stops at `COMPLETED` or `FAILED` or when the user leaves the page. 3. A `FAILED` analysis shows the text for its `failureCode` (§6.3) and a "Try again" button that starts FR-008 with a **new** idempotency key and the same inputs. |
| FR-016 | Analysis history | The user sees all their analyses. | Must | 1. API-008 returns the caller's analyses sorted by `createdAt` descending then `id` descending, paginated (§6.1). 2. Optional filter `status` (one of the four statuses). 3. Each item has `id`, `status`, `failureCode`, `resumeId` (null after the resume expired), `resumeFilename`, `jobRole {id, name}`, `experienceLevel`, `overallScore`, `scoreBand`, `createdAt`, `completedAt`; no suggestion text. |
| FR-017 | Delete an analysis | The user deletes one analysis. | Must | 1. API-011 deletes the analysis and its child rows and returns `204`. 2. `QUEUED` or `PROCESSING` → `409 ANALYSIS_IN_PROGRESS`. 3. Missing or not owned → `404 RESOURCE_NOT_FOUND`. 4. No quota is given back. |
| FR-018 | Download PDF report | The user downloads a completed analysis as a PDF. | Must | 1. API-010 returns `200`, `Content-Type: application/pdf`, `Content-Disposition: attachment; filename="surfingbear-resume-analysis-<yyyyMMdd>-<first 8 chars of id>.pdf"` (date = `createdAt` in `Asia/Kolkata`). 2. Status not `COMPLETED` → `409 ANALYSIS_NOT_COMPLETED` (ERR-026). 3. Content and layout follow §4.8; a FREE user's report contains only visible suggestions plus the locked-count notice. 4. The PDF is generated on each request and never stored. 5. A report for an analysis with 10 suggestions is generated in under 3 s at p95 (NFR-004). |
| FR-019 | Failure handling | AI errors, timeouts and crashes never leave an analysis stuck and never cost the user quota. | Must | 1. AI call failures map to failure codes per §4.4: provider unavailable / rate-limited / timeout after the SDK retry → `AI_UNAVAILABLE` (ERR-031); invalid output twice → `AI_INVALID_RESPONSE` (ERR-032); `stop_reason = refusal` → `AI_REFUSED` (ERR-034), not retried; processing past `PROCESSING_DEADLINE` → `PROCESSING_TIMEOUT` (ERR-033). 2. While the circuit breaker (§4.4) is open, analyses fail at once with `AI_UNAVAILABLE`. 3. A lease that expires (worker crash) makes the analysis claimable again; on the claim that would be attempt `MAX_ATTEMPTS + 1`, it is set `FAILED` / `PROCESSING_TIMEOUT` instead. 4. A `QUEUED` analysis older than `MAX_QUEUE_AGE` is set `FAILED` / `PROCESSING_TIMEOUT` when the claimer sees it. 5. Every transition to `FAILED` gives the quota unit back (FR-009 criterion 5) and sets `completed_at`. 6. No analysis stays in `QUEUED` or `PROCESSING` longer than `MAX_QUEUE_AGE + MAX_ATTEMPTS × LEASE_DURATION` (26 minutes) (NFR-011). |
| FR-020 | Export my data | The user downloads everything the Module holds about them. | Must | 1. API-012 returns `200`, `Content-Type: application/json`, `Content-Disposition: attachment; filename="surfingbear-resume-optimizer-data-<yyyyMMdd>.json"`. 2. The body contains `exportedAt`, `userId`, `consents[]`, `resumes[]` (all fields including `textRedacted`), `analyses[]` (all fields, all dimensions, both skill lists and **all** suggestions regardless of plan) and `usage[]` (all TBL-007 rows), with the shapes in §6.2 API-012. 3. Writes audit event `DATA_EXPORTED`. 4. Rate-limited by bucket `export` (CR-020). |
| FR-021 | Erase my data | The user deletes all their data and withdraws consent in one action. | Must | 1. API-013 requires body `{"confirm":"DELETE_MY_DATA"}`; anything else → `400 VALIDATION_FAILED`. 2. In one transaction: delete all the user's TBL-003 rows (children cascade) **including** `QUEUED`/`PROCESSING` ones, delete all TBL-002 rows, set `withdrawn_at = now` on every active TBL-001 row, insert one TBL-010 row with the counts, and write audit event `DATA_ERASED`. 3. Returns `200` with `erasureId`, `deletedResumeCount`, `deletedAnalysisCount`, `completedAt`. 4. A worker processing an erased analysis discards its result (FR-010 criterion 5). 5. Afterwards API-001 shows `consent.accepted = false`; uploads need a new consent. 6. TBL-007 usage rows are kept (FR-009 criterion 7). 7. Completes within 10 s for a user holding `MAX_STORED_RESUMES` resumes and 300 analyses (NFR-009). |
| FR-022 | Automatic retention | Data is deleted automatically when its retention period ends. | Must | 1. `RetentionJob` runs at minute 15 of every hour (`cron = "0 15 * * * *"`, zone UTC), guarded by ShedLock so only one instance runs it (`lockAtMostFor = 50m`). 2. It deletes, in batches of 500 rows per transaction: analyses with `expires_at <= now` that are not `QUEUED`/`PROCESSING`; resumes with `expires_at <= now` that have no `QUEUED`/`PROCESSING` analysis (their remaining analyses keep `resume_id = NULL` via `ON DELETE SET NULL`); consents withdrawn more than `AUDIT_RETENTION` ago; TBL-010 and TBL-011 rows older than `AUDIT_RETENTION`; TBL-007 rows whose period ended more than 13 months ago. 3. It logs one `INFO` line with the deleted counts per table, and `ERROR` (plus metric `ro_retention_failures_total`) on failure; the next run retries. 4. Data is never kept more than 1 hour 15 minutes past its expiry (NFR-008). |
| FR-023 | Manage job-role catalogue | An admin creates and edits job roles and their skills. Roles are deactivated, never deleted, because analyses reference them. | Should | 1. API-014 lists all roles including inactive ones, sorted by name. 2. API-015 creates a role; API-016 replaces a role's `name`, `description`, `isActive` and the full `skills` list. 3. Validation: `name` 3–100 characters, unique ignoring case (else `409 JOB_ROLE_NAME_TAKEN`, ERR-028); `description` 0–500 characters; `skills` 1–40 items, each `name` 1–60 characters, unique within the role ignoring case, `importance` ∈ `REQUIRED`, `PREFERRED`; at least one `REQUIRED` skill. 4. Creating the 201st role → `409 JOB_ROLE_LIMIT_REACHED` (ERR-035). 5. Changes affect only analyses created afterwards (FR-012 criterion 4). 6. Each create/update writes audit event `JOB_ROLE_CREATED` / `JOB_ROLE_UPDATED`. 7. Non-admins get `403 FORBIDDEN`. |
| FR-024 | Upgrade prompt | Free users are told what they are missing and where to upgrade. | Should | 1. When `suggestions.lockedCount > 0`, the analysis page shows "Upgrade to Premium to see {lockedCount} more suggestions" (key `analysis.lockedCta`) linking to `UPGRADE_URL` in the same tab. 2. When API-007 returns `QUOTA_EXCEEDED` for a FREE user, the page shows the error text plus the same upgrade link and the reset date (`details.resetsAt`, shown per CR-031). 3. PREMIUM users never see an upgrade prompt. 4. The Module does not handle payment (OOS-02). |

### 3.2 Non-functional requirements

| ID | Category | Requirement | Measurable target | How it is verified |
|---|---|---|---|---|
| NFR-001 | Performance | Resume upload (API-004) including validation, extraction and redaction | p95 ≤ 3 s, p99 ≤ 5 s for a 5 MB, 10-page PDF at 5 uploads/s | k6 load test `upload.js` in staging before each MINOR release |
| NFR-002 | Performance | All JSON read endpoints (API-001, 003, 005, 008, 009) | p95 ≤ 300 ms, p99 ≤ 800 ms server-side at 50 requests/s | k6 `read-mix.js`; metric `http_server_requests_seconds` in prod |
| NFR-003 | Performance | Analysis end-to-end, from API-007 response to `COMPLETED` | p95 ≤ 60 s, p99 ≤ 120 s | Metric `ro_analysis_duration_seconds{status="COMPLETED"}`; staging test with live model on the 20-resume eval set |
| NFR-004 | Performance | PDF report generation (API-010) | p95 ≤ 3 s for an analysis with 10 suggestions | k6 `report.js` |
| NFR-005 | Availability | API availability, measured as the share of `/api/ro/v1/**` requests that do not end in 5xx, per calendar month, excluding announced maintenance (announced ≥ 48 h ahead, ≤ 2 h per month) | ≥ 99.5% (≈ 3 h 39 min of errors per month) | CloudWatch metric math on ALB `HTTPCode_Target_5XX_Count` / `RequestCount`; monthly report |
| NFR-006 | Scalability | Capacity without code change | 2,000 analyses/day, peak 300 analyses/hour, 200 concurrent users on 2 backend tasks (1 vCPU / 2 GB each); scales out to 6 tasks when average CPU > 60% for 3 minutes, in when < 30% for 15 minutes | k6 `peak.js` at 300 analyses/hour (stub AI with 20 s latency) shows no SLO breach; ECS autoscaling policy review |
| NFR-007 | Security | Known vulnerabilities and web-security baseline | 0 dependencies with CVSS ≥ 7.0 at release (CR-028); OWASP ZAP baseline scan of staging: 0 High, 0 Medium findings | CI dependency-check; ZAP scan job before each release |
| NFR-008 | Privacy | No personal data in logs; redaction before AI; retention on time | Canary test finds 0 occurrences (CR-010); 100% of the patterns in `PiiRedactorTest` redacted; no row exists more than 1 h 15 min after its `expires_at` | `NoPersonalDataInLogsIT`, `PiiRedactorTest`, `RetentionJobIT`; weekly prod query `SELECT count(*) FROM resumes WHERE expires_at < now() - interval '75 minutes'` = 0 |
| NFR-009 | Privacy | Erasure speed and completeness | API-013 completes in ≤ 10 s for 20 resumes + 300 analyses; after it, 0 rows with that `user_id` remain in TBL-002…006 and active TBL-001; backups age out within 7 days | `DataErasureIT` with that data volume |
| NFR-010 | Observability | Metrics, logs and alerts | 100% of requests carry `X-Request-Id` (CR-011); the metrics in §4.9 exist; the alerts in §4.9 fire within 5 minutes of their condition | `ObservabilityIT` asserts metric names on `/actuator/prometheus`; alert test in staging by injecting AI failures via WireMock |
| NFR-011 | Reliability | No analysis is stuck; crashes recover | 0 analyses in `QUEUED`/`PROCESSING` older than 26 minutes; after killing a backend task mid-analysis, the analysis reaches a terminal state within `LEASE_DURATION + 60 s` | `WorkerRecoveryIT` (kill simulated by expiring the lease); prod alert `ro_analysis_stuck` |
| NFR-012 | Backup and recovery | Database recovery | RPO ≤ 5 minutes (RDS point-in-time recovery), RTO ≤ 4 hours | Restore drill in staging every 6 months, timed |
| NFR-013 | Cost | AI cost per completed analysis | Average ≤ US$ 0.15 per completed analysis per month, computed from `ro_ai_tokens_total` × the model's per-token price | Monthly cost review; alert if the 7-day average exceeds US$ 0.20 |
| NFR-014 | Maintainability | Code quality gates | Line coverage ≥ 80% on the class set in §2.5; 0 ArchUnit violations; 0 Checkstyle errors | CI (`./mvnw verify`) |
| NFR-015 | Accessibility | Frontend accessibility | WCAG 2.1 AA; axe-core reports 0 `serious` or `critical` violations on every route in §4.7; all actions usable by keyboard only | Playwright + `@axe-core/playwright` in CI |
| NFR-016 | Compatibility | Supported clients | Latest 2 major versions of Chrome, Edge, Firefox and Safari (desktop and mobile); layouts work from 360 px to 1920 px width | Playwright projects for Chromium, Firefox, WebKit at 360 px and 1280 px |
| NFR-017 | AI quality | Scoring consistency | Analysing the same resume with the same role and level 3 times gives overall scores within 5 points of each other for ≥ 90% of the 20 resumes in the eval set | `run-eval.sh` before any change to the model, effort or prompt; result attached to the PR |

### 3.3 Assumptions and out of scope

**Assumptions** (each one is a decision this spec makes where the product description is silent):

| ID | Assumption | Reason |
|---|---|---|
| A-01 | The Platform issues an RS256 JWT with the claims in §1.1, stored in the httpOnly, `Secure`, `SameSite=Lax` cookie `sb_at` on host `app.surfingbear.com`, valid for 15 minutes and refreshed by the Platform. JWKS: `https://auth.surfingbear.com/.well-known/jwks.json`, issuer `https://auth.surfingbear.com`. | The description says users are already logged in; the Module needs a verifiable identity and plan without calling the Platform on every request. |
| A-02 | The Platform puts the subscription plan in the `sb_plan` claim. | Avoids a runtime dependency on a billing API; a plan change takes effect within one token lifetime (≤ 15 minutes), which is acceptable. |
| A-03 | Users are 18 or older (the Platform enforces this). | DPDP requires verifiable parental consent for children; that flow is not part of this Module. |
| A-04 | The Module is served under the Platform's host (`/resume-optimizer` for the UI, `/api/ro/v1` for the API) through path routing on the Platform's ALB. | Same origin means the Platform cookie reaches the API, and no CORS is needed. |
| A-05 | `FREE_MONTHLY_ANALYSES = 3` and `FREE_VISIBLE_SUGGESTIONS = 3`. | Enough to try the product a few times, small enough to show the value of Premium. |
| A-06 | Premium has a fair-use cap of 50 analyses per month. | Bounds AI cost (NFR-013) per user; well above realistic use (a job seeker tailors a few resumes per week). |
| A-07 | Anthropic (Claude API) is the AI provider, under a data processing agreement; it processes data outside India. | Reliable structured JSON output and strong instruction following for a rubric task; cross-border transfer is allowed by the DPDP Act except to countries the Government restricts, and it is disclosed in the notice (CR-023). |
| A-08 | Resumes are in English (CR-032). | ATS screening for the seeded roles is in English. |
| A-09 | The original file is not needed after upload. | Users download the analysis, not their own file back; storing files would add an object store and a breach surface (CR-025). |
| A-10 | Platform user ids (`sub`) are pseudonymous UUIDs and may be logged and stored. | They are needed to link data to its owner and contain no direct identifier. |

**Out of scope** (this spec deliberately does not cover):

| ID | Item |
|---|---|
| OOS-01 | Sign-up, login, logout, password reset, session refresh (the Platform does these). |
| OOS-02 | Payment, subscription purchase or change (the Platform does these; the Module only links to `UPGRADE_URL`). |
| OOS-03 | OCR of scanned or image-only resumes (they are rejected with `RESUME_TEXT_TOO_SHORT`). |
| OOS-04 | Legacy `.doc`, `.rtf`, `.odt`, `.txt` and image uploads. |
| OOS-05 | Free-text job roles or job-description pasting. |
| OOS-06 | Editing the resume inside the Module, or generating a rewritten resume file. |
| OOS-07 | Non-English resumes and UI locales other than `en-IN` (the UI is i18n-ready, CR-029). |
| OOS-08 | E-mail or push notifications when an analysis finishes. |
| OOS-09 | A native mobile app (the web UI is responsive, NFR-016). |
| OOS-10 | Admin access to users' resumes or analyses, and admin analytics screens (operations use metrics, §4.9). |

### 3.4 Business constants

The single source for every number used in more than one place. Property names are Spring properties under `application.yml`; values marked **fixed** are not configurable.

| Name | Value | Property | Used in |
|---|---|---|---|
| `MAX_FILE_BYTES` | 5,242,880 bytes (5 MB) | `ro.upload.max-file-bytes` (also `spring.servlet.multipart.max-file-size=5MB`, `max-request-size=6MB`) | FR-004, API-004, ERR-012 |
| `MAX_PDF_PAGES` | 10 | `ro.upload.max-pdf-pages` | FR-004, ERR-017, TBL-002 |
| `PARSE_TIMEOUT` | 10 s | `ro.upload.parse-timeout` | FR-004, ERR-016 |
| `MIN_TEXT_CHARS` | 300 characters (after normalisation) | `ro.upload.min-text-chars` | FR-004, ERR-018 |
| `MAX_TEXT_CHARS` | 50,000 characters (after normalisation) | `ro.upload.max-text-chars` | FR-004, ERR-019, TBL-002 |
| `MIN_LATIN_RATIO` | 0.70 | `ro.upload.min-latin-ratio` | §4.6.2, ERR-020 |
| `MAX_STORED_RESUMES` | 20 per user | `ro.upload.max-stored-resumes` | FR-004, ERR-021 |
| `FREE_MONTHLY_ANALYSES` | 3 per quota period | `ro.quota.free-monthly-analyses` | FR-001, FR-009 |
| `PREMIUM_MONTHLY_ANALYSES` | 50 per quota period | `ro.quota.premium-monthly-analyses` | FR-001, FR-009 |
| `QUOTA_TIME_ZONE` | `Asia/Kolkata` | **fixed** | FR-009, CR-031 |
| `MAX_IN_PROGRESS_PER_USER` | 1 | **fixed** (DB index) | FR-008, ERR-024 |
| `FREE_VISIBLE_SUGGESTIONS` | 3 | `ro.analysis.free-visible-suggestions` | FR-014 |
| `MAX_SUGGESTIONS` | 10 | **fixed** (in the AI output schema) | FR-013, TBL-006 |
| `AI_MODEL` | `claude-opus-5-5` | `ro.ai.model` | §4.6.4, TBL-003 `ai_model` |
| `AI_EFFORT` | `medium` | `ro.ai.effort` | §4.6.4 |
| `AI_MAX_TOKENS` | 16,000 | `ro.ai.max-tokens` | §4.6.4 |
| `AI_REQUEST_TIMEOUT` | 75 s per HTTP attempt | `ro.ai.request-timeout` | §4.4 |
| `AI_SDK_MAX_RETRIES` | 1 (so at most 2 HTTP attempts per call) | `ro.ai.max-retries` | §4.4 |
| `AI_INVALID_OUTPUT_RETRIES` | 1 | `ro.ai.invalid-output-retries` | FR-010, §4.6.5 |
| `PROCESSING_DEADLINE` | 300 s per attempt | `ro.analysis.processing-deadline` | FR-019 |
| `LEASE_DURATION` | 330 s | `ro.analysis.lease-duration` | FR-019, §4.6.5 |
| `MAX_ATTEMPTS` | 2 | `ro.analysis.max-attempts` | FR-019, TBL-003 |
| `MAX_QUEUE_AGE` | 900 s | `ro.analysis.max-queue-age` | FR-019 |
| `WORKER_POLL_INTERVAL` | 1 s | `ro.worker.poll-interval` | §4.6.5 |
| `WORKER_CONCURRENCY` | 4 per backend task | `ro.worker.concurrency` | FR-010, NFR-006 |
| `CIRCUIT_BREAKER` | sliding window 20 calls, opens at ≥ 50% failures, stays open 60 s, 3 trial calls when half-open | `resilience4j.circuitbreaker.instances.anthropic.*` | §4.4, FR-019 |
| `RESUME_RETENTION` | 30 days | `ro.retention.resume` | CR-022, FR-004, FR-022 |
| `ANALYSIS_RETENTION` | 180 days | `ro.retention.analysis` | CR-022, FR-022 |
| `AUDIT_RETENTION` | 365 days | `ro.retention.audit` | CR-022, FR-022 |
| `USAGE_RETENTION` | 13 months after the period | `ro.retention.usage-months` | CR-022, FR-022 |
| `CONSENT_NOTICE_VERSION` | `2026-10-v1` | `ro.consent.current-version` | FR-002, API-001, API-002 |
| `CLIENT_POLL_INTERVAL` | 3 s (10 s after 120 s) | frontend `NEXT_PUBLIC_POLL_INTERVAL_MS=3000` | FR-015 |
| `DEFAULT_PAGE_SIZE` / `MAX_PAGE_SIZE` | 20 / 100 | **fixed** | §6.1 |
| `UPGRADE_URL` | `https://app.surfingbear.com/pricing` | frontend `NEXT_PUBLIC_UPGRADE_URL` | FR-024 |
| `LOGIN_URL` | `https://app.surfingbear.com/login` | frontend `NEXT_PUBLIC_LOGIN_URL` | CR-004 |

Experience levels (fixed enum, used by FR-008, the prompt and TBL-003):

| Value | Label (UI) | Years of experience given to the AI |
|---|---|---|
| `ENTRY` | Entry level (0–2 years) | 0–2 |
| `MID` | Mid level (3–5 years) | 3–5 |
| `SENIOR` | Senior (6–10 years) | 6–10 |
| `LEAD` | Lead / Principal (11+ years) | 11 or more |

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
