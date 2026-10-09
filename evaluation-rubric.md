# Architecture Spec Homework — Evaluation Rubric

**Total: 100 points**

---

## Section Scores

### 1. Company-Wide Common Requirements (8 pts)

| Criteria | 0 pts | 4 pts | 8 pts |
|----------|-------|-------|-------|
| Completeness | Missing 3+ areas (auth, logging, security, etc.) | Covers all areas but some are vague | All areas covered with specific rules, values, and formats |

**What "8 pts" looks like**: Auth method specified (JWT, token expiry values), password policy with exact rules, logging format defined (structured JSON, which fields), error response format with examples, CORS/rate-limit values, i18n rules (timezone, currency format), all environments listed with deploy triggers.

---

### 2. Coding Conventions (8 pts)

| Criteria | 0 pts | 4 pts | 8 pts |
|----------|-------|-------|-------|
| Specificity | Generic ("use camelCase") | Covers naming + structure but missing tooling | Naming table for every element type, folder tree, branch strategy, commit format, lint/format config with exact rules |

**What "8 pts" looks like**: A builder can set up ESLint, Prettier, Husky, and folder structure from this section alone without Googling.

---

### 3. Requirements Definition (15 pts)

| Criteria | Points | Description |
|----------|--------|-------------|
| Coverage | 0–5 | All 7 features have FR-IDs. All NFRs (performance, availability, security) present. |
| Acceptance criteria | 0–5 | Every FR has numbered, testable criteria. No "should work correctly" — only measurable. |
| Priority | 0–3 | Must / Should / Could assigned. Prioritization makes sense. |
| ID consistency | 0–2 | IDs are sequential, no gaps, no duplicates. Referenced correctly in other sections. |

**Deductions**: -2 per FR without acceptance criteria. -1 per vague criterion ("fast", "secure", "user-friendly").

---

### 4. Architecture (10 pts)

| Criteria | Points | Description |
|----------|--------|-------------|
| Diagram | 0–3 | System diagram shows all components and their connections. Readable. |
| Tech justification | 0–4 | Every tech choice has a reason (not "because it's popular"). Alternatives considered. |
| Data flow | 0–3 | Request lifecycle from user action → frontend → API → DB → response is traced. |

**What "10 pts" looks like**: A new developer can draw the system on a whiteboard from this section alone.

---

### 5. ERD & DB Design (20 pts)

| Criteria | Points | Description |
|----------|--------|-------------|
| Table coverage | 0–5 | Every entity in the requirements has a table. No phantom tables without a requirement. |
| Column detail | 0–8 | Every column has: type, nullable, default, description. No ambiguity. |
| Keys & indexes | 0–4 | PK, FK, unique constraints, and indexes defined with reason. |
| Constraints | 0–3 | CHECK constraints, cascading rules (ON DELETE), soft delete strategy documented. |

**Deductions**: -2 per table missing a column that the API section references. -1 per FK without ON DELETE behavior specified.

**Build test check**: Can `CREATE TABLE` statements be written from this section with zero guesswork?

---

### 6. API Design (20 pts)

| Criteria | Points | Description |
|----------|--------|-------------|
| Endpoint coverage | 0–5 | Every FR maps to at least one endpoint. CRUD is complete (no missing DELETE or PATCH). |
| Request schema | 0–5 | Every endpoint has: request body (JSON), validation rules (required, type, min/max). |
| Response schema | 0–5 | Success + error responses with actual JSON examples. Status codes correct. |
| Auth & pagination | 0–3 | Which endpoints need auth, which roles can access. Pagination format for list endpoints. |
| Error codes | 0–2 | Application-specific error codes beyond HTTP status (e.g., EVENT_FULL, ALREADY_CHECKED_IN). |

**Deductions**: -2 per endpoint without request/response schema. -1 per endpoint without auth specification.

**Build test check**: Can all API routes be implemented from this section without reading the requirements section?

---

### 7. Traceability Matrix (10 pts)

| Criteria | Points | Description |
|----------|--------|-------------|
| Completeness | 0–5 | Every FR-ID and NFR-ID appears in the matrix. |
| Accuracy | 0–5 | Mappings are correct: the listed tables and endpoints actually implement the requirement. |

**Deductions**: -1 per requirement ID missing from the matrix. -2 per incorrect mapping (FR says one thing, mapped endpoint does another).

---

### 8. Change Log (4 pts)

| Criteria | 0 pts | 2 pts | 4 pts |
|----------|-------|-------|-------|
| Structure | Missing or just dates | Has version + date + description | Version, date, author, what changed, why, affected requirement IDs |

*This section is scored mainly in the Revision Round.*

---

## Bonus & Penalty

### Process Evidence (bonus +5 pts)

| Criteria | Points |
|----------|--------|
| AI conversation links submitted | +1 |
| Commit history shows iterative work (not one giant commit) | +2 |
| README reflection is thoughtful (not generic) | +2 |

### Revision Round (bonus +10 pts)

| Criteria | Points |
|----------|--------|
| Change log updated correctly | +2 |
| Requirement IDs consistent after change | +2 |
| ERD updated to match | +2 |
| API updated to match | +2 |
| Traceability matrix still in sync | +2 |

### Penalties

| Issue | Deduction |
|-------|-----------|
| Spec handed to Claude Code, builder had to guess | -1 per guess |
| Section completely missing | -full section score |
| Copy-pasted AI output without editing (detected via conversation log) | -5 |
| Inconsistency between sections (e.g., API references a column not in ERD) | -2 per instance |

---

## Grade Scale

| Score | Grade | Meaning |
|-------|-------|---------|
| 90–115 | A | Spec is buildable with near-zero guesswork |
| 75–89 | B | Spec is buildable with minor clarifications needed |
| 60–74 | C | Spec has significant gaps — builder needs to make assumptions |
| Below 60 | D | Spec cannot serve as a build document — major revision needed |

---

## Evaluator Notes

- Score each section independently first, then check cross-section consistency
- The **Build Test** is the ultimate validation — if the spec builds cleanly, minor formatting issues don't matter
- Evaluate the **quality of thinking**, not just completeness — a shorter spec that makes precise choices is better than a long spec that lists every option without deciding
