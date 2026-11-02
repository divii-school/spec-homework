# Build test, round 2 (spec v0.4)

Exported transcript, 2026-10-25. Claude Code agent run on its own, reading only `sachin/spec.md`. Tool results (the spec text it read) are left out; each tool call is shown as one line.

## Prompt

```text
You are simulating the "build test" of a spec-writing homework. Read ONLY this one file: sachin/spec.md (about 2,700 lines; read all of it, in chunks). Do NOT open any other file or folder in C:\Users\user\spec-homework (other participants' folders must not be read; copying is penalised). Do not edit any file.

Pretend you are Claude Code asked to build the whole system (Spring Boot 3.4 / Java 21 backend, Next.js frontend, PostgreSQL, Redis) from this spec alone, and to log every place where you would have to GUESS or ASSUME something the spec does not state, and every CONTRADICTION between two places in the spec. The scoring counts guesses: 0–3 is best, more than 20 is worst, so be strict but fair. A guess is a decision a builder must make that changes code or behaviour and is not answered anywhere in the spec. It is not a guess if the spec answers it somewhere, even in another section; check before logging. Generic framework defaults (e.g. how to configure Jackson) are not guesses unless behaviour visible to a client or tester depends on them.

Focus most on the v0.4 "Resume Pro" round-2 change (Razorpay subscription ₹299/month, unlimited analyses, all suggestions, Pro analysis history, comparison of two analyses, cancel with access until end of paid month, free users unchanged). Look at: FR-024 – FR-030, NFR-019 – NFR-021, CR-032/033, A-11 – A-16, CMP-018 – CMP-022, flows 4.3.8 – 4.3.11 and the subscription state diagram, 4.4/4.5/4.7 billing rows, TBL-016 – TBL-019 and 5.6 migration, API-017 – API-022 and the "Subscription view", ERR-021 – ERR-025, the traceability rows and the v0.4 change-log row. Also check that every place that used to say PREMIUM or "plan from the token" was updated consistently, and that the round-1 parts still agree with the new parts (quota, visibility, erasure, retention, statistics, usage endpoint).

Return a numbered list. For each item: section/ID, what is missing or contradictory (quote the conflicting text briefly), why a builder would have to guess, and a one-sentence proposed fix. Order by importance. Then give a total count of real guesses, separately count contradictions, and list anything you checked and found fine in one short paragraph. Keep the report under 1,200 words.
```

- *tool:* Read `spec.md` lines 1-400
- *tool:* Read `spec.md` lines 1-300
- *tool:* Read `spec.md` lines 300-599
- *tool:* Read `spec.md` lines 600-899
- *tool:* Read `spec.md` lines 899-1198
- *tool:* Read `spec.md` lines 1199-1498
- *tool:* Read `spec.md` lines 1499-1748
- *tool:* Read `spec.md` lines 1749-2048
- *tool:* Read `spec.md` lines 2049-2328
- *tool:* Read `spec.md` lines 2329-2578
- *tool:* Read `spec.md` lines 2578-2827
## Final report

**Build test of sachin/spec.md (v0.4, Resume Pro)**

I read all 2,677 lines and opened no other file. The items are ordered by importance. **G** marks a guess and **C** marks a contradiction.

1. **C: 5.6 migration order.** Step 1 runs `UPDATE analysis SET plan_at_start = 'PRO' WHERE plan_at_start = 'PREMIUM'` (and the same on `analysis_outcome`). Step 2 is "Replace `ck_analysis_plan_at_start` … with `IN ('FREE','PRO')`". When step 1 runs, the v0.3 CHECK `IN ('FREE','PREMIUM')` is still in place. Every dev or staging row holding PREMIUM therefore fails the check, and `V3` aborts. The spec says those rows exist ("only synthetic dev and staging rows can hold `PREMIUM`"). A builder must reorder the steps against the text. **Fix:** drop both constraints, then UPDATE, then add the new constraints.

2. **G: Pro page for a FREE user whose subscription is still open (FR-030 c3, FR-024 c2, FR-028 c1).** A renewal that fails puts the row in `PENDING`, and the plan becomes FREE at `access_until` (A-16, 4.3.9 timeline). An `AUTHENTICATED` row before the first charge is also FREE. FR-030 c3 shows every FREE user "Subscribe", but API-017 returns 409 `SUBSCRIPTION_EXISTS` for PENDING or AUTHENTICATED rows. FR-028 allows cancelling PENDING, yet the UI shows "Cancel" only to PRO users. The builder has to invent this page state, its text and its buttons. **Fix:** add a FR-030 c3 case: "FREE with an AUTHENTICATED/PENDING subscription: show 'Payment pending, Razorpay is retrying' and Cancel, no Subscribe."

3. **C: consent withdrawal vs A-14.** A-14 says buying Pro "is a separate purpose" and "asking for the resume-processing consent would mix the two purposes". But FR-002 c1 and API-003 say that withdrawing the analysis consent runs the full erasure, "including the Razorpay cancellation of FR-019 criterion 7", with no refund. So withdrawing one purpose's consent kills the other purpose's paid service. The warning dialog in FR-019 c8 is defined only for erasure, so whether to warn a PRO user before withdrawing consent is a further UI guess. **Fix:** either keep billing rows on consent withdrawal (as A-14 implies), or state the trade-off in A-14 and extend FR-019 c8 to API-003.

4. **G: `FakeRazorpayClient` behaviour (CMP-019, 1.7 environment table).** `FakeAiClient` is specified exactly, but the fake billing client is not: the ids it returns (they must match `^sub_[A-Za-z0-9]{14}$`), what `fetch` returns (`active`? `created`?) and what `cancel` does. Real `checkout.js` cannot open a fake `sub_` id, and the spec does not say how local and Playwright users get through Checkout. Local runs, e2e tests and `scripts/send-test-webhook.sh` all depend on this. **Fix:** add a FakeRazorpayClient paragraph like the one for FakeAiClient, plus a local "simulate payment" path.

5. **C: when the plan is computed (FR-025 c3 vs API-018).** FR-025 c3 says the plan is "computed once at the start of each request that needs it", and its list (and the 7.1 row) leaves out API-018 and API-020. API-018 answers "200 when the user's plan is `PRO` after processing", and API-018 and API-020 return the subscription view, which contains `plan` and `accessUntil`. A builder cannot satisfy both. **Fix:** add API-018 and API-020 to FR-025 c3 and state that these two compute the plan after their own update.

6. **C: FR-028 c7 vs API-020 and 4.3.10.** FR-028 c7 says the response has "`accessUntil` and `nextChargeAt` null". API-020 and the 4.3.10 diagram say "`accessUntil` unchanged, `nextChargeAt` null". **Fix:** reword FR-028 c7 to "`accessUntil` unchanged and `nextChargeAt` null".

7. **G: what still applies when a status change is skipped (4.3.9, 4.3.8).** For `charged`, only "the payment and `access_until` parts" are applied when the status change is skipped. It is not stated whether `current_start` / `current_end` are still overwritten by an older event. That value drives `currentPeriodEnd` and `nextChargeAt`. The same applies to `activated`. For verification, "never changes a terminal status" does not say whether `access_until` is still raised on a terminal (for example EXPIRED) row when Razorpay says `active`; the webhook rule says yes for `charged`. **Fix:** list per event which fields are written when the status change is skipped, and give verification the same rule.

8. **C: 5.4 vs 4.3.6 and FR-019 c7.** The 5.4 delete table says open subscriptions are "cancelled at Razorpay first". 4.3.6 and FR-019 c7 cancel them "right after the commit" (TBL-019 row in the transaction, call after the commit). **Fix:** change 5.4 to "TBL-019 row written in the erasure transaction, cancelled at Razorpay after the commit".

9. **G: `billing_activation_seconds` (NFR-020).** The metric is defined as "payment time → `access_until` set, with a label for the path", but:
   - the label name and its values are not given;
   - the verification path only fetches the subscription, which carries no payment time, so the builder must choose the start instant;
   - the metric is missing from the NFR-013 metric list.

   **Fix:** name the label and its values (for example `path` = `verification|webhook|reconciliation`), define the start time for each path, and add the metric to NFR-013.

10. **C: leftovers from v0.3.** A-1 still says the token carries "`sub`, `plan`, `roles`, `aud`", against CR-003 v0.4 ("the `plan` claim is no longer read"). CR-011 was edited in v0.4 (fair-use and reconciliation WARNs) but is missing from the v0.4 change-log "Changed" list and its Affected IDs. This changes no code, but it breaks the "same fact everywhere" rule. **Fix:** mark A-1 changed (drop `plan`) and add CR-011 and A-1 to the v0.4 row.

11. **C (low): the subscription state diagram is not complete.** The 4.3.9 event table and FR-029 c8 allow transitions the diagram lacks:
    - `subscription.cancelled` from `CREATED` (support cancels in the dashboard);
    - `expired` → `EXPIRED` from any open row during reconciliation;
    - `activated` from `PENDING`.

    The analysis diagram is enforced in code ("any other transition is rejected"), so a builder could read this diagram as exhaustive too. **Fix:** say that the event table plus the skip rule are the only guard and the diagram is illustrative, or add the missing arrows.

**Totals:** 4 real guesses (items 2, 4, 7, 9) and 7 contradictions (items 1, 3, 5, 6, 8, 10, 11). Item 1 is the only one that would fail on deploy as written. Items 10 and 11 change no behaviour.

**Checked and found fine:**
- **Every former PREMIUM and token-plan place was updated:** FR-007, FR-012, FR-016, FR-022, FR-023, CR-003, CMP-002 and CMP-007, TBL-006 and TBL-014, API-009, API-012 and API-015, ERR-007, 4.7 (QUOTA_PREMIUM deprecated) and 7.3.
- **Quota:** PRO usage is still counted. The 20-used example gives 0 remaining, and the fair-use WARN fires at exactly 300.
- **Visibility:** read-time visibility applies to both the web view and the PDF.
- **Erasure:** advisory lock, four-table guard, TBL-019 outliving the deleted row, CREATED rows also cancelled, webhooks after erasure IGNORED.
- **Retention:** ended subscriptions deleted after 180 days, events after 90 days, and the audit query.
- **Billing flow:** FR-024 check order and lock, the uniqueness backstop, Checkout and webhook HMAC rules, webhook check order, dedup and ordering, the `GREATEST` rule, the renewal timeline, reconciliation mapping, cancel paths (cycle-end vs immediate), and ShedLock names.
- **API and errors:** API-021 check order and CR-014's 404 for both ids, FR-026 filters (400 vs 403 order, unknown `resumeId` gives an empty list), ERR-021–025 against their endpoints, the "Continue payment" flow (re-POST returns the same CREATED row with its key), and the traceability rows for FR-024–030 and NFR-019–021.
