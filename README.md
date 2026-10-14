# AI at Work — Architecture Spec Homework

Coordinator: **Saswat Kumar Sahoo** · Slack: **#ai-spec-homework**

## What this is

Each participant writes **one Markdown spec (`spec.md`)** detailed enough that the whole system could be built from that file alone, with no other context.
You may use AI as much as you like. What is scored is how **precise, complete and consistent** the document is, and how well it stays consistent when requirements change.

No code is written in this repo. Only Markdown.

## Participants and folders

| Participant | Folder |
|---|---|
| Saswat | [`saswat/`](saswat/) |
| Sovan | [`sovan/`](sovan/) |
| Sandip | [`sandip/`](sandip/) |
| Kakali | [`kakali/`](kakali/) |
| Vaishali | [`vaishali/`](vaishali/) |
| Neeraj | [`neeraj/`](neeraj/) |

Each folder has:
- `spec.md` — your spec. It starts as a copy of [`TEMPLATE.md`](TEMPLATE.md); fill it in.
- `process.md` — links to your AI conversations and short notes on how you worked.

## Timeline

| Step | Date |
|---|---|
| Kickoff (brief, system to specify, scoring rubric) | Wed, Oct 14 |
| Round 1: write your spec | Wed, Oct 14 – Tue, Oct 20 |
| Holiday | Wed, Oct 21 |
| Round 2: requirement change, update your spec | Thu, Oct 22 – Fri, Oct 23 |
| Build test | Mon, Oct 26 |
| Scoring and review | Tue, Oct 27 |

**Start here:** read [`BRIEF.md`](BRIEF.md) (the system to specify and the rules) and [`RUBRIC.md`](RUBRIC.md) (how you are scored).

## Rules

1. **Edit only your own folder.** Do not change other people's files, `README.md` or `TEMPLATE.md`.
2. **Commit often** — at least once a day while you work. The history is part of how the work is evaluated, so please do not squash commits.
3. **Write clear commit messages**, for example `add ERD and DB tables`, `add API error codes`, `round 2: add new requirement`.
4. **Push directly to `main`** (no pull requests needed), but always pull first.
5. Keep everything in Markdown. Diagrams in Mermaid, inside the `.md` file.

## How to commit (quick guide)

First time only:

```bash
git clone https://github.com/divii-school/spec-homework.git
cd spec-homework
```

Every time you work:

```bash
git pull                                  # get the latest
# ... edit <your-name>/spec.md ...
git add <your-name>/
git commit -m "add requirements section"
git push
```

If `git push` is rejected, run `git pull` and then `git push` again. Because everyone edits only their own folder, there should be no conflicts.

Questions → **#ai-spec-homework** on Slack.
