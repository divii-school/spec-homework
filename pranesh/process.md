# Process evidence — Pranesh Ghosh

## AI conversations

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16 | Claude Code desktop app (model Claude Opus 5.5) | Local Claude Code session, no public link yet. I will add the shared transcript link here. | Planning the design decisions, then drafting `spec.md` one section at a time (sections 1–8), with one commit per section. |
| 2026-10-16 | Claude Code `claude-api` skill reference, inside the same session | (same session) | Checking the Anthropic Java SDK details used in §4.4–4.6: SDK version, model id, structured JSON output, timeout and retry settings, and that `temperature` is not accepted by the model. |

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
