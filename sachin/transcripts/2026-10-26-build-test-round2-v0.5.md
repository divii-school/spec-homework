# Build test, round 2 second run (spec v0.5)

Exported transcript, 2026-10-26. Claude Code agent run on its own, reading only `sachin/spec.md`. Tool results (the spec text it read) are left out; each tool call is shown as one line.

## Prompt

```text
You are simulating tomorrow's scored "build test": a Claude Code agent will be asked to build the SurfingBear Resume Optimizer (Spring Boot 3 / Java 21 backend, Next.js frontend, PostgreSQL, Redis, Razorpay) from ONE spec file alone, and every point where the builder has to guess, or where two parts of the spec contradict each other, costs the author points.

Read ONLY this file: sachin/spec.md (about 2,700 lines; read all of it, in chunks). You may also read BRIEF.md and RUBRIC.md for the scoring rules. Do NOT open any other participant's folder in spec-homework (copying scores zero) and do not modify any file. Do not use git.

Pretend you must implement it end to end tomorrow. Walk through it as a builder would: project setup and configuration, the DB migrations (V1..V3) in order, every entity, every endpoint in section 6 (request validation order, status codes, response bodies, error codes), the worker/queue and AI call, the Razorpay subscription flows and webhooks, scheduled jobs, erasure, the frontend pages the spec describes, and the tests named in the traceability matrix.

The spec has already been through two such passes (v0.3 found 15 guesses, v0.4 found 4 guesses + 7 contradictions, all fixed), so look for what remains, especially in the round-2 Resume Pro parts (FR-024 to FR-030, TBL-016 to TBL-019, API-017 to API-022, ERR-021 to ERR-025, flows 4.3.8 to 4.3.11, migration 5.6) and where they touch older sections.

Report, as your final message, a numbered list. For each item:
- type: GUESS (builder must invent something), CONTRADICTION (two places disagree; quote both with section/ID), or MISSING (something referenced but not defined);
- severity: HIGH (a builder would likely build it differently from the author's intent / break a flow), MEDIUM, LOW;
- location (section number and IDs);
- one sentence on what exactly is unclear and the smallest wording fix you'd suggest.
Only report real, specific issues you can point to in the text - no style comments, no "could add more detail" items. If you find fewer than 3 real issues, say so plainly. Put HIGH items first. End with a one-line count by type and severity.
```

- *tool:* Bash `wc -l sachin/spec.md BRIEF.md RUBRIC.md`
- *tool:* Read `BRIEF.md`
- *tool:* Read `RUBRIC.md`
- *tool:* Read `spec.md` lines 1-400
- *tool:* Read `spec.md` lines 1-250
- *tool:* Read `spec.md` lines 250-449
- *tool:* Read `spec.md` lines 450-669
- *tool:* Read `spec.md` lines 670-899
- *tool:* Read `spec.md` lines 899-1118
- *tool:* Read `spec.md` lines 1119-1318
- *tool:* Read `spec.md` lines 1319-1548
- *tool:* Read `spec.md` lines 1549-1748
- *tool:* Read `spec.md` lines 1748-1947
- *tool:* Read `spec.md` lines 1947-2176
- *tool:* Read `spec.md` lines 2176-2405
- *tool:* Read `spec.md` lines 2405-2564
- *tool:* Read `spec.md` lines 2565-2689
- *tool:* Grep `SPRING_PROFILES|profiles\.active|APP_ENV|<env>``
- *tool:* Grep `open subscription|is open|that is open|open \(|open one|open local|open rows`
## Final report

I read all 2,684 lines of spec.md v0.5, plus BRIEF.md and RUBRIC.md, and built it on paper end to end: setup, V1–V3, every entity, API-001 to API-022, worker, Razorpay flows, jobs, erasure, frontend and the tests in section 7. No file was changed, no git was used and no other participant's folder was opened.

**No HIGH issues.** Nothing left would make a builder break a main flow. I found 9 real items, all in the Resume Pro parts or where they touch older sections.

1. **CONTRADICTION · MEDIUM · CR-033 vs FR-024 c7 / API-018 / ERR-024.** CR-033 says "A missing or wrong signature returns 401 `UNAUTHENTICATED` … The same rule (with `RAZORPAY_KEY_SECRET` and the checkout string of FR-024) applies to the checkout signature." API-018 and ERR-024 say a checkout signature mismatch is 400 `PAYMENT_VERIFICATION_FAILED`. A 401 would likely send the browser to re-login.
   - Fix: in CR-033, say only the HMAC and constant-time comparison carry over to the checkout signature, and a mismatch there is 400 `PAYMENT_VERIFICATION_FAILED` (API-018).

2. **CONTRADICTION · MEDIUM · FR-028 c7, 4.3.10 vs FR-030 c1 / subscription view.** FR-028 c7 says an immediate cancel (criterion 3, `AUTHENTICATED` or `PENDING`) responds with the view showing "status `CANCELLED`". But the view's `subscription` is the "current subscription", meaning open, or `CANCELLED` with `access_until > now`. An `AUTHENTICATED` row never paid (`access_until` null), and a `PENDING` row's `access_until` is already past. So after the cancel the row is not current and `subscription` is `null`.
   - Fix: either say API-020 always returns the row it just cancelled in `subscription`, or change FR-028 c7 to say `subscription` is `null` (plan `FREE`) in the criterion 3 case.

3. **GUESS · MEDIUM · FR-030 c3 page states (a)–(e).** Nothing covers status `ACTIVE` with plan `FREE`. That happens:
   - after `subscription.activated` arrives before `charged` (4.3.9 sets `ACTIVE` but not `access_until`);
   - at every renewal, between `access_until` and the `charged` or `pending` event;
   - after a cycle-end cancel, between `access_until` and `subscription.cancelled`.
   
   These fall to (e) "Subscribe", and API-017 then answers 409. The spec itself avoids exactly this in (a).
   - Fix: add `ACTIVE` with plan `FREE` to state (a) (or give it its own state), with no "Subscribe" button.

4. **GUESS · MEDIUM · FR-029 c8 reconciliation; FR-024 c3 / 4.3.8 local expiry.** For a `CREATED` row past `checkout_expires_at`, two rules in the same list both match when Razorpay reports `active` or `authenticated`: "`active` → `ACTIVE`" and "a `CREATED` row past `checkout_expires_at` → `EXPIRED`". API-017 also expires stale `CREATED` rows without asking Razorpay.
   - If such a row becomes `EXPIRED` while Razorpay is charging it, later `charged` events still raise `access_until`, because the GREATEST rule ignores terminal status.
   - Nothing can then stop the charges: FR-028 c1 cancel returns 404 (`EXPIRED` is not current), reconciliation reads only open rows, and FR-019 c7 erasure cancels only `CREATED`, `AUTHENTICATED`, `ACTIVE` or `PENDING`.
   - Fix: "Razorpay's fetched status wins; a row becomes `EXPIRED` by time only when Razorpay still reports `created` or `expired`." Also: "`active` or `charged` on a terminal row inserts a TBL-019 cancellation."

5. **CONTRADICTION · LOW · the word "open".** The Glossary, `uq_subscription_user_id_open`, 5.4 and FR-019 c7 count `CREATED` as open. FR-028 c1 defines open as "(`AUTHENTICATED`, `ACTIVE` or `PENDING`)". API-020's 404 row says "no open subscription and no `CANCELLED` one with access left", which under the first meaning would not give 404 for a `CREATED`-only user. FR-028 c1 says it should.
   - Fix: define "open" once (with `CREATED`) and give FR-028's set its own name, for example "current".

6. **CONTRADICTION · LOW · NFR-013 alert vs FR-029 c8.** NFR-013 fires an alert on "any reconciliation fix". FR-029 c8 counts every changed row in `billing_reconciliation_fixes_total`, including the routine `CREATED` → `EXPIRED` of every abandoned checkout. The alert would fire almost daily.
   - Fix: leave the time-based `CREATED` → `EXPIRED` change out of the counter and the WARN.

7. **GUESS · LOW · 4.3.9 / TBL-018 `outcome`.** For a known subscription where nothing is applied (a stale or terminal skip, or `authenticated` on a non-`CREATED` row), the diagram says "apply the event table, outcome `APPLIED`". The table only defines `IGNORED` for unknown event types.
   - Fix: say whether a skipped event is recorded as `APPLIED` or `IGNORED`. This also sets the `outcome` label of the webhook metric.

8. **GUESS · LOW · CR-027, 4.4 fake guard, Redis keys `ro:<env>:…`.** The prod key check ("refuses to start unless `rzp_live_`") and "a `dev`, `staging` or `prod` task with `BILLING_PROVIDER=fake` refuses to start" both need the environment name, as does `<env>` in the Redis keys. Nothing in 4.7 says where it comes from.
   - Fix: add a row such as `app.env` / `SPRING_PROFILES_ACTIVE` with values `local|dev|staging|prod`.

9. **GUESS · LOW · FR-026 / 2.2 history page Pro filters.** The source and label of the filter options are undefined. API-005 lists only `ACTIVE` resumes and they have no name (file names are not stored), yet analyses of expired resumes stay in history. API-001 lists only active roles, while the `jobRoleCode` filter accepts inactive ones.
   - Fix: one sentence, e.g. "resume options are the distinct `resumeId`s of the loaded history, labelled by upload date; role options come from API-001 plus roles present in history."

Count: CONTRADICTION 4 (2 MEDIUM, 2 LOW) · GUESS 5 (2 MEDIUM, 3 LOW) · MISSING 0 · HIGH 0 — 9 total.
