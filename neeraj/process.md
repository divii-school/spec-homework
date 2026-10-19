# Process evidence — Neeraj Gurjar

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16 | Claude Code (VS Code extension, Claude Opus 5.5) | Claude Code sessions have no shareable link. Session ID: `fac3a8e1-2e04-4feb-8e9b-83f8b8252d78` (transcript can be exported on request) | Read BRIEF.md and RUBRIC.md, fixed the design decisions, drafted all 8 sections of `spec.md`, ran a simulated build test and a consistency audit with separate review agents, and fixed what they found. |

## Notes

- **Order of work:** first I wrote down every decision the brief leaves open (limits, scoring dimensions, retention, AI provider, storage, auth contract), then put every number in one constants table (§0.5) so that the FRs, API validation, DB constraints and error messages all point to the same value. This was done to make round 2 changes easy to follow.
- **Facts I checked instead of trusting memory:**
  - Claude model ID, pricing and the structured-output request shape were checked against Anthropic's current API documentation. Older request shapes (`budget_tokens`, `temperature` on this model) would now return errors.
  - The DPDP Rules 2025 dates (notified 13 Nov 2025, most duties from 13 May 2027) and the breach timeline (Board informed without delay, detailed report within 72 h) were checked with a web search.
- **Self-checks before submitting:**
  - A separate AI agent was given only `spec.md` and asked to act as the build-test builder and list every place it would have to guess. On v0.1 it estimated 16–20 guesses, mostly in deployment and operations (Dockerfiles, CI jobs, per-environment URLs, test tokens, metrics pipeline) plus a few runtime traps (Next.js CSP, a health check on a redirecting path, build-time `NEXT_PUBLIC_*` values in a single promoted image). These were fixed in v0.2 (see the change log).
  - A second agent audited internal consistency against `RUBRIC.md`: constants, enums, error codes, traceability links and JSON examples. It found real defects, for example that erasing data reset the FREE monthly limit and that `used` meant two different things in two endpoints. These are fixed in v0.2.
  - A second simulated build test on v0.2 estimated about 10–12 guesses, almost all in tooling and deployment (env loading for the web app, the 503/500 rule, OIDC trust, the ESM-only `pdfjs-dist` in a CommonJS NestJS build, CODEOWNERS). These were fixed in v0.3.
  - A script checks that every JSON example parses, every table has a consistent column count, every referenced ID is defined, and every FR/NFR is in the traceability matrix.
- **Decisions I changed while writing:**
  - The raw resume file is no longer kept. It is deleted right after parsing, with a 1-day S3 lifecycle rule as a backup, because nothing needs it after the text is extracted (DPDP data minimisation).
  - Hindi PDF reports were dropped, because `pdfmake` cannot shape Devanagari text correctly. The UI stays bilingual.
