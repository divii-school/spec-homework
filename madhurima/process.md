# Process evidence — Madhurima Gorain

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16, 2026-10-19 | Claude Code (desktop app) | Local session — no shareable link | Read BRIEF/RUBRIC, set up repo access, drafted sections 0–8 section by section, ran an ID cross-reference check (every CR/FR/NFR/CMP/TBL/API/ERR reference defined; every FR/NFR in the traceability matrix) |
| 2026-11-03 | Claude (claude.ai, linked to Notion and Slack) | Private session — no shareable link | Read Saswat's feedback DM and build-test run 1; fixed feedback point 3 (#7 alert job finds processes through Redis, #28 expired checkout asks Razorpay first) as spec v2.2 |

## Notes
- Backend written for **Go** (my main stack): Gin, pgx with plain SQL, golang-migrate, my own Redis-list queue with a goroutine worker pool and DLQ, Docker Compose.
- First draft used chi, sqlc, goose, asynq and AWS ECS. I replaced them in v1.1 with the tools above, which I have used in my own projects, so I can explain and build every part.
- Added a section 0 "Glossary and fixed values" so limits (5 MiB, 10 pages, 3/30 analyses, retention) are defined once and referenced everywhere, which makes round 2 changes easier to apply consistently.
- Key decisions: Anthropic Claude for analysis (forced tool call + JSON schema check), S3 in ap-south-1, fixed job-role catalogue with curated skills so missing-skills output is checkable, overall score computed by the backend instead of the AI.
- Checked by a script: no undefined IDs, no leftover template placeholders, all requirements present in section 7.
