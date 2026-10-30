# Process evidence — Sandip Kumar Mondal

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-09 | Claude Code (desktop app, Claude Fable 5.1) | **TO ADD BY SANDIP** — one Claude Code desktop session was used for all work on this homework (session id `e4885936-378f-455e-aec2-cf8f03266ef0`); the shareable link is not visible from inside the session and must be copied from the desktop app (session menu → Share / Copy link) and pasted here | Read the two Notion architecture pages and the homework page, unpacked the company `project-template-v7`, and drafted a first spec before the official brief existed (that draft targeted the Tutor App and was discarded — see Notes). |
| 2026-10-19 | Claude Code (same session) | same link as above | Re-read the Notion comments and `#ai-spec-homework`, read `BRIEF.md` / `TEMPLATE.md` / `RUBRIC.md`, wrote `sandip/spec.md` v1.0–v1.1 section by section, ran the consistency checks, committed each section. |
| 2026-10-22 | Claude Code (same session) | same link as above | Round 2: impact analysis of the Resume Pro requirement, v1.2 section by section, then a second Claude agent with no prior context reviewed v1.2 (27 findings) → v1.3. |
| 2026-10-30 | Claude Code (same session) | same link as above | Applied the coordinator's post-scoring feedback → v1.4 (expiry rule alignment, report purge and regeneration rules, process notes). |

## Notes
Anything worth knowing about how you worked: decisions you changed, things the AI got wrong, what you checked by hand.

### How the brief was interpreted
- The one paragraph in `BRIEF.md` is treated as the only confirmed input. Every number (5 MiB, 10 pages, 3 free analyses/month, six dimensions and their weights, retention periods, timeouts) is an engineering assumption, tagged `[A-nn]` in the spec and collected in §3.3.2 with the reason and who must confirm it. They are deliberately not presented as stakeholder requirements.
- "Free users see only some of the suggestions" became a read-time projection (BR-015): the full result is always stored and gating is applied from the plan at response time, so an upgrade reveals the stored result without re-running the AI.
- "Handled under the DPDP Act" became concrete engineering controls (masking before the AI call, retention and purge jobs, erasure flow, audit log, consent versioning) **plus** an explicit list of legal questions (§3.3.6, L-1…L-9) that need authorised review. The spec does not claim compliance.

### Architecture and design decisions, with trade-offs
- **Postgres is the quota source of truth; Redis is a fast path.** A quota check under a row lock in the same transaction as the analysis insert is simple and correct under concurrency; Redis only short-circuits obvious rejections and can be lost at any time (BR-004/005). Trade-off: one extra DB round trip on every start. Accepted because starts are rare.
- **One in-flight analysis per user**, enforced by a partial unique index rather than a Redis lock, so it survives Redis loss and worker crashes.
- **Text extraction server-side, PII masked before the AI call.** Keeps the raw file out of the third-party processor and bounds what crosses the border (L-1). Trade-off: no OCR, so scanned PDFs are rejected with guidance (A-04).
- **Structured JSON output from the AI + system-side validation (FR-014).** The fact guards (evidence must be a verbatim substring; rewrites may not add numbers or proper nouns absent from the resume; missing skills must come from the role catalogue) make "the system must not invent qualifications" testable rather than a prompt wish.
- **Worker in the same NestJS codebase, separate entrypoint.** Avoids a second framework; scales by process count; BullMQ gives retries and stalled-job recovery.
- **`@react-pdf/renderer` instead of headless Chromium** for the PDF: smaller image, no browser CVEs, deterministic output.
- **Transactional refund of quota on system-caused failures (BR-007)** so a provider outage never costs the user an analysis.
- **Round 2:** the plan is resolved from the module's own `subscriptions` table (FR-034, BR-017), not a platform token claim the brief never guaranteed; the v1.0 fair-use daily cap for paid users was deprecated because Round 2 says "unlimited"; history stays available to free users ("exactly as today") and only the comparison is Pro-only; activation is webhook-driven with an idempotent event inbox, never trusted from the browser; billing records are retained pseudonymised on erasure (L-9). Internal plan value renamed `premium` → `pro` (A-21).

### Failure-path design
Every external dependency has a named failure behaviour in §4.4 and a sequence-diagram branch in §4.3: invalid upload (SEQ-01), quota/concurrency/DB/Redis failures on start (SEQ-02), parse failure, AI timeout, rate limit, provider error, malformed response and refusal (SEQ-03), PDF render/storage failure and unauthorised download (SEQ-05), erasure failures (SEQ-06), purge lock and object-delete failures (SEQ-07), checkout and webhook failures (SEQ-08), cancel/expiry (SEQ-09), comparison errors (SEQ-10). Degraded modes are a requirement (FR-029) with chaos tests (TC-050–TC-055, TC-129).

### Traceability and validation method
- Every requirement, component, table, endpoint, error, business rule, assumption, index, diagram, screen and test case has an ID; §7.1 maps FR/NFR → CMP → TBL → API → TC, and a second table maps every CR to where it is satisfied.
- Inline shell checks run on every version against the assembled file (no checker script committed): defined = referenced with no duplicates and no dangling references for every ID family; sections 1–8 in template order; scoring weights sum to 100; every error code named in §3/§4 exists in §6.3; every FR/NFR, table and endpoint appears in §7.1; the word `premium` occurs only inside Deprecated notes; all Mermaid blocks parse with Mermaid 11.4.1 in a locally served browser page.
- v1.1 (2026-10-19): a manual read-through as the builder found and fixed three contradictions (purged text vs `ready` resume; FR-002 counting by status vs the refund rule; admin tokens on user endpoints) and added the sections a builder would otherwise guess (env vars, queues, web token handling, AI contract, seed data).
- v1.3 (2026-10-22): a second Claude Code agent with no prior context reviewed v1.2 against Saswat's five points and rubric §H and returned 27 findings (2 blockers, 13 should-fix, 12 nits), all applied. The blockers were real: a check constraint that would have made the expiry job fail, and an erasure path colliding with the one-live-subscription index. One v1.2 feature (comparison inside the PDF) was withdrawn because the data model allows one report per analysis.
- v1.4 (2026-10-30): coordinator feedback applied — see "Coordinator feedback" below. Checks re-run: all ID families balanced (BR 21, TC 84), 12 Mermaid blocks parse.

### Things the AI got wrong or that I changed
- The first draft (2026-10-09) specified the Tutor App because the official brief was not published yet; it was discarded when the brief named the Resume Optimizer. Only the system-agnostic reasoning behind §1 and §2 was reused, rewritten into the template's `CR-` table format.
- The first version of API-017 contained a half-finished sentence in a validation cell; caught by reading the assembled file.
- v1.2 initially left three places disagreeing on when Pro access ends and whether an `active` subscription can expire; fixed in v1.3 and completed in v1.4 after the coordinator pointed at SEQ-09 and the index.

### Coordinator feedback after scoring (2026-10-28) and what was done
1. *"Align the subscription expiry rule across FR-033, SEQ-09 and the index: does `active` expire too?"* — Yes. v1.3 had already put `active` into FR-033 AC-4 and BR-018 but SEQ-09's `UPDATE` and IDX-25's partial index still listed only `cancelled, past_due`. v1.4 uses the one status set `active, past_due, cancelled` in all four places; an `active` row expires when its period lapses without a renewal webhook.
2. *"Define what the 30-day report purge removes (the row or only the file), and whether a report is regenerated after a plan change."* — Only the PDF object is removed; the row stays as `purged` with `purged_at`, and is deleted only with its analysis at 365 days. New BR-021 defines regeneration: re-render after purge or after a free → Pro upgrade; a Pro-rendered PDF is never downgraded. TBL-009 gains `plan_at_render` and `purged_at`; API-012 returns `regenerated` and `planAtRender`; API-013 has a `purged` response; TC-134 covers it.
3. *"Commit across several days, and replace the `<LINK-PLACEHOLDER>` in `process.md` with real AI links."* — Acknowledged. Commit history: Round 1 was written in one day (2026-10-19) and Round 2 in one day (2026-10-22), each as six to nine section-wise commits; the cadence lesson is recorded here for the next exercise — start on the kickoff day and commit each section the day it is written. The AI link: all work happened in a single Claude Code desktop session whose id is given in the table above; the shareable URL must be copied from the desktop app and pasted into the table (marked **TO ADD BY SANDIP**).

### Outstanding risks and decisions requiring confirmation
- All `A-01`…`A-22` assumptions, in particular the platform token shape (A-01), the free quota (A-06), the scoring weights (A-09), the AI provider / cross-border processing (A-10, L-1, L-2), and the billing assumptions (A-17–A-20).
- Legal review items L-1…L-9 (§3.3.6) are open; the spec's defaults are engineering positions only.
- Hostnames, container platform and managed-service providers are not named beyond region and type.

### Progress log
- 2026-10-19: v1.0 sections 1–8 in seven commits; v1.1 builder-readiness additions; Mermaid parse check added. All on `spec/sandip`, later fast-forwarded to `main`.
- 2026-10-22: Round 2 v1.2 in five section-wise commits; v1.3 after the independent review (one commit, 27 findings). On `main`.
- 2026-10-30: v1.4 coordinator feedback (one commit). On `main`.
