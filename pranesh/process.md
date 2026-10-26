# Process evidence — Pranesh Ghosh

## AI conversations

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session, no public link yet. I will add the shared transcript link here. | Planning the design decisions, then drafting `spec.md` one section at a time (sections 1–8), with one commit per section. |
| 2026-10-16 | Claude Code `claude-api` skill reference, inside the same session | (same session) | Checking the Anthropic Java SDK details used in §4.4–4.6: SDK version, model id, structured JSON output, timeout and retry settings, and that `temperature` is not accepted by the model. |
| 2026-10-26 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session `2967978d-bf23-4fc1-8513-970fcddb6f02`, no public link | Round 2 (Resume Pro): reading the Round 2 post in #ai-spec-homework, `BRIEF.md` and `RUBRIC.md`, listing every place the change touches, deciding the open points (plan source, cancel behaviour, history for free users, Razorpay flow), then updating the spec one section at a time with one commit per section, and an independent consistency review at the end. |

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
