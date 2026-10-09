# AI at Work — Architecture Spec Homework Brief

## Objective

Write **one Markdown specification** that is detailed enough to build the entire system from it alone — using AI (Claude Code), but designed from the ground up by you.

The goal is not to produce a perfect document on the first try. The goal is to **learn how precisely a spec must be written** so that an AI tool (or another developer) can build from it with zero guesswork.

---

## Sample System: **Event Check-in System (이벤트 체크인 시스템)**

A system where organizers create events, attendees register, and check in via QR code on the day.

### Why this system
- Familiar domain — everyone has used event registration
- Covers auth, roles, CRUD, relationships, real-time, file uploads
- Small enough to spec in one document, complex enough to test depth

### Core Features to Specify

| # | Feature | Key Complexity |
|---|---------|---------------|
| 1 | User registration & login | Auth, password rules, JWT |
| 2 | Role-based access (Admin / Organizer / Attendee) | Permission matrix |
| 3 | Event CRUD (create, list, detail, edit, delete) | Pagination, filtering, soft delete |
| 4 | Event registration (attendee signs up for event) | Capacity limits, waitlist, duplicate check |
| 5 | QR code check-in | Generate QR per registration, scan to check in, prevent double check-in |
| 6 | Dashboard — Organizer sees attendee list + check-in stats | Aggregate queries, real-time count |
| 7 | Email notification on registration + reminder | Async job, template, retry |

### Out of Scope (do NOT include)
- Payment / ticketing
- Social login (Google, Kakao)
- Mobile app (web only)
- Multi-language UI (backend i18n rules still apply)

---

## Deliverable

One `.md` file following the **standard template** (provided separately). The file must cover all 8 sections:

1. **Company-wide common requirements** — auth, roles, logging, error handling, security, i18n, environments
2. **Coding conventions** — naming, folder structure, branching, code review, lint/format
3. **Requirements definition** — FR-001 ~ FR-NNN, NFR-001 ~ NFR-NNN, each with acceptance criteria
4. **Architecture** — system diagram, components, tech stack with justification, data flow
5. **ERD & DB design** — every table, every column (type, nullable, default, key, index), constraints, relationships
6. **API design** — every endpoint (method, path, auth, request/response schema, error codes, pagination)
7. **Traceability matrix** — every requirement ID mapped to architecture component → DB table → API endpoint
8. **Change log** — version history of the document itself

### Quality Bar

Your spec will be judged by the **Build Test**:
> Hand the .md file to Claude Code (or to another participant) with **no other context** — just the file. Try to build the system from it.
> Every place where the builder has to **guess** counts against the spec.

If the builder can stand up the DB, implement all APIs, and pass the acceptance criteria without asking a single question — the spec is good.

---

## Timeline

| Date | Milestone |
|------|-----------|
| Oct 10 (Fri) | Kickoff — brief + template shared, Q&A |
| Oct 17 (Fri) | **First submission** — push to repo |
| Oct 20 (Mon) | Build test — specs are cross-tested |
| Oct 21 (Tue) | **Requirement change** issued (see Revision Round below) |
| Oct 23 (Thu) | **Revised submission** — updated spec pushed |
| Oct 24 (Fri) | Evaluation + review meeting |

---

## Revision Round

After the first submission, a **requirement change** will be issued. Examples:
- "Add a waitlist feature when event capacity is full"
- "Add export to CSV for the organizer dashboard"
- "Change QR check-in to allow check-in within 30-min window only"

You must update your spec — and the evaluator will check:
- Did the **change log** capture what changed and why?
- Did the **requirement IDs** stay consistent (no orphaned or missing IDs)?
- Did the **ERD** and **API** sections get updated to match?
- Did the **traceability matrix** stay in sync?

This round tests whether your document is **maintainable**, not just correct at a point in time.

---

## Submission Rules

### Repository
- Repo: `[TBD — Saswat to create]`
- Each participant creates a branch: `spec/[your-name]` (e.g., `spec/sovan`, `spec/saswat`)
- Submit via PR to `main`

### What to Submit
1. `spec.md` — your architecture specification (following the template)
2. `ai-log/` folder — links or exports of AI conversations used while writing
3. `README.md` — 3 sentences: what you found hardest, what you'd do differently, how much AI helped vs. hindered

### Rules
- You **may** (and should) use AI tools (Claude, ChatGPT, etc.) to help write
- You **must** submit the AI conversation history as evidence
- Copy-pasting an AI output without reviewing / editing counts against you (see Rubric: "Process Evidence")
- Collaboration with other participants is allowed for Q&A but not for the spec content itself

---

## Participants

| Name | Branch |
|------|--------|
| Saswat | `spec/saswat` |
| Sovan | `spec/sovan` |
| [Participant 3] | `spec/[name]` |
| [Participant 4] | `spec/[name]` |
| [Participant 5] | `spec/[name]` |
| [Participant 6] | `spec/[name]` |

---

## Questions?

Post in the Notion page comments or ask in the kickoff meeting (Oct 10).
