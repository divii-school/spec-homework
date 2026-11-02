# Spec Playbook: how the top specs were written

Everyone used AI in this homework, and that is allowed. The scores showed a difference in **workflow**, not in ability or effort.
The highest scorers used AI in a **write, test, fix** loop. This playbook turns that workflow into steps anyone can follow.

> **Improvement round:** Fri, Oct 30 – Wed, Nov 4, 23:59 IST · **Re-score:** Thu, Nov 5 · **Goal:** everyone 90+
> Use this playbook together with the feedback you received by DM.

---

## The method in 7 steps

### 1. Decide first, write second
Before asking AI to write anything, list every decision the brief leaves open, then decide each one and give a reason.

> Example: free limit 3 per month · file max 5 MB · keep files 30 days · AI provider · storage · how Pro status is stored

Put these in your **Assumptions** section with an ID (`A-01`, `A-02`, …).
**Why:** if you skip this, the AI invents a different answer in each section, and they contradict each other.

### 2. One constants table
Every number appears **once**, in one table, and every other section refers to it by name.

| ID | Name | Value | Used by |
|---|---|---|---|
| K-01 | MAX_FILE_SIZE | 5 MB | FR-002, API-001, TBL-002 CHECK, ERR-007 |

**Why:** most "consistency" points were lost because the same value was written in two places and later changed in only one.

### 3. Write one section at a time, and commit after each
Follow the template order: 1–2, then 3, 4, 5, 6, 7, 8. Commit after each section with a clear message.
**Why:** the history shows how you worked (process points), and small steps are easier to check.

### 4. Run your own build test, more than once
Open a **new, separate** AI session and give it **only your `spec.md`**, using the prompt below. It lists every place a developer would have to guess.
Fix each finding **at its source** (the requirement or constant), not just where the problem shows up. Then run it again.

> Neeraj ran this 6 times, Sachin 3 times. **Repeat until it finds fewer than 5 guesses.**

### 5. Run consistency checks
Ask AI to write a small script for you, or check by hand:
- every ID used (`FR-`, `NFR-`, `CMP-`, `TBL-`, `API-`, `ERR-`, `K-`) is actually defined
- every error code used in the text is in the error list (6.3)
- every FR and NFR appears in the traceability matrix
- every Mermaid diagram renders
- worked examples add up (recompute scores by hand)

### 6. Handle a change like an engineer
When a requirement changes (Round 2):
1. **List every affected ID first**, before editing anything.
2. Update in order: requirements, then architecture, DB, API, traceability, change log.
3. **Search the whole document** for old wording (for example "out of scope", old limits, old plan names) and fix every stale line.
4. Never renumber IDs. Mark removed items **Deprecated**.
5. Add a change log row with the version, date, what changed, why, and the affected IDs.

### 7. Check facts, don't trust memory
Check API details (AI model names, Razorpay fields, library behaviour) against the official docs. Write in `process.md` what you checked and what the AI got wrong.

---

## Copy-paste prompts

### Build test prompt (use in a NEW session, with only your spec attached)

```
You are a senior developer. You must build this whole system using ONLY the attached spec.md.
You cannot ask anyone questions.

Walk through what you would build: database migrations, every API endpoint, background
workers and the AI call, frontend screens, payments (Razorpay), configuration for
local/dev/staging/prod, and deployment.

List every place where you would have to GUESS or ASSUME something because the spec does not
say it, or where two parts of the spec contradict each other. Only count gaps that would change
what gets built (ignore wording and style).

For each item give: the section/ID, what is missing or contradictory, and a one-line suggested fix.
End with the total count.
```

### Consistency check prompt

```
Check this spec.md for internal consistency:
1. List every ID that is used but never defined, and every ID defined but never used.
2. List every error code used in the text but missing from the error code list.
3. List every FR/NFR missing from the traceability matrix.
4. List every number (limits, sizes, durations, prices) that appears in more than one place
   with different values.
5. Pick 5 requirements and trace each through architecture, DB and API; report any broken link.
```

### Change-request prompt

```
Here is a requirement change: <paste change>.
Before editing, list every ID in spec.md that this change affects (FR, NFR, CR, CMP, TBL, API,
ERR, constants, assumptions, out-of-scope items), and every place where old wording would become
wrong. Do not edit yet.
```

---

## Checklist before you push

- [ ] All open decisions are written as assumptions, with reasons
- [ ] Every number lives in one constants table
- [ ] Build test run at least twice, and the last run found fewer than 5 guesses
- [ ] Consistency check is clean (IDs, error codes, matrix, numbers)
- [ ] Payment edge cases covered: late or out-of-order webhooks, expired checkouts, cancel, erase with an active subscription
- [ ] Frontend deployment and per-environment config (local/dev/staging/prod) are written
- [ ] Change log has a row for this improvement round, with the affected IDs
- [ ] `process.md` has your AI sessions (a shareable link or exported transcript) and what you checked by hand
- [ ] Committed in small steps over several days

---

## Most common gaps from the first scoring

1. **Rules drifting apart.** The same rule was written in two places, and only one was updated.
2. **Payment edge cases.** Razorpay status mapping, late webhooks overwriting newer state, and expired checkouts still charging.
3. **Frontend and deployment.** Hosting, environment variables, and the 401 redirect to the platform login.
4. **Config per environment.** Values were given only for prod.
5. **Process.** A whole round committed in 10 minutes, and session IDs instead of shareable AI links.

Questions → **#ai-spec-homework**.
