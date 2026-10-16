# Homework Brief — AI at Work: Architecture Spec Homework

Coordinator: **Saswat Kumar Sahoo** · Slack: **#ai-spec-homework** · Scoring: [`RUBRIC.md`](RUBRIC.md) · Template: [`TEMPLATE.md`](TEMPLATE.md)

---

## 1. The goal

Write **one Markdown spec (`<your-name>/spec.md`) detailed enough that the whole system can be built from it alone**, with no other context and no questions to you.

Use AI (Claude, ChatGPT, etc.) as much as you like. What is scored is how **precise, complete and consistent** your document is, and how well it stays consistent when a requirement changes.

## 2. The system: SurfingBear Resume Optimizer

Everyone specifies the same system, so the specs can be compared side by side.

**Product description — this is all the input you get:**

> SurfingBear users can upload their resume (PDF or DOCX) and choose a target job role and experience level. The system reads the resume, uses AI to score it out of 100 on several ATS dimensions, lists the skills that are missing for that role, and suggests rewrites for weak lines. Free users get a limited number of analyses per month and see only some of the suggestions. Users can download the analysis as a PDF. Users are already logged in through the main SurfingBear platform; this module does not handle sign-up or login itself. Resumes contain personal data and must be handled under India's DPDP Act.

**Tech stack:**
- **Backend:** your own main stack — NestJS (TypeScript), Spring Boot (Java) or Go. Write the spec for the stack you work in.
- **Frontend:** Next.js. App developers may specify a mobile app client in their own app stack instead.
- **Database and cache (same for everyone):** PostgreSQL and Redis.

Whichever stack you use, explain why each part fits this system.

**Everything not stated above is your design decision** — limits, scoring dimensions, tables, endpoints, error codes, data retention, performance targets, which AI provider and file storage to use. Decide it, write it down, and give the reason. There is no single "correct" answer; there is only clear and complete versus vague and incomplete.

**Do not copy** existing internal SurfingBear spec pages or another participant's spec. The point is to build the document yourself.

## 3. What you submit

In your own folder:

| File | What goes in it |
|---|---|
| `spec.md` | Your spec. Start from the copy of the template that is already there and replace every `<placeholder>`. All 8 sections are required. |
| `process.md` | Links to every AI conversation you used, with one line on what each was for. |

The 8 sections: (1) company-wide common requirements, (2) coding conventions, (3) requirements — FR / NFR with IDs and acceptance criteria, (4) architecture, (5) ERD and DB design, (6) API design, (7) traceability matrix, (8) change log.

## 4. Timeline

Group A: Saswat, Sovan, Sandip, Kakali, Vaishali, Neeraj
Group B (joined Oct 16): Bilal, Madhurima, Nikky, Pranesh, Pratiksha, Sachin

| Step | What happens | Group A | Group B |
|---|---|---|---|
| Start | Brief, template and rubric shared | Wed, Oct 14 | Fri, Oct 16 |
| **Round 1: write** | Write your spec; commit as you go | **Tue, Oct 20, 23:59 IST** | **Thu, Oct 22, 23:59 IST** |
| Holiday | — | Wed, Oct 21 | Wed, Oct 21 |
| **Round 2: change** | A new requirement is announced. Update your spec everywhere it is affected and add a change log entry | Announced Thu, Oct 22 · due **Fri, Oct 23, 23:59 IST** | Announced Fri, Oct 23 · due **Mon, Oct 26, 23:59 IST** |
| Build test | Each spec is handed to Claude Code with no other context; every guess it has to make is logged | Mon, Oct 26 | Tue, Oct 27 |
| Scores and review | Scores per area with short comments; review session | Wed, Oct 28 | Wed, Oct 28 |

Commits after a deadline are not counted for that round.

## 5. How to work

- Edit **only your own folder**. Commit at least once a day while you work, with clear messages (`add ERD`, `add API error codes`, `round 2: add new requirement`). Do not squash commits — the history is part of your score.
- Always `git pull` before you start and `git push` when you stop. See `README.md` for the commands.
- Diagrams go inside the `.md` file as Mermaid.
- Tip: before you finish, read your spec as if you were the developer. Every time you think "what does this mean?" or "what happens if…?", the spec is missing something.

## 6. How you are scored

Full details in [`RUBRIC.md`](RUBRIC.md). In short:

| Area | Points |
|---|---|
| Requirements (FR / NFR) | 20 |
| Common requirements & coding conventions | 10 |
| Architecture | 10 |
| ERD & DB design | 10 |
| API design | 10 |
| Traceability & consistency | 10 |
| Build test (fewer guesses = more points) | 15 |
| Round 2 change handling | 10 |
| Process evidence | 5 |
| **Total** | **100** |

Questions → **#ai-spec-homework**.
