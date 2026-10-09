# Architecture Spec Homework

**AI at Work** — Write one Markdown spec that is detailed enough to build the entire system from it alone.

## Objective

Train the team to produce a single `.md` architecture specification covering Requirements, Coding Conventions, ERD/DB, API Design, and Traceability — using AI tools, but designed from the ground up.

## Sample System

**Event Check-in System (이벤트 체크인 시스템)** — Organizers create events, attendees register, and check in via QR code.

## Documents

| File | Description |
|------|-------------|
| [architecture-spec-template.md](architecture-spec-template.md) | Standard template — all 8 sections with format rules and examples |
| [homework-brief.md](homework-brief.md) | Homework brief — sample system, scope, timeline, submission rules |
| [evaluation-rubric.md](evaluation-rubric.md) | Evaluation rubric — 100-point scoring with build-test validation |

## Timeline

| Date | Milestone |
|------|-----------|
| Oct 10 (Fri) | Kickoff — brief + template shared, Q&A |
| Oct 17 (Fri) | First submission — push to repo |
| Oct 20 (Mon) | Build test — specs are cross-tested |
| Oct 21 (Tue) | Requirement change issued |
| Oct 23 (Thu) | Revised submission |
| Oct 24 (Fri) | Evaluation + review meeting |

## How to Submit

1. Create your branch: `spec/[your-name]`
2. Write `spec.md` following the template
3. Include `ai-log/` folder with your AI conversation history
4. Include `README.md` with your reflection (what was hardest, what you'd do differently)
5. Open a PR to `main`

## Participants

| Name | Branch | Status |
|------|--------|--------|
| Saswat | `spec/saswat` | — |
| Sovan | `spec/sovan` | — |
| TBD | `spec/[name]` | — |

## Evaluation

- **100 points** total across 8 sections
- Heaviest: ERD/DB (20 pts) + API Design (20 pts) + Requirements (15 pts)
- **Build Test**: Hand the spec to Claude Code with no other context — every guess counts against the spec
- **Revision Round**: +10 bonus points for maintaining consistency after a requirement change
- Grade A (90+) = buildable with zero guesswork

## Owner

Saswat leads. Created by 최광철 TL.
