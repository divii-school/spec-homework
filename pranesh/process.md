# Process evidence — Pranesh Ghosh

## AI conversations

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session, no public link yet. I will add the shared transcript link here. | Planning the design decisions, then drafting `spec.md` one section at a time (sections 1–8), with one commit per section. |
| 2026-10-16 | Claude Code `claude-api` skill reference, inside the same session | (same session) | Checking the Anthropic Java SDK details used in §4.4–4.6: SDK version, model id, structured JSON output, timeout and retry settings, and that `temperature` is not accepted by the model. |
| 2026-10-26 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session `2967978d-bf23-4fc1-8513-970fcddb6f02`, no public link | Round 2 (Resume Pro): reading the Round 2 post in #ai-spec-homework, `BRIEF.md` and `RUBRIC.md`, listing every place the change touches, deciding the open points (plan source, cancel behaviour, history for free users, Razorpay flow), then updating the spec one section at a time with one commit per section, and an independent consistency review at the end. |
| 2026-10-29 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session `1d6f08e7-aaf2-4236-a67f-0a83d1c33786`, transcript to be added | Improvement-round prep (no commits; the window opened Oct 30): reading the Oct 28 scores and review DM, a first build test (35 gaps) and consistency check (38 findings) in fresh agents, a Razorpay fact check against razorpay.com/docs, and a six-day plan with one scheduled session per day. |
| 2026-10-30 | Claude Code desktop app, scheduled session (Day 1) | Transcript to be added | Payments (review point 1): expired checkouts cancelled at Razorpay, `PENDING_CREATE` row before the Razorpay call, one webhook/sync transition table, 120 h renewal grace. |
| 2026-10-31 | Claude Code desktop app, scheduled session (Day 2) | Transcript to be added | One source per value (review point 2): every repeated number moved into §3.4 and referenced by name; derived time limits; Anthropic Java SDK retry and timeout checked. |
| 2026-11-02 | Claude Code desktop app, scheduled sessions (Days 3 and 4) and one status check | Transcripts to be added | Auth and per-environment configuration, then Terraform infrastructure, deployment and observability (review point 3); a status build test (24 gaps). |
| 2026-11-03 | Claude Code desktop app, scheduled session (Day 5) | Transcript to be added | The 24 status gaps, traceability in both directions, build test #1 (17 gaps) and a consistency check, all fixed. |
| 2026-11-04 | Claude Code desktop app, scheduled session (Day 6) | Transcript to be added | Close-out: build tests until two runs in a row found 3 or fewer guesses, consistency check, v1.2 change-log row, this file. |

## Notes

**Decisions I made where the brief was silent** (each is written down with its reason in the spec):

- No file storage: the uploaded file is parsed in memory and thrown away. Only the redacted text is stored (CR-025, A-09). This removes an object store and a breach surface, which fits the DPDP Act.
- PostgreSQL is also the job queue (row leases with `FOR UPDATE SKIP LOCKED`), so there is no message broker. Redis is used only for rate limiting.
- The AI does not compute the overall score. It returns six dimension scores, and the backend computes the weighted total. The AI only says which catalogue skills are present; the backend works out the missing ones. This keeps the numbers consistent and testable.
- Quota is reserved when an analysis is created and refunded if it fails, all inside one row-locked transaction.

**Things I checked by hand and corrected:**

- An example response (API-009) had an overall score that did not match its own dimension scores. I fixed it to 73 and added the calculation.
- The fields of the history list item in FR-016 did not match the API's `AnalysisSummary` shape. I aligned FR-016 with it.
- Two component IDs were left as "reserved" placeholders. I replaced them with real components: the rate limiter and the error handler.
- The worker time limits did not fit together at first: two AI attempts plus a retry for invalid output could run longer than the lease. I set them as `PROCESSING_DEADLINE` 300 s and `LEASE_DURATION` 330 s, and derived the 26-minute "never stuck" bound from them.

## Round 2 (v1.1, Resume Pro)

**How I approached it:** first I listed every section the change touches (requirements, common rules, architecture, DB, API, traceability, change log), then edited them in that order with one commit per section, so the history shows each part of the change. Nothing was renumbered; retired items are marked **Deprecated**.

**Decisions where the Round 2 text was silent** (each is in the spec with its reason):

- Round 1 had a `PREMIUM` plan read from the Platform token. Since the Module now sells Resume Pro itself, the plan comes from the Module's own `subscriptions` table; `PREMIUM`, its 50-a-month cap, A-02, A-06 and OOS-02 are deprecated (A-11).
- Pro access is decided only by `paid_through` (the end of the last paid cycle), so "access continues until the end of the paid month" holds whatever order Razorpay's webhooks arrive in. A 24-hour grace covers late renewal webhooks only (FR-025).
- Free users already had the history list in Round 1, and item 5 says they must stay exactly as they are, so the list stays for everyone and the new comparison is Pro-only (A-13).
- "Unlimited" means no monthly cap; the per-hour rate limit and one-analysis-at-a-time rule still apply because they protect the AI budget and queue (A-12).
- Activation has three paths that end in the same state: the browser confirmation (API-020), the webhook (API-022) and a 10-minute reconciliation job (FR-032).
- Billing records are kept 8 years for Indian accounting law and are not removed by erasure (CR-042).

**Things the review found and I corrected** (an independent read of the whole spec after the edits, plus a script that checks every cited ID exists and is in the traceability matrix):

- The Pro rule was written three ways (with and without the renewal grace). It is now one definition, `access_until`, used in the glossary, CR-006, FR-001 and FR-025.
- A delayed `subscription.pending` webhook could undo a renewal that the reconciliation job had already repaired. Now every sync moves `last_provider_event_at` forward, so that event counts as stale.
- The 24-hour renewal grace was too short for e-mandate debits, which confirm T+1 to T+3 working days later. Raised to 72 hours.
- Cancel held a row lock during the Razorpay call, against the spec's own "no lock during HTTP" rule. Razorpay is now called first.
- Unpaid, expired checkouts would have been kept 8 years. They are now deleted 7 days after they end.
- The reconciliation job could keep picking the same rows. Every attempt now bumps `updated_at`.
- Missing transitions were added to the state machine. Message keys were added for new UI texts. The webhook body-size check now runs before the signature. Missed items were added to the change log (FR-012, CR-023, CR-030, CMP-001/016/017/018).
- One table row had been merged by an edit, and three Round 1 regexes broke their tables; all fixed.

## Improvement round (v1.2, 2026-10-30 – 2026-11-04)

**Starting point:** score 87 on 2026-10-28. The review DM named three gaps: (1) an expired checkout could still be charged, (2) limits were defined in more than one place, (3) configuration was given for prod only and the infrastructure was not defined. The build test counted 12 guesses (G 8/15) and process evidence was thin (I 2/5).

**How I worked.** On Oct 29 I planned the round with Claude Code: a first strict build test and a consistency check in fresh agents, a Razorpay fact check, and a written plan with one block per day. Each day a scheduled Claude Code session did that day's block in small commits (each message names the findings it fixes), ran the consistency scripts after every change, and logged the day (commits, findings fixed, facts checked, what was left). After the Oct 30 demo session (Sachin, Kakali) I added two rules: verify after every AI step (re-read the edited section and everything that references it, then run the scripts), and trace every addition through every section it touches before committing.

| Day | Date | Block | Commits |
|---|---|---|---|
| 1 | Oct 30 | Payments (review point 1): expired checkouts synced and cancelled at Razorpay, `PENDING_CREATE` row before the Razorpay call, one transition table for webhooks and sync, freshness from Razorpay timestamps only, renewal grace 72 h → 120 h | 2 |
| 2 | Oct 31 | One source per value (review point 2): every repeated number into §3.4, values enforced by a DB `CHECK` made fixed constants, derived time limits, error messages with `{detail}` placeholders | 8 |
| 3 | Nov 2 | Auth and per-environment configuration (review point 3): Platform auth per environment, runtime frontend configuration (no `NEXT_PUBLIC_*`), database bootstrap and users, a complete configuration table per environment | 6 |
| 4 | Nov 2 | Infrastructure (review point 3): Terraform layout and resources, deploy procedure, metrics pipeline on Fargate, alert routing, AI cost constants | 6 |
| 5 | Nov 3 | The 24 gaps of a status build test, traceability in both directions, build test #1 and a consistency check, all fixed | 7 |
| 6 | Nov 4 | Build tests until two runs in a row found 3 or fewer guesses, consistency check, change log, this section | see `git log` |

(Nov 1 has no commits: the app was closed, and the Day 3 block ran on Nov 2.)

**Build tests** (fresh agent, given only `spec.md`, the PLAYBOOK prompt word for word):

| Run | Date | Spec at | Guesses | Main themes |
|---|---|---|---|---|
| Official | Oct 27 | round 2 | 12 | scored by the coordinators |
| Prep (strict) | Oct 29 | `934c692` | 35 | payments, repeated values, per-environment config, infrastructure |
| Status | Nov 2 | `55972ca` | 24 | about a third were gaps created by Days 1–4 text that was not followed through |
| #1 | Nov 3 | `ef222f1` | 17 | dev token, cancel dialog, upload count race, bring-up order, legal texts |
| #2 | Nov 4 | `413e58c` | 14 | infrastructure ownership, lost renewal-failure webhooks, repeated cancel, frontend mechanics |
| #3 | Nov 4 | `cb056dd` | 16 | filter-level errors, scheduler locks, shutdown release vs queue age, export fields, Grafana sharing |
| #4 and #5 (parallel, same commit) | Nov 4 | `d2da78c` | 15 and 18 (26 distinct) | CI job list, Grafana token rotation, skills snapshot, parse queue timeout, JWKS outage, bootstrap mount; 3 items came from that morning's own fixes |
| #6 and #7 (parallel, same commit) | Nov 4 | `a561fc7` | 12 and 11 | test accounts outside `local`, e2e environment, tag-deploy digests, base-URL placeholders, one item #3 found that its fix batch had missed |
| #8 and #9 (parallel, same commit) | Nov 4 | `30e8a44` | 14 and 10 | checkout reuse edge, lapsed subscription display, multipart temp files, enum binding, load-test rate limits |
| #10 and #11 (parallel, same commit) | Nov 4 | `58d63a3` | 14 and 14 | webhook answers that would disable Razorpay delivery, JWKS refetch, worker slot counting, frontend alerts and logging, lapsed subscription edge cases |
| #12 and #13 (parallel, same commit) | Nov 4 | `22909ca` | 6 and 11 | one diagram not updated with a rule, Spring media-type errors, release history, small races |
| #14 and #15 (parallel, same commit) | Nov 4 | `abaea8e` | 12 and 7 | erasure vs concurrent writes, AI escaping owner, load-test accounts, cancel during renewal grace |
| #16 and #17 (parallel, same commit) | Nov 4 | `b6bcbf9` | 10 and 9 | outage alerts, test reachability with the stubs, a wrong Flyway version from v1.0 |
| #18 and #19 (parallel, same commit) | Nov 4 | `edd2ba7` | 10 and 9 | mostly gaps in that day's own additions (eval harness, load-test mechanics), bootstrap privileges on RDS, idempotency races |
| #20 and #21 (parallel, same commit) | Nov 4 | `bc8eb25` | 8 and 9 | text normalisation details, deploy digests after a rollback, start-up plan check, DPDP correction rights |
| #22 and #23 (parallel, same commit) | Nov 4 | `0fa60da` | 12 and 7 | IAM details of the bootstrap task, security filter order, load-test account setup, webhook reverse-order promise |
| #24 and #25 (parallel), consistency check | Nov 4 | `8a53b65` | still running when this session hit its usage limit; not yet recorded | |

**Result against the goal:** the target was two runs in a row with 3 or fewer guesses. It was not reached. Counts fell from 35 to between 6 and 12 per run, then stayed there: each fresh agent found different, narrower edge cases in a 3,500-line spec, and some fixes created new gaps that the next run caught. Every finding listed above was fixed at its source.

Every finding was fixed at its source (requirement or constant first, then architecture, DB, API, traceability), and each run’s list is kept in my working notes with how each item was fixed.

**Consistency checks** (PLAYBOOK prompt in a fresh agent, plus small scripts written with the AI that check every ID is defined, every error code is listed, §7 covers every FR and NFR in both directions, tables have the right number of cells and Mermaid sequence messages have no `;`): 38 findings on Oct 29, 11 on Nov 3, 11 on Nov 4, all fixed; the scripts are clean on the final commit.

**Facts I checked against official documentation** (not from memory):

- Razorpay (razorpay.com/docs): subscription states; the cancel API and its six `400` errors ("in its final cycle", "another subscription operation is in progress"; `cancel_at_cycle_end` must be a boolean); fetch all subscriptions (`count` max 100, `skip`, `notes`); invoices of a subscription (paging, no documented order); the ten subscription webhook events including `authenticated`, `paused` and `resumed`; webhook delivery (at least once, unordered, retried for 24 hours); eMandate later debits take up to 2 working days, which set the 120 h grace.
- Anthropic: Java SDK retries (2 by default; connection errors, 408, 409, 429, 5xx), timeout settings; pricing of Claude Opus 5.5 (US$ 4 / 20 per million input / output tokens).
- Next.js: `NEXT_PUBLIC_*` is inlined at build, runtime values read after `connection()`; `output: "standalone"` does not copy `public/` and `.next/static`; CSP nonce from middleware.
- Spring: Spring Boot 3.5 manages Flyway 11.7.2 and `flyway-database-postgresql` (the v1.0 spec said Flyway 10.x; caught by build test #16, fixed); Tomcat `max-swallow-size` defaults to 2 MB; `spring.flyway.user` gives Flyway its own connection; resource-server JWT defaults (60 s clock skew, 5-minute JWKS cache); Logback event metrics and common metric tags.
- PostgreSQL 16: `CREATE SCHEMA … AUTHORIZATION` needs `SET ROLE` to the owner, and `createrole_self_grant` is empty by default (so the RDS master user must grant itself the role).
- AWS and Terraform: ECS `stopTimeout` (default 30 s, max 120 s) and connection draining; ALB deregistration delay (default 300 s); Amazon Managed Prometheus collectors (EKS only, so ECS needs an ADOT sidecar); Managed Grafana contact points; ECR lifecycle rule priority; Terraform S3 backend `use_lockfile`; `aws_ecs_service` `wait_for_steady_state` and circuit breaker; OpenTelemetry Collector `${env:VAR}` expansion; GitHub Actions sets `GITHUB_REF_TYPE` to `branch` or `tag`.

**What the AI got wrong, and how it was caught:**

- The Round 2 text claimed e-mandate debits confirm "T+1 to T+3 working days" and set a 72 h grace. That figure is not in Razorpay's docs; the docs say up to 2 working days, and a weekend plus a holiday exceeds 72 h. Caught by the Oct 29 fact check; grace now 120 h.
- The AI first wrote rules that ended an expired checkout locally without asking Razorpay, which is exactly the review's point 1. Caught by the build test; `closeCheckout` now syncs and cancels at Razorpay first.
- New sections added on Days 1–4 left their own inputs undefined (a new status missing from a shared type, Terraform resources without the roles that create them). Caught by the Nov 2 status build test: about a third of its 24 gaps were self-inflicted. Since then every addition is traced through every section before committing.
- Day 1 was one large commit instead of several small ones; corrected from Day 2 on.
- Tooling mistakes: the shell stripped backslashes from two `psql` commands (`\gexec`, `\c`), a `sed` rewrite mangled a helper script, and an edit helper silently stopped matching after line endings changed. All were caught by re-reading and by a rule that every scripted edit must match its anchor exactly once or fail.
- A draft fix contained a garbled error code; caught on review before it was applied.
- On Nov 4 the AI cited a test class (`BillingIT`) that does not exist in the spec; caught by a search before committing and replaced with the existing `ReconciliationJobIT`.
- On Nov 4 several fixes created new gaps: the new graceful-shutdown rule was not carried into the state diagram and the TBL-003 note, moving the Grafana contact points left the CI-role reason and the first apply out of step, and a new screen state for a lapsed subscription offered a button another check refused. The next build test caught each one. I then kept every fix as small as possible and searched for every place the changed rule is mentioned before committing.
- Build test #3 listed 16 items and my fix batch for it skipped one (a sync with no Razorpay timestamp). Build test #6 found it again; it is now fixed.
- Since v1.0 the spec named Flyway 10.x, a version the AI wrote from memory. Spring Boot 3.5 manages 11.7.2 (and needs `flyway-database-postgresql`). Build test #16 questioned it, and the Spring Boot dependency-versions page confirmed it.
- ALB health-check defaults came from the model's memory; the spec now sets explicit values instead of relying on defaults.

**Transcripts:** the session transcripts are being exported and reviewed by me; links will be added to the table at the top of this file.
