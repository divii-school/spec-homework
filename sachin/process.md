# Process evidence — Sachin S M

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|

## Notes
Anything worth knowing about how you worked: decisions you changed, things the AI got wrong, what you checked by hand.

- 2026-10-19: Fixed the core design decisions before writing (Spring Boot 3 / Java 21, Claude API for scoring, S3 in ap-south-1 for DPDP, 5 MB / 5-page limit, FREE 3 analyses and 3 suggestions per month, PREMIUM 50 and all, 30-day file retention, 180-day result retention). Then wrote sections 1–3 so that every later section can trace back to an ID.
- 2026-10-20: Sections 4 (architecture) and 5 (database). Checked the AI call against the current Anthropic API reference: current Claude models reject forced `tool_choice` and do not accept `temperature`, so the spec uses structured outputs (`output_config.format`) and A-9 was corrected. Their reasoning counts towards `max_tokens`, so NFR-015 went from 4,000 to 8,000. Designing the sweeper showed that FR-008 had no end state for a message lost twice; added it to criterion 7. Tested the four redaction regexes against sample phone, email and URL formats (the first phone pattern left a stray `(` on `(080) 2345 6789`, fixed), and parsed all 8 Mermaid diagrams with Mermaid 11 before committing.
