# Spec Homework — Scores and Feedback (Oct 28)

Scored against `RUBRIC.md` (100 points). Each spec was reviewed by a separate AI reviewer with only the brief, the rubric, that person's spec, process notes and commit history. The **build test** is a full "build it from this document only" walk-through (DB, APIs, worker, AI call, screens, payments, deploy) that logs every point where the builder would have to guess.

**Late work is shown but not yet deducted** — see "Timeliness" (coordinator decision pending).

## Summary

| Rank | Name | A Req /20 | B Rules /10 | C Arch /10 | D DB /10 | E API /10 | F Trace /10 | G Build /15 | H Round 2 /10 | I Process /5 | **Total** | Build-test guesses | Timeliness |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Sachin | 20 | 10 | 10 | 10 | 10 | 10 | 12 | 10 | 4 | **96** | 7 | On time |
| 2 | Neeraj | 20 | 10 | 10 | 10 | 10 | 10 | 12 | 10 | 3 | **95** | 6 | R2 late (Oct 27, within extension) |
| 3 | Kakali | 20 | 10 | 10 | 10 | 10 | 7 | 12 | 10 | 3 | **92** | 7 | On time |
| 4 | Vaishali | 20 | 10 | 10 | 10 | 10 | 7 | 12 | 10 | 2 | **91** | 6 | On time |
| 5 | Pranesh | 20 | 10 | 10 | 10 | 10 | 7 | 8 | 10 | 2 | **87** | 12 | On time |
| 6 | Sandip | 20 | 10 | 10 | 10 | 10 | 7 | 8 | 10 | 1 | **86** | 9 | On time |
| 6 | Nikky | 20 | 10 | 10 | 10 | 10 | 7 | 8 | 9 | 2 | **86** | 11 | On time |
| 8 | Madhurima | 20 | 10 | 7 | 10 | 10 | 7 | 8 | 10 | 3 | **85** | 12 | On time |
| 9 | Bilal | 20 | 10 | 7 | 10 | 10 | 7 | 8 | 10 | 2 | **84** | 12 | R1 late (Oct 26); R2 within extension |
| 10 | Pratiksha | 14 | 10 | 10 | 10 | 10 | 4 | 12 | 9 | 3 | **82** | 7 | R1 §5–8 late (Oct 23) |
| 11 | Sovan | 14 | 10 | 7 | 7 | 7 | 7 | 4 | 8 | 1 | **65** | 18 | R1 late (Oct 22) |

**Group average: 86.8 / 100.**

## Patterns across the group

1. **Content quality is high.** 10 of 11 specs scored full marks on requirements, rules, DB and API; most have a single constants table, testable acceptance criteria and failure-path diagrams.
2. **Where points were lost:**
   - **Consistency (F)** — the same rule written in two places drifting apart after edits, especially after Round 2.
   - **Build test (G)** — mostly payment edge cases (Razorpay status mapping, late/out-of-order webhooks, expired checkouts), frontend deployment and missing dev/staging config.
   - **Process (I)** — almost everyone committed whole rounds in a 10–40 minute burst instead of daily, and most `process.md` files have session IDs instead of shareable AI links.
3. **Round 2 was handled well by everyone** — new IDs added, old items marked Deprecated, no renumbering.

## Individual feedback

### Sachin — 96
An outstanding spec: a developer could build almost all of the backend without asking a single question, and contradictions were fixed at the source. The Resume Pro change is handled very well (one `access_until` rule, cancellation retry on erasure).
**Improve:** add a frontend deploy section (hosting, pipeline, env vars, 401 → platform login with return URL); name the logging/metrics/alert destinations, task sizes and IaC tool; link exported AI transcripts, not just session IDs.

### Neeraj — 95
Exceptionally precise: one constants table, exact error bodies and an outage matrix. Resume Pro reached every section with a clean change log.
**Improve:** give reconcile/pro-usage jobs their own queue so the per-minute stale-job check still holds; define payment period fields for verify/webhook, mock-cancel behaviour and existing platform-Pro users at cutover; commit daily as you write.
**Note for coordinator:** `process.md` says other participants' submissions were looked at "at concept level" during reviews — check against the copying rule.

### Kakali — 92
Remarkably precise: single source of constants, exact check order, hand-verified worked examples; Resume Pro carried through every section cleanly.
**Improve:** fix the stale `created` checkout row that blocks re-subscribing for up to 24 h; enforce "one analysis at a time" with a DB unique index; set `last_event_at` on every subscription write and align the webhook diagram with FR-025.

### Vaishali — 91
One of the most build-ready specs: scoring rules, AI contract, quota locking and Razorpay state rules are precise enough to code from.
**Improve:** re-check rules written in two places (payment fallback "1 month" vs 31 days; how cancel works per status); define "live subscription" once (does it include `halted`?); commit across several days and link the AI transcript.

### Pranesh — 87
Very buildable: upload checks, worker leases, entitlement rule and webhook handling are exact, and Round 2 touched every section while keeping all IDs.
**Improve:** when a checkout expires locally, also cancel it at Razorpay (otherwise a late payer is charged with no way to cancel); pick one source for each limit (DB CHECKs vs configurable constants); add infrastructure/deploy details and non-prod auth config.

### Sandip — 86
Unusually precise requirements, thorough failure paths (10 sequence diagrams) and an exemplary Round 2 change.
**Improve:** align the subscription expiry rule across FR-033, SEQ-09 and the index; define what the 30-day report purge removes and whether reports regenerate after a plan change; commit across several days and replace the `<LINK-PLACEHOLDER>` in `process.md`.

### Nikky — 86
Complete and buildable: limits, error codes and DB constraints match everywhere; failure flows and the `pro_until` design show real thought.
**Improve:** after a change request, search for stale lines (out-of-scope still lists "comparing two analyses"; 1–1000 limit vs unlimited Pro); write a subscription state-transition table so an `active` event can't undo a cancellation; commit over several days.

### Madhurima — 85
Very precise requirements and a thorough DB design; Round 2 was a model change with a clean change log.
**Improve:** remove leftovers from the v1.1 tool switch (Prometheus metric names, PDF tool installed only in the worker image); add one config table with every env var per environment; define the expired-checkout and alert-job discovery edge cases.

### Bilal — 84
Exceptionally precise: single fixed-values table, constraint-level DB design, and Resume Pro added without touching any existing ID or the free journey.
**Improve:** say how the API knows the AI circuit breaker is open (shared Redis flag); write the "is this user Pro now" rule as a small truth table; add a Razorpay mapping table and remove v1.2 leftovers (`report.hbs`, the removed admin erasure permission). Commit as you write over several days.

### Pratiksha — 82
Very buildable: single constants table, exact AI contract and a strong Razorpay design.
**Main issue:** in Round 2 a script pasted traceability rows into the wrong tables — it overwrote the FR summary in §3.1 and several NFR definitions in §3.2 (including the NFR-014 alarms), and §7 kept old v0.4 rows.
**Improve:** restore §3.1/§3.2 and move the v0.5 rows into §7; add the skill seed for all 16 roles; add a short infrastructure section.

### Sovan — 65
Strong common rules, DB constraints and error catalogue — precise enough to build from directly.
**Improve:** choose sync or async analysis (the spec describes both, and the queue it mentions is never defined); pick one upload design (presigned vs multipart); define where a user's Pro status is stored and how it expires; gate Pro features in the API. Commit a little each day and include AI conversation links.

## Timeliness (decision needed before final scores)

| Name | Issue |
|---|---|
| Sovan | Round 1 landed Oct 22 (deadline Oct 20) |
| Pratiksha | Round 1 §5–8 landed Oct 23 (deadline Oct 22) |
| Bilal | Round 1 landed Oct 26 (deadline Oct 22); GitHub invite was not accepted until Oct 27 |
| Neeraj | Round 2 landed Oct 27 (deadline Oct 23; extension to Oct 27 18:00 given) |

Rubric rule: commits after the deadline are not counted for that round. Options: apply strictly, apply a small deduction (e.g. −5), or no deduction since extensions were granted.
