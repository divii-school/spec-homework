# Process evidence — Kakali Dinda

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-20 | Claude Code (web) | https://claude.ai/code/session_01RSBw3nxhDZRYsdGaSvhTFT | Read the brief, rubric and template; chose NestJS + Next.js; drafted sections 1–7 one at a time with a commit after each; ran an independent "build test" review of the draft and fixed the gaps it found |

## Notes

**How the spec was built**
- I started from the homework page and `BRIEF.md` / `RUBRIC.md` / `TEMPLATE.md` only. I did not read other participants' specs.
- I chose the stack I work in: NestJS (API + BullMQ worker in one codebase) and the Next.js web client.
- Sections were written in order (1–2, 3, 4, 5, 6, 7) with one commit per step, so the history shows how the document grew.

**Key design decisions (and why)**
- **The original file is never stored.** It is parsed in memory and only the redacted, encrypted text is kept, for 30 days. This removes object storage from the design and the biggest DPDP risk. The trade-off is that re-analysing a file needs a re-upload.
- **4 of the 6 score dimensions are computed by code**, from a job-role term catalog. Only experience relevance, impact and the rewrites come from the AI. This makes scores repeatable and testable, and makes "missing skills" exact instead of AI-guessed.
- **The rewrite "no invented numbers" rule is enforced in code:** any digit run in the suggestion must already be in the original line, otherwise the suggestion is dropped.
- **Quota counts only successful analyses.** It is reserved in the same DB transaction as the insert and released on any failure.
- **Every repeated number is a constant (`K-xx`, section 3.4)**, so the file-size limit and the other numbers cannot drift between the FR, API and error sections.

**Checks done**
- An ID cross-check script confirmed every referenced `K-`, `ERR-`, `CMP-`, `TBL-`, `API-`, `FR-` and `NFR-` ID is defined.
- I recomputed the API-007 example by hand against the scoring formula. The first draft's numbers did not add up (v1.0 change log).
- A separate AI reviewer read only `spec.md` and listed every place a builder would have to guess. Fixes are in change log v1.1.
