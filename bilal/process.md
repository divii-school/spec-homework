# Process evidence — Bilal Mansuri (Frontend)

## AI conversations

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-26 | Claude (Cowork, Opus) | https://claude.ai/code/session_01FPL1KPXT1RTUwK9BthDbyr | The whole document in one working session: read `BRIEF.md` and `RUBRIC.md`, settled the design decisions listed in 3.3.4, drafted sections 1–8 of `spec.md` (v1.0), applied the round 2 Resume Pro change (v1.1), then ran an adversarial review of the finished document and fixed what it found (v1.2). |
| 2026-10-26 | Claude (separate review pass, same session) | https://claude.ai/code/session_01FPL1KPXT1RTUwK9BthDbyr | A deliberately separate pass with no memory of writing the document: "find contradictions and things a developer would have to guess". It returned 40 findings; 13 were real contradictions. The v1.2 change-log entry lists what changed and why. |

## Notes

**Round 1 was late.** Group B's round 1 deadline was Thu, Oct 22, 23:59 IST. I was on the YJN / OmniCos delivery that week and did not start; `bilal/spec.md` was still the untouched template on Mon, Oct 26. Both v1.0 and v1.1 were written on Oct 26, so the commit history for this folder is three commits on one day, not daily commits over the writing period. That is a real gap against `RUBRIC.md` section I, and I am not going to dress it up: the round 1 commit is late and should be scored as late. The round 2 commit is inside its deadline.

**Why v1.0 was committed separately.** It would have been quicker to write one file with the paid plan already in it. I wrote the round 1 scope first, committed it, and only then applied the change, so that the v1.1 diff is a real diff — new IDs appended, existing IDs untouched, a change log entry that matches what actually changed. A single combined commit would have made section H of the rubric unverifiable.

**Decisions I made that the brief did not make**, each with the reason, are in section 3.3.4 (A1–A17) rather than scattered through the document. The ones I went back and forth on:

- *History in round 1.* I nearly added an analysis-history list in v1.0 because it is obvious product sense. I left it out because the product description does not mention it — and that turned out to matter: round 2 made history a **paid** feature, and requirement 5 says free users stay exactly as they are today. If I had shipped history to everyone in v1.0, round 2 would have forced me either to take a feature away from free users or to contradict the requirement. A6 and FR-031 record this.
- *What happens to a Pro user's analyses when they stop paying.* First draft evaluated gating against the user's live plan, which means an analysis silently re-locks when the subscription ends, and a PDF exported last month no longer matches the screen. I changed it to fix the tier on the analysis row at creation time (`analysis.visibility_tier`, A17, FR-036.2). This is the one place where the round 2 change touched an existing table rather than only adding new ones.
- *"Unlimited".* Taken literally it means no abuse guard at all. I kept a 20/hour rate limit and wrote down why it is not a product limit (A15), rather than quietly ignoring the word or quietly capping the month.
- *Erasure versus financial records.* A DPDP erasure request deletes resumes and analyses, but payment rows have to survive for the statutory retention period. I wrote that exception into 5.4 explicitly instead of letting a future implementer discover the conflict.

**What I checked by hand after the AI drafted it:**

- Every `CR-`, `FR-`, `NFR-`, `CMP-`, `TBL-`, `API-` and `ERR-` id is contiguous with no gaps and no duplicates (script over the finished file).
- Every component, table and API named in the traceability matrix exists in sections 4, 5 and 6.
- All eight Mermaid diagrams render — `mmdc` rejected three sequence diagrams on the first pass because semicolons inside message text end a statement in Mermaid. Worth knowing: a diagram that looks fine in a GitHub preview can still be broken.
- The numbers that appear in more than one place (5 MB, 3 free analyses, 3 free suggestions, 180 s timeout, ₹299 / 29 900 paise, 3-day grace) are identical everywhere. The two consistency paragraphs at the end of section 7 list where each one appears, so a reviewer can check them quickly.

**What the AI got wrong or did loosely**, and I corrected: it wanted to add a `plan` column on a user row (there is no user table in this module — identity is external, A1), it proposed storing the comparison result in a table (it is a pure function of two rows, A16), and its first draft of the free-plan limit hid locked suggestions in the response instead of omitting them, which is a gating bug a client could trivially read around (FR-010.2).

**Known weak spots in this spec**, stated rather than hidden:

- The AI prompt itself is specified by its contract (Appendix A) and its parameters (3.3.1), not written out. A builder would have to write the prompt text. That is a deliberate boundary, but it is a place the build test may have to guess.
- NFR-013's "±5 points on 18 of 20 resumes" is a target I have not measured against a real model. It is the acceptance bar, not an observed result.
- The ATS dimension weights (30/20/15/15/10/10) are my judgement. They are stated and versioned so they can be argued with, but nothing in the brief validates them.
