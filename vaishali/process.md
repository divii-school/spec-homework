# Process evidence — Vaishali Naruka

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-20 | Claude Code (CLI on Windows, Claude Opus 5.5) | Claude Code CLI sessions have no shareable link. Session ID: `dfc099ad-cf47-4fa2-b1b2-279e9db4f7ea` (transcript can be exported on request) | Read `#ai-spec-homework` on Slack, the Notion homework page and its comments, and `BRIEF.md` / `RUBRIC.md` / `TEMPLATE.md`. Wrote down the open design decisions first, then drafted `spec.md` one section at a time with a commit after each (v0.1–v0.6), ran an ID cross-check script, and ran a simulated build test with a separate review agent that saw only `spec.md`; fixed what it found (v1.0). |

## Notes

### Inputs I used
- `BRIEF.md` (the product paragraph is the only product input), `RUBRIC.md`, `TEMPLATE.md`.
- Slack `#ai-spec-homework`: kickoff message (Oct 14), the Oct 16 update (backend stack = own main stack; Group A deadlines unchanged), Saswat's Oct 12 preview.
- Notion page "TL | AI at Work — Architecture Spec Homework" and its comment thread (build test and revision round are how the spec will be judged).
- I looked at other participants' folders **only** to see how far everyone had got (line counts, headings). I did not reuse their content: the brief says copying scores 0, and the point is to make my own decisions.

### Stack
- **NestJS + Next.js + PostgreSQL + Redis.** I'm a frontend developer working in TypeScript, so NestJS lets me share types and constants with the Next.js app (`@ro/shared`). That way a limit like the 5 MB file size has one source in code, just as it has one source in the spec (`CFG-01`).

### Key decisions (and why)
- **The original file is never stored.** It is parsed in memory; only the *redacted* text is kept, encrypted with AES-256-GCM, for 30 days. This is DPDP data minimisation and removes object storage from the design. Trade-off: no "download my original file", and re-extraction needs a re-upload.
- **4 of the 5 score dimensions are computed by code** (keyword match, section structure, parseability, impact). Only "experience relevance" and the rewrites come from the AI. Scores are repeatable and testable, and "missing skills" come only from the catalogue, so the AI cannot invent them.
- **Rewrites may not invent numbers.** Any digit in a rewrite must already be in the original line; otherwise the AI must use a placeholder such as `[percent]`, which the UI highlights for the user to fill in. This is checked in code, not just asked for in the prompt.
- **Quota is exact under concurrency.** It is reserved under a row lock in the same transaction as the insert. A partial unique index allows only one in-flight analysis per user. Failed analyses give the quota back.
- **Gating is applied at read time.** All suggestions are stored, and the current plan decides what is shown. An upgrade shows everything at once, without re-running the AI.
- **AWS Bedrock in Mumbai** as the AI provider: the data stays in India and in our AWS account, and auth is by IAM role (no API key to leak).
- **Same origin for web and API** (ALB path routing). The platform cookie reaches the API directly, so there is no CORS and no token exchange; CSRF is covered by `SameSite=Lax` plus an `Origin` check.
- **Every number is a `CFG-` constant** (§3.4), so FRs, API validation, DB checks and error messages can't drift apart — mainly for round 2.

### Things I corrected while writing (see change log §8)
- The first upload check order put "resume limit" before extraction, but duplicate detection needs the extracted text. Re-uploading an existing resume at the 10-resume limit would then have failed wrongly. I moved the duplicate and limit checks after extraction (v0.5).
- Drawing the worker flow showed that a BullMQ retry would find the analysis already `processing` and stop. I made the claim step accept `processing` for the same job (§4.11, v0.3).
- AI outages longer than 10 minutes conflicted with "queued analyses complete after recovery"; I made the rule explicit (FR-018 AC 5, NFR-006).
- Some seed skill aliases (`rest`, `spring`, `ci`, `vision`) would have matched ordinary words; I replaced them with specific phrases.
- The ERD first used a Mermaid attribute syntax that does not render; I rewrote it.

### Checks done
- ID cross-check (shell script): every `FR/NFR/CR/CFG/CMP/TBL/API/ERR/SCR` ID referenced anywhere is defined exactly once.
- Simulated build test: a separate agent got only `spec.md` and listed every guess and contradiction; the fixes are in v1.0 (change log).
