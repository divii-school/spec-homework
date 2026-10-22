# Process evidence — Sovan Kumar Bera

## AI conversations

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-22 | Claude Code (Opus) | This session — local Claude Code CLI session, transcript stored locally | Full specification generation: all 8 sections (company-wide requirements, coding conventions, requirements, architecture, ERD/DB, API, traceability, change log). Includes Round 2 (Resume Pro paid plan via Razorpay). |

## Notes

- **Round 1 and Round 2 written together**: Because Round 1 was overdue and Round 2 was announced the same day, I wrote both rounds in a single pass and documented the changes in the change log (v1.0 → v1.1). The v1.1 entry lists every ID that was added or modified for the paid plan.
- **Design decisions I made** (not given in the brief): free tier limit (3/month), ATS scoring dimensions and weights (7 dimensions totalling 100), max file size (5 MB), data retention (90 days), AI provider (OpenAI GPT-4o), file storage (AWS S3 with presigned URLs), PDF generation (Puppeteer + Handlebars), currency storage (paise to avoid float errors).
- **What I checked by hand**: cross-referenced every FR-ID in the traceability matrix against the requirements table to ensure no orphaned IDs. Verified every API endpoint references a valid FR. Checked every DB column referenced in API responses exists in the ERD. Confirmed dimension weights sum to 100 (10+15+20+20+10+15+10).
- **AI limitations**: Claude generated the initial structure but I reviewed and adjusted: (a) added magic bytes validation for file uploads (AI initially only checked MIME type), (b) added the partial unique index on subscriptions to prevent double-active subscriptions, (c) specified `forbidNonWhitelisted: true` on ValidationPipe (AI defaulted to just `whitelist`).
