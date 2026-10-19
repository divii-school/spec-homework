# Process evidence — Nikky Navvya

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-19 | Claude Code (VS Code extension, model Claude Fable 5.1) | Local session `6e99ee2c-037e-4432-99e2-321b30efe5df` (Claude Code sessions have no public share link; transcript available on request) | Read the Slack channel and repo files, explained the homework to me, then drafted all 8 sections of `spec.md` for the Go + Next.js + PostgreSQL + Redis stack and wrote this file. |

## Notes
Anything worth knowing about how you worked: decisions you changed, things the AI got wrong, what you checked by hand.

- **Stack:** Go (Gin + GORM) backend using the same flat layered folder layout as our GoTalk project, Next.js frontend, PostgreSQL and Redis as required. I chose Go because it is the stack I currently work in day to day.
- **Decisions I asked the AI to make explicit** rather than leave open: free/pro limits (3 and 50 analyses per month, 3 missing skills and 2 suggestions visible on free), six scoring dimensions with fixed weights, file limits (5 MiB, 10 pages, 200–30,000 characters), async analysis with 2 s polling, synchronous text extraction at upload, 30-day file retention and 365-day result retention, Anthropic Claude as AI provider, S3 in ap-south-1 for DPDP data residency.
- **Checked by hand:** the same number appears everywhere it is used (file size, page limit, text length, plan limits, retention days, dimension weights). The worked example in FR-009 and API-011 (80, 60, 70, 90, 50, 70 → 72) was recomputed manually.
- **Things to watch in round 2:** IDs are never renumbered; new requirements get the next free FR/API/TBL/ERR number and a change log row.
