# Process evidence — Sandip Kumar Mondal

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-09 | Claude Code (desktop app, Claude Fable 5.1) | `<LINK-PLACEHOLDER: Claude Code session "spec-homework" — exact URL not available from inside the session; to be pasted from the app's share/history view>` | Read the two Notion architecture pages and the homework page, unpacked the company `project-template-v7`, and drafted a first spec before the official brief existed (that draft targeted the Tutor App and was discarded — see Notes). |
| 2026-10-19 | Claude Code (desktop app, Claude Fable 5.1) | same session as above (`<LINK-PLACEHOLDER>`) | Re-read the Notion page comments and `#ai-spec-homework`, read `BRIEF.md` / `TEMPLATE.md` / `RUBRIC.md`, then wrote `sandip/spec.md` for the Resume Optimizer section by section, ran the consistency checks, and committed each section. |

## Notes
Anything worth knowing about how you worked: decisions you changed, things the AI got wrong, what you checked by hand.

### How the brief was interpreted
- The one paragraph in `BRIEF.md` is treated as the only confirmed input. Every number (5 MiB, 10 pages, 3 free analyses/month, six dimensions and their weights, retention periods, timeouts) is an engineering assumption, tagged `[A-nn]` in the spec and collected in §3.3.2 with the reason and who must confirm it. They are deliberately not presented as stakeholder requirements.
- "Free users see only some of the suggestions" became a read-time projection (BR-015): the full result is always stored and gating is applied from the plan claim at response time, so an upgrade reveals the stored result without re-running the AI.
- "Handled under the DPDP Act" became concrete engineering controls (masking before the AI call, retention and purge jobs, erasure flow, audit log, consent versioning) **plus** an explicit list of legal questions (§3.3.6, L-1…L-8) that need authorised review. The spec does not claim compliance.

### Architecture and design decisions, with trade-offs
- **Postgres is the quota source of truth; Redis is a fast path.** A quota check under a row lock in the same transaction as the analysis insert is simple and correct under concurrency; Redis only short-circuits obvious rejections and can be lost at any time (BR-004/005). Trade-off: one extra DB round trip on every start. Accepted because starts are rare (a few per user per month).
- **One in-flight analysis per user**, enforced by a partial unique index rather than a Redis lock, so it survives Redis loss and worker crashes.
- **Text extraction server-side, PII masked before the AI call.** Keeps the raw file out of the third-party processor and bounds what crosses the border (L-1). Trade-off: no OCR, so scanned PDFs are rejected with guidance (A-04).
- **Structured JSON output from the AI + system-side validation (FR-014).** The fact guards (evidence must be a verbatim substring; rewrites may not add numbers or proper nouns absent from the resume; missing skills must come from the role catalogue) are what make "the system must not invent qualifications" testable rather than a prompt wish.
- **Worker in the same NestJS codebase, separate entrypoint.** Avoids a second framework; scales by process count; BullMQ gives retries and stalled-job recovery. Trade-off: the API image carries worker dependencies (pdf parsers, react-pdf).
- **`@react-pdf/renderer` instead of headless Chromium** for the PDF: smaller image, no browser CVEs, deterministic output. Trade-off: less layout freedom.
- **Transactional refund of quota on system-caused failures (BR-007)** so a provider outage never costs the user an analysis; user-caused failures (unreadable file, refusal) are not refunded.
- Model and effort are configuration (`AI_MODEL` default `claude-opus-5-5`, `AI_EFFORT` `medium`); the provider sits behind `AiPort` so it can be swapped by replacing one adapter.

### Failure-path design
Every external dependency has a named failure behaviour in §4.4 and a sequence-diagram branch in §4.3: invalid upload (SEQ-01), quota/concurrency/DB/Redis failures on start (SEQ-02), parse failure, AI timeout, AI rate limit, provider error, malformed response and refusal (SEQ-03), PDF render/storage failure and unauthorised download (SEQ-05), erasure storage/DB failures (SEQ-06), purge lock and object-delete failures (SEQ-07). Degraded modes are stated as a requirement (FR-029) and have chaos tests (TC-050–TC-055).

### Traceability and validation method
- Every requirement, component, table, endpoint, error, business rule, assumption, index, diagram and test case has an ID; §7.1 maps FR/NFR → CMP → TBL → API → TC, and a second table maps every CR to where it is satisfied.
- Checks run on 2026-10-19 with inline shell commands against the assembled file (no checker script committed):
  - defined = referenced, no duplicates, no dangling references for `CR` (40), `FR` (30), `NFR` (18), `CMP` (10), `TBL` (13), `API` (20), `ERR` (34), `BR` (16), `A` (16), `TC` (69), `IDX` (21), `SEQ` (7);
  - section headings 1–8 present in template order;
  - scoring weights 30+20+15+15+10+10 = 100;
  - every error code named in §3/§4 exists in §6.3;
  - no template `<placeholder>` text left (remaining `<…>` occurrences are pattern notation such as `idx_<table>_<cols>`).
- Manual read-through found and fixed three contradictions before the final commit: (1) a resume whose text had been purged after 30 days was still `ready` but could not be analysed — added `details.reason = text_expired` to ERR-019 / FR-008 / API-008 and a defensive branch in SEQ-03; (2) FR-002 counted "used" by analysis status, contradicting the refund rule — now reads the counter; (3) A-01 did not state whether an admin token can call user endpoints — now explicit.

### Things the AI got wrong or that I changed
- The first draft (2026-10-09) specified the Tutor App because the official brief was not published yet; it was discarded entirely when the brief named the Resume Optimizer. Nothing from it was reused except the system-agnostic reasoning behind the company-wide rules in §1 and the conventions in §2, rewritten into the template's `CR-` table format.
- The first version of API-017 contained a half-finished sentence in a validation cell; caught by reading the assembled file.

### Outstanding risks and decisions requiring confirmation
- All `A-01`…`A-16` assumptions, in particular the platform token shape (A-01), the free quota (A-06), the scoring weights (A-09) and the AI provider / cross-border processing (A-10, L-1, L-2).
- Legal review items L-1…L-8 (§3.3.6) are open; the spec's defaults are engineering positions only.
- Hostnames, container platform and managed-service providers are not named beyond region and type; they are deployment details outside the brief.

### Progress and review activity
- 2026-10-19: sections 1–2 committed; section 3 committed; section 4 committed; sections 5, 6, 7–8 committed; consistency checks run; this file committed. All commits on branch `spec/sandip`, pushed to `origin/spec/sandip`.
- 2026-10-19 (builder-readiness review, spec v1.1): read the whole file as the implementer. Gaps that would have been guesses were added as §4.6 (environment variables, rate-limit algorithm), §4.7 (queues, job names, attempts, cron schedule), §4.8 (how the web app obtains the token — BFF route handlers forwarding the platform cookie; screens SCR-001–SCR-004; draft consent notice), §4.9 (AI input/output schema, prompt skeleton, error mapping) and §4.10 (seed catalogue: 24 roles, complete skill rows for two roles, rule that incomplete roles are hidden). All nine Mermaid blocks were parsed with Mermaid 11.4.1 in a browser page served locally; two syntax problems (semicolons inside sequence messages, `FK_UK` key token) were found and fixed before this commit.
- Known remaining guess points for the build test: the skill rows of 22 of the 24 roles are product content (A-16) and ship as data, not code; hostnames and the container platform are deployment details.
- 2026-10-22 (Round 2, spec v1.2): the coordinator's requirement (Resume Pro ₹299/month via Razorpay, unlimited analyses and all suggestions for Pro, history + comparison, cancel with access to period end, free unchanged) was first turned into an impact list against the v1.1 spec, then applied section by section with one commit each: §3 requirements → §4 architecture → §5 data model → §6 API → §7–8 traceability and change log. Decisions: plan now comes from the module's own `subscriptions` table instead of the platform token claim (the brief never guaranteed such a claim); the v1.0 fair-use daily cap for paid users is deprecated because Round 2 says "unlimited"; history stays available to free users (they must stay "exactly as today") and only the comparison is Pro-only; activation is webhook-driven with an idempotent event inbox, never trusted from the browser; billing records are retained pseudonymised on erasure for the statutory period (legal question L-9). Internal plan value renamed `premium` → `pro` (recorded as A-21). No existing ID was renumbered; superseded items are marked Deprecated with their rows kept. Consistency checks and the Mermaid parse were re-run on the final file (results below in the validation notes).
- Round 2 validation (2026-10-22, v1.2): the inline checks were re-run on the final file — defined = referenced, no duplicates, no dangling references for `CR` (40), `FR` (37), `NFR` (21), `CMP` (11), `TBL` (15), `API` (25), `ERR` (40), `BR` (20), `A` (21), `L` (9), `TC` (83), `IDX` (28), `SEQ` (10), `SCR` (5); sections 1–8 in template order; every FR/NFR, table and endpoint present in §7.1; the word `premium` occurs only inside Deprecated/v1.0 notes; all 12 Mermaid blocks parse with Mermaid 11.4.1. Round 2 commits: §3 (`0a47f73`), §4 (`167de4c`), §5 (`a9819a3`), §6 (`517a672`), §7–8 + process (this commit), all on `main`.
