# Process evidence — Pratiksha Patil

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-19 | Claude Code (Opus 5.5) | <add session link> | Read the brief, rubric and Slack threads; chose the stack (NestJS + Next.js); drafted design decisions, limits and §3 requirements (FR-001–FR-020, NFR-001–NFR-020) |

## Notes
Anything worth knowing about how you worked: decisions you changed, things the AI got wrong, what you checked by hand.

- **Stack:** I'm an app developer with no main backend stack, so I picked NestJS. It uses the same language (TypeScript) as the Next.js frontend, and BullMQ on Redis handles the background AI jobs.
- **Single source of truth:** every limit (file size, quota, retention and so on) is defined once in §3.3.2 and referenced by name elsewhere, so a round 2 change only has to update one table plus the sections that use it.
- **Date format change (v0.2):** while writing the i18n rules I chose "the API always sends UTC". That clashed with FR-006/FR-019, which used `+05:30`, so both were updated and the change was logged.
- **Checked by hand:** the extension-vs-content rule for uploads. The first draft said "type decided by content only" but also rejected a `.pdf` file containing DOCX. Fixed so the extension must match the detected type.
