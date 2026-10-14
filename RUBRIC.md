# Scoring Rubric — AI at Work: Architecture Spec Homework

**Total: 100 points.** Each spec is scored on what it says, not on how pretty it looks.
Following the template exactly earns little on its own: a spec that uses a slightly different layout but is precise and complete scores higher than a neat one that is vague.

---

## Summary

| # | Area | Points | What is judged |
|---|---|---|---|
| A | Requirements (FR / NFR) | 20 | Complete, testable, with IDs and acceptance criteria |
| B | Common requirements & coding conventions | 10 | Specific, checkable rules |
| C | Architecture | 10 | Components, flows, integrations, reasons for choices |
| D | ERD & DB design | 10 | Tables, types, keys, indexes, constraints |
| E | API design | 10 | Endpoints, schemas, errors, auth, pagination |
| F | Traceability & internal consistency | 10 | Every requirement maps to component, table and API, with no contradictions |
| G | Build test | 15 | How many times Claude Code had to guess |
| H | Round 2 change handling | 10 | The change updated everywhere it should, and the change log is correct |
| I | Process evidence | 5 | Commit history and AI conversation links |
| | **Total** | **100** | |

---

## Scoring levels (used for areas A–F)

Each area is scored at one of four levels, as a share of its points:

| Level | Share | Meaning |
|---|---|---|
| **Excellent** | 100% | A developer could implement this part with no questions |
| **Good** | 70% | Mostly clear; a few small gaps a developer would have to ask about |
| **Basic** | 40% | Present, but vague or missing important details |
| **Missing** | 0% | Not there, or only headings and placeholders |

---

## A. Requirements — 20 points

| Check | Excellent looks like |
|---|---|
| Coverage | Every feature in the product description has at least one FR. Edge cases are covered (bad file, AI failure, limit reached) |
| IDs | Every FR and NFR has a unique, permanent ID |
| Acceptance criteria | Each FR has numbered, **testable** criteria, e.g. "max 5 MB, otherwise `INVALID_FILE`", not "should handle large files" |
| NFRs are measurable | Numbers and units, e.g. "p95 < 15 s", "files deleted after 30 days", not "fast" or "secure" |
| Assumptions & scope | Decisions not given in the description are stated and justified; out-of-scope items are listed |

## B. Common requirements & coding conventions — 10 points

| Check | Excellent looks like |
|---|---|
| Common rules | Auth, roles, logging, error format, security, i18n, environments and deployment are all covered with specific rules |
| Checkable | Each rule can be verified, e.g. "personal data is never logged", "JWT in httpOnly cookie only" |
| Conventions | Naming, folder structure, branch and commit rules, review rules and lint/format tools are concrete, with examples |

## C. Architecture — 10 points

| Check | Excellent looks like |
|---|---|
| Components | Each component has one clear responsibility |
| Flows | Sequence diagrams for the main flows, including failure paths (AI timeout, parse failure) |
| Integrations | AI provider, file storage and platform auth: how they are called and what happens when they fail |
| Reasons | Every tech choice has a reason that fits this system, not a generic one |

## D. ERD & DB design — 10 points

| Check | Excellent looks like |
|---|---|
| Tables | Every piece of stored data has a home; no table is missing for a feature in the requirements |
| Columns | Types, null rules, defaults and constraints are given for every column |
| Keys & indexes | PKs, FKs and indexes are defined, and each index has a reason linked to a real query |
| Relationships | Cardinality and on-delete rules are stated, including what happens when a user asks to delete their data |

## E. API design — 10 points

| Check | Excellent looks like |
|---|---|
| Endpoints | Every user action in the requirements has an endpoint |
| Schemas | Request and response fields with types, required flags, validation and an example response |
| Errors | Every endpoint lists its error cases with HTTP status and error code; one full error list |
| Common rules | Auth, pagination and the error format are defined once and used consistently |

## F. Traceability & internal consistency — 10 points

| Check | Excellent looks like |
|---|---|
| Complete matrix | Every FR and NFR appears in the traceability table |
| Correct links | The components, tables and APIs named in the matrix actually exist in the spec |
| No contradictions | The same fact is the same everywhere, e.g. the file-size limit in the FR, API validation and error code all match |

**How to check quickly:** pick 5 requirements at random and follow each one through architecture, DB and API. Every broken link costs points.

## G. Build test — 15 points

On Oct 26 each spec is given to Claude Code **with no other context**, with the instruction to build the system and **log every place it had to guess or assume** something the spec did not say.

| Guesses logged | Points |
|---|---|
| 0–3 | 15 |
| 4–7 | 12 |
| 8–12 | 8 |
| 13–20 | 4 |
| More than 20 | 0 |

The same prompt and the same model are used for every spec. The guess log is shared with each participant.

## H. Round 2 change handling — 10 points

A new requirement is announced on Oct 22. The updated spec is checked for:

| Check | Points |
|---|---|
| New or changed FR/NFR with IDs and acceptance criteria | 2 |
| Architecture updated where needed | 1 |
| DB updated (new tables or columns, constraints) | 2 |
| API updated (new or changed endpoints and errors) | 2 |
| Traceability matrix updated | 1 |
| Change log entry: version, date, what, why, affected IDs | 2 |

Existing IDs must not be renumbered. A removed requirement is marked **Deprecated**, not deleted.

## I. Process evidence — 5 points

| Check | Points |
|---|---|
| Regular commits over the writing period with clear messages (not one big commit at the end) | 3 |
| AI conversation links listed in `process.md` | 2 |

---

## Rules for scoring

- **Late submission:** commits after the deadline (round 1: Tue Oct 20, 23:59 IST; round 2: Fri Oct 23, 23:59 IST) are not counted for that round.
- **Copying:** specs that are substantially the same as another participant's, or as existing internal documents, score 0 in the affected areas.
- **Coordinator's own spec:** Saswat's spec is scored by a different reviewer.
- **Feedback:** every participant receives their scores per area with short comments on Tue, Oct 27.
