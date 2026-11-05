# Build test log — Madhurima Gorain

Self-run build tests for the improvement round (PLAYBOOK step 4).

**Method (same every run):** a fresh AI session that reads **only** `madhurima/spec.md` (no other file, no chat history) gets the "Build test prompt" from `PLAYBOOK.md` word for word. Every guess it reports is listed here and fixed at its source, then the test is run again. **Goal:** fewer than 5 guesses.

| Run | Date | Spec version | Guesses | Notes |
|---|---|---|---|---|
| 1 | 2026-10-30 | v2.0 | 33 | Baseline before any improvement-round change |
| 2 | 2026-11-04 | v2.7 | 35 | All 33 of run 1 fixed (v2.1–v2.7). Run 2 read deeper; about 10 findings were side effects of the v2.3 payment rules. All fixed in v2.8 |
| 3 | 2026-11-04 | v2.8 | 24 | Deeper again: mostly deployment and operations details (VM files, migrations network, Nginx, local payments). All fixed in v2.9 |

---

## Run 1 — 2026-10-30 (spec v2.0)

Status: **Open** = not fixed yet · **Fixed (vX.Y)** = fixed in that spec version.

| # | Where | Gap found | Planned fix | Status |
|---|---|---|---|---|
| 1 | CR-002, CR-003, FR-001 AC3, 1.7 | Issuer, JWKS URL and login redirect are prod-only; local fakeauth cookie flow not described | Per-env auth config (`RO_AUTH_ISSUER`, `RO_AUTH_JWKS_URL`, `RO_AUTH_AUDIENCE`, login URL); describe how fakeauth sets `sb_access` | Fixed (v2.1) |
| 2 | CR-011, 6.1, 1.5 | No dev web origin / API base URL; CSP `connect-src` only allows prod API | Per-env table: web origin, API base URL, CSP `connect-src` | Fixed (v2.1) |
| 3 | 1.8, 6.1, 4.1 | Hosting of `api.` vs `app.surfingbear.com` unclear: Nginx server names, who routes the UI path, TLS certificates | Define Nginx `server_name` blocks, routing per host, certificate source and renewal | Fixed (v2.7) — 1.8 "Hosting and routing" + A-15 |
| 4 | API-016/017 vs 1.8 | `/readyz` is at the root but Nginx only forwards `/resume-optimizer/v1/*` to `api` | Add Nginx route for health checks to `api` | Fixed (v2.7) — health checks private on port 8080, not routed publicly |
| 5 | 1.8 vs FR-004, CMP-005 | API extracts text, but only the worker image has `poppler-utils` | Install `poppler-utils` in the API image | Fixed (v2.1) |
| 6 | CR-010, config | No full env var list (Redis, S3 bucket/region/endpoint, Slack webhook, log level, CORS, dev model, frontend API URL) | One config table: name, secret?, required?, value per env | Fixed (v2.1) |
| 7 | API-018, NFR-011, CMP-011 | Alert job reads "all processes" on port 9090 but has no way to find them; scaled workers can't share one host port | Processes push counters to Redis, or define scrape targets | Fixed (v2.2) — every process publishes its snapshot to `ro:metrics:*` (TTL 90 s); alert job uses `SCAN` (4.7 "Alert check") |
| 8 | API-018 vs FR-018, NFR-002, 4.4 | Metric names `ro_retention_deleted_total`, `ro_analysis_duration_seconds` and the JWKS alert are not in API-018 | Add fields to API-018 or remove references; say which process alerts | Fixed (v2.1) |
| 9 | NFR-002 vs NFR-010 | 10 analyses/s target vs 1,000/hour capacity | Align load numbers | Fixed (v2.6) — NFR-002 uses the NFR-010 load |
| 10 | FR-008 AC4 vs 4.7 | `attempts` incremented twice (on take and on recovery) | Increment only when a worker takes the job | Fixed (v2.6) — only the worker take increments `attempts` |
| 11 | NFR-013, `stuckAnalyses` | "Stuck > 15 min" has no reference timestamp; conflicts with retry timing | Define by status + `created_at`/`started_at`; align alert | Fixed (v2.6) — stuck = `created_at` / `started_at` older than 15 min |
| 12 | FR-017 AC4, 4.7 | Erasure request stuck in `processing` is never retried | Sweep resets old `processing` rows to `pending` | Fixed (v2.4) — sweep resets `processing` older than 30 min (`started_at`) |
| 13 | FR-017 | Uploads/analyses allowed during an open erasure request | Write endpoints return 409 `ERASURE_IN_PROGRESS` | Fixed (v2.4) — API-003/004/008 → 409 `ERASURE_IN_PROGRESS` |
| 14 | FR-021 AC4, API-014 | UI can't find the erasure request after reload; no settings route | Add "latest erasure request" endpoint and settings page | Fixed (v2.4) — API-025 latest request + settings page |
| 15 | FR-021 vs 2.2, NFR-014 | 5 pages in FR-021, 3 in folder tree and accessibility check | Add `pro/`, `compare/` to 2.2 and NFR-014 | Fixed (v2.4) — 2.2 and NFR-014 list all 6 pages |
| 16 | CR-017 vs FR-021 | Locale from JWT claim vs `/{locale}/` URL: which wins | URL wins; bare path redirects to JWT locale | Fixed (v2.7) — URL wins; bare path redirects |
| 17 | CR-021, FR-002 | Consent notice text not provided | Add notice text (en, hi) | Fixed (v2.7) — notice text in 3.5 |
| 18 | 1.5 headers | CSP blocks Next.js inline scripts; who sets headers is unclear | Nonce-based CSP set in Next.js middleware | Fixed (v2.7) — nonce CSP in Next.js middleware |
| 19 | CR-011 vs API-008 | CORS doesn't allow `Idempotency-Key`; no expose-headers | Add allow and expose headers | Fixed (v2.5) — CR-011 allows `Idempotency-Key`, exposes `X-Request-Id`, `Retry-After` |
| 20 | CR-012 vs API-022 | 16 KiB limit and strict JSON conflict with Razorpay webhook | Exempt API-022: raw body, 64 KiB cap, lenient decode | Fixed (v2.5) — CR-012 exception for API-022 |
| 21 | 1.4 vs FR-013, API-008 | `details` rule vs QUOTA_EXCEEDED shape | Allow `details` for QUOTA_EXCEEDED as 3 items | Fixed (v2.5) — one `details` shape in 1.4 |
| 22 | FR-007 AC1/AC4 | Order of idempotency check vs quota/resume checks; same key with different body | Check key first; different body → 422 | Fixed (v2.5) — key checked first; different body → ERR-030 |
| 23 | FR-003 AC8, 5.3 | Duplicate-file index not unique; parallel uploads both stored | Partial UNIQUE index; return existing on conflict | Fixed (v2.5) — UNIQUE partial index; loser deletes its S3 object |
| 24 | 5.5 vs 2.2 vs 1.7 | Seed described 3 different ways | One mechanism: `000002_seed_job_roles.up/down.sql` | Fixed (v2.7) — only migration `000002_seed_job_roles` |
| 25 | 1.8 vs 2.5, 1.6, NFR-014 | CI list misses lint, coverage gate, i18n check, Playwright/axe, `npm audit` | List every CI job in 1.8 | Fixed (v2.7) — full CI list in 1.8 |
| 26 | FR-022 AC1, 5.3 | Parallel subscribe creates two Razorpay subscriptions | Lock or placeholder row before calling Razorpay | Fixed (v2.3) — 4.7 "Subscribe steps": per-user Redis lock; second click returns the first row |
| 27 | FR-022 AC2/AC3, API-019 | Stale `created` / old `pending`/`halted` row blocks a free user | Expire stale rows inline; define outcome | Fixed (v2.3) — 4.7 "Subscribe steps" step 4: stale rows fetched, then expired or cancelled |
| 28 | FR-022 AC7, 4.7 billing | Expired-checkout job may cancel a subscription that was just paid | Fetch from Razorpay first; cancel only if still `created` | Fixed (v2.2) — 4.7 "Expired checkout": fetch first, conditional updates, payment wins over expiry |
| 29 | FR-022 AC5, API-020 | Razorpay may return `authenticated` with no period; breaks CHECK | Set `active` only when Razorpay says `active` with `current_end` | Fixed (v2.3) — 4.7 "Status mapping": `active` only with `current_end`; `authenticated` stays `created` |
| 30 | FR-023 AC6, `last_event_at` | "Never moves backwards" rule not precise (order, ties, fetches) | Skip whole event if older; define ties; fetches update `last_event_at` | Fixed (v2.3) — 4.7 "Event order rule": older events skipped whole; ties; fetches set `last_event_at` |
| 31 | API-024, FR-027, API-001 | "Latest subscription" unclear when an older row still grants Pro | Return the Pro-granting row first, else newest | Fixed (v2.6) — Pro-granting row first, else newest |
| 32 | TBL-013 vs 4.7, 3.3 | 8-year deletion of TBL-013 not in billing job; start date differs | Add step for TBL-013; name exact date column per table | Fixed (v2.6) — billing job step (d); dates named per table |
| 33 | TBL-012, FR-023 | `subscription.activated` audit trigger, `entity_type`, `user_id` undefined for webhooks | Define trigger, add `subscription` type, `user_id` NULL if unknown | Fixed (v2.6) — subscription audit events defined |

**Overlap with Saswat's feedback DM:** #5 and #8 (leftovers), #1, #2 and #6 (config table), #7 and #28 (alert job, expired checkout).

---

## Run 2 — 2026-11-04 (spec v2.7)

Same method and prompt as run 1. All items fixed in **v2.8** (change log row lists each fix).

| # | Gap found | Status |
|---|---|---|
| 1 | 1.2 vs FR-012: free-plan suggestion rule | Fixed (v2.8) |
| 2 | FR-007 vs API-008: repeat-key response | Fixed (v2.8) |
| 3 | FR-007: check order incl. erasure and validation | Fixed (v2.8) |
| 4 | API-002 vs 1.5: Cache-Control | Fixed (v2.8) |
| 5 | 6.1: which rate-limit bucket in headers; CORS expose | Fixed (v2.8) |
| 6 | CR-012: oversized multipart error code | Fixed (v2.8) |
| 7 | NFR-005: uptime monitor vs private /readyz | Fixed (v2.8) |
| 8 | 1.7: worker needs RO_RAZORPAY_KEY_ID | Fixed (v2.8) |
| 9 | CR-010 vs 1.7: S3 key secrecy | Fixed (v2.8) |
| 10 | 3.3 vs TBL-014: billing event retention | Fixed (v2.8) |
| 11 | Expired checkout vs Status mapping | Fixed (v2.8) |
| 12 | TBL-013 last_event_at note | Fixed (v2.8) |
| 13 | Who may revive an `expired` row | Fixed (v2.8) |
| 14 | Unique index vs authenticated / late payment | Fixed (v2.8) |
| 15 | FR-023: status back to active after retry | Fixed (v2.8) |
| 16 | Lock release wording | Fixed (v2.8) |
| 17 | Flow 4: erasure attempts counted twice | Fixed (v2.8) |
| 18 | 4.6: phone regex matches date ranges | Fixed (v2.8) |
| 19 | 4.6: full system prompt missing | Fixed (v2.8) |
| 20 | NFR-012: evaluation set location | Fixed (v2.8) |
| 21 | Alert rules spread over the spec | Fixed (v2.8) |
| 22 | p95 since start has no window | Fixed (v2.8) |
| 23 | Cookie domain across hosts | Fixed (v2.8) |
| 24 | Locale middleware JWT check | Fixed (v2.8) |
| 25 | Frontend idempotency key per click | Fixed (v2.8) |
| 26 | Pro page after 60 s polling | Fixed (v2.8) |
| 27 | PDF language for hi users | Fixed (v2.8) |
| 28 | UI copy, Hindi text, design | Fixed (v2.8) |
| 29 | CI e2e without Razorpay stub or Pro user | Fixed (v2.8) |
| 30 | Server compose file, dev data, Nginx path | Fixed (v2.8) |
| 31 | Where migrations run | Fixed (v2.8) |
| 32 | No infrastructure as code | Fixed (v2.8) |
| 33 | search_path / schema_migrations | Fixed (v2.8) |
| 34 | NFR-015 `db` package | Fixed (v2.8) |
| 35 | Matrix: NFR-010 autoscaling | Fixed (v2.8) |

---

## Run 3 — 2026-11-04 (spec v2.8)

Same method and prompt. All items fixed in **v2.9**.

| # | Gap found | Status |
|---|---|---|
| 1 | Header version vs change log | Fixed (v2.9) |
| 2 | Cookie name fixed vs per-environment | Fixed (v2.9) |
| 3 | Files on the VM / what deploy copies | Fixed (v2.9) |
| 4 | Local payments: fake vs real test mode | Fixed (v2.9) |
| 5 | Migrate container network on dev | Fixed (v2.9) |
| 6 | Local schema, migrations, MinIO bucket | Fixed (v2.9) |
| 7 | Terraform: who runs it, state, schema | Fixed (v2.9) |
| 8 | Nginx config deploy and upstreams | Fixed (v2.9) |
| 9 | Nginx OSS has no active health checks | Fixed (v2.9) |
| 10 | Certificate for internal `ro-web` host | Fixed (v2.9) |
| 11 | Routing misses paths CR-017 redirects | Fixed (v2.9) |
| 12 | ghcr.io login on VMs | Fixed (v2.9) |
| 13 | Health checks for web and worker | Fixed (v2.9) |
| 14 | ElastiCache has no AOF | Fixed (v2.9) |
| 15 | `processing → queued` not allowed | Fixed (v2.9) |
| 16 | Text-matching normalisation undefined | Fixed (v2.9) |
| 17 | Token counting method | Fixed (v2.9) |
| 18 | Retention check job and storage | Fixed (v2.9) |
| 19 | Alert "cleared" state | Fixed (v2.9) |
| 20 | `users` has no hash column | Fixed (v2.9) |
| 21 | Integer scores vs average | Fixed (v2.9) |
| 22 | Admin stats have no UI | Fixed (v2.9) |
| 23 | S3 delete failure on API-007 | Fixed (v2.9) |
| 24 | API-001/024 vs FR-027 row choice | Fixed (v2.9) |
