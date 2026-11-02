# Build test log — Madhurima Gorain

Self-run build tests for the improvement round (PLAYBOOK step 4).

**Method (same every run):** a fresh AI session that reads **only** `madhurima/spec.md` (no other file, no chat history) gets the "Build test prompt" from `PLAYBOOK.md` word for word. Every guess it reports is listed here and fixed at its source, then the test is run again. **Goal:** fewer than 5 guesses.

| Run | Date | Spec version | Guesses | Notes |
|---|---|---|---|---|
| 1 | 2026-10-30 | v2.0 | 33 | Baseline before any improvement-round change |

---

## Run 1 — 2026-10-30 (spec v2.0)

Status: **Open** = not fixed yet · **Fixed (vX.Y)** = fixed in that spec version.

| # | Where | Gap found | Planned fix | Status |
|---|---|---|---|---|
| 1 | CR-002, CR-003, FR-001 AC3, 1.7 | Issuer, JWKS URL and login redirect are prod-only; local fakeauth cookie flow not described | Per-env auth config (`RO_AUTH_ISSUER`, `RO_AUTH_JWKS_URL`, `RO_AUTH_AUDIENCE`, login URL); describe how fakeauth sets `sb_access` | Open |
| 2 | CR-011, 6.1, 1.5 | No dev web origin / API base URL; CSP `connect-src` only allows prod API | Per-env table: web origin, API base URL, CSP `connect-src` | Open |
| 3 | 1.8, 6.1, 4.1 | Hosting of `api.` vs `app.surfingbear.com` unclear: Nginx server names, who routes the UI path, TLS certificates | Define Nginx `server_name` blocks, routing per host, certificate source and renewal | Open |
| 4 | API-016/017 vs 1.8 | `/readyz` is at the root but Nginx only forwards `/resume-optimizer/v1/*` to `api` | Add Nginx route for health checks to `api` | Open |
| 5 | 1.8 vs FR-004, CMP-005 | API extracts text, but only the worker image has `poppler-utils` | Install `poppler-utils` in the API image | Open |
| 6 | CR-010, config | No full env var list (Redis, S3 bucket/region/endpoint, Slack webhook, log level, CORS, dev model, frontend API URL) | One config table: name, secret?, required?, value per env | Open |
| 7 | API-018, NFR-011, CMP-011 | Alert job reads "all processes" on port 9090 but has no way to find them; scaled workers can't share one host port | Processes push counters to Redis, or define scrape targets | Open |
| 8 | API-018 vs FR-018, NFR-002, 4.4 | Metric names `ro_retention_deleted_total`, `ro_analysis_duration_seconds` and the JWKS alert are not in API-018 | Add fields to API-018 or remove references; say which process alerts | Open |
| 9 | NFR-002 vs NFR-010 | 10 analyses/s target vs 1,000/hour capacity | Align load numbers | Open |
| 10 | FR-008 AC4 vs 4.7 | `attempts` incremented twice (on take and on recovery) | Increment only when a worker takes the job | Open |
| 11 | NFR-013, `stuckAnalyses` | "Stuck > 15 min" has no reference timestamp; conflicts with retry timing | Define by status + `created_at`/`started_at`; align alert | Open |
| 12 | FR-017 AC4, 4.7 | Erasure request stuck in `processing` is never retried | Sweep resets old `processing` rows to `pending` | Open |
| 13 | FR-017 | Uploads/analyses allowed during an open erasure request | Write endpoints return 409 `ERASURE_IN_PROGRESS` | Open |
| 14 | FR-021 AC4, API-014 | UI can't find the erasure request after reload; no settings route | Add "latest erasure request" endpoint and settings page | Open |
| 15 | FR-021 vs 2.2, NFR-014 | 5 pages in FR-021, 3 in folder tree and accessibility check | Add `pro/`, `compare/` to 2.2 and NFR-014 | Open |
| 16 | CR-017 vs FR-021 | Locale from JWT claim vs `/{locale}/` URL: which wins | URL wins; bare path redirects to JWT locale | Open |
| 17 | CR-021, FR-002 | Consent notice text not provided | Add notice text (en, hi) | Open |
| 18 | 1.5 headers | CSP blocks Next.js inline scripts; who sets headers is unclear | Nonce-based CSP set in Next.js middleware | Open |
| 19 | CR-011 vs API-008 | CORS doesn't allow `Idempotency-Key`; no expose-headers | Add allow and expose headers | Open |
| 20 | CR-012 vs API-022 | 16 KiB limit and strict JSON conflict with Razorpay webhook | Exempt API-022: raw body, 64 KiB cap, lenient decode | Open |
| 21 | 1.4 vs FR-013, API-008 | `details` rule vs QUOTA_EXCEEDED shape | Allow `details` for QUOTA_EXCEEDED as 3 items | Open |
| 22 | FR-007 AC1/AC4 | Order of idempotency check vs quota/resume checks; same key with different body | Check key first; different body → 422 | Open |
| 23 | FR-003 AC8, 5.3 | Duplicate-file index not unique; parallel uploads both stored | Partial UNIQUE index; return existing on conflict | Open |
| 24 | 5.5 vs 2.2 vs 1.7 | Seed described 3 different ways | One mechanism: `000002_seed_job_roles.up/down.sql` | Open |
| 25 | 1.8 vs 2.5, 1.6, NFR-014 | CI list misses lint, coverage gate, i18n check, Playwright/axe, `npm audit` | List every CI job in 1.8 | Open |
| 26 | FR-022 AC1, 5.3 | Parallel subscribe creates two Razorpay subscriptions | Lock or placeholder row before calling Razorpay | Open |
| 27 | FR-022 AC2/AC3, API-019 | Stale `created` / old `pending`/`halted` row blocks a free user | Expire stale rows inline; define outcome | Open |
| 28 | FR-022 AC7, 4.7 billing | Expired-checkout job may cancel a subscription that was just paid | Fetch from Razorpay first; cancel only if still `created` | Open |
| 29 | FR-022 AC5, API-020 | Razorpay may return `authenticated` with no period; breaks CHECK | Set `active` only when Razorpay says `active` with `current_end` | Open |
| 30 | FR-023 AC6, `last_event_at` | "Never moves backwards" rule not precise (order, ties, fetches) | Skip whole event if older; define ties; fetches update `last_event_at` | Open |
| 31 | API-024, FR-027, API-001 | "Latest subscription" unclear when an older row still grants Pro | Return the Pro-granting row first, else newest | Open |
| 32 | TBL-013 vs 4.7, 3.3 | 8-year deletion of TBL-013 not in billing job; start date differs | Add step for TBL-013; name exact date column per table | Open |
| 33 | TBL-012, FR-023 | `subscription.activated` audit trigger, `entity_type`, `user_id` undefined for webhooks | Define trigger, add `subscription` type, `user_id` NULL if unknown | Open |

**Overlap with Saswat's feedback DM:** #5 and #8 (leftovers), #1, #2 and #6 (config table), #7 and #28 (alert job, expired checkout).
