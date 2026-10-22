# Process evidence — Neeraj Gurjar

## AI conversations
List every AI conversation you used, with a link and one line on what it was for.

| Date | Tool | Link | What I used it for |
|---|---|---|---|
| 2026-10-16 | Claude Code (VS Code extension, Claude Opus 5.5) | Claude Code sessions have no shareable link. Session ID: `fac3a8e1-2e04-4feb-8e9b-83f8b8252d78` (transcript can be exported on request) | Read `BRIEF.md`, `RUBRIC.md` and the Slack channel, fixed the open design decisions, drafted all 8 sections (v0.1), then ran two simulated build tests and a consistency audit with separate review agents and fixed what they found (v0.2, v0.3). |
| 2026-10-19 | Claude Code (same session as above) | same session ID | Third simulated build test and fixes (v0.4); Korean UI language and check-mark permission table (v0.5); review of my spec against the rubric and fixes (v0.6–v0.9). |

## Notes

### How I worked

1. Wrote down every decision the brief leaves open (limits, scoring dimensions, retention, AI provider, storage, auth contract) before writing any section.
2. Put every number in one constants table (§0.5) with a "Used by" column, so the FRs, API validation, DB constraints, error messages and the notice all point to the same value. This is mainly for round 2: a changed limit can be followed to every place it appears.
3. Wrote the sections in template order, then reviewed the whole document as a builder would (below), and recorded every round in the change log (§8.2).

### Timeline

| Date | Version | What changed | Commit |
|---|---|---|---|
| 2026-10-16 | v0.1 | First complete draft of sections 0–8 | committed 2026-10-19 (`6063a71`); I wrote it on 16 Oct but only committed on 19 Oct |
| 2026-10-16 | v0.2, v0.3 | Fixes from build tests 1 and 2 and the consistency audit | `f816b29` |
| 2026-10-19 | v0.4, v0.5 | Build test 3 fixes; Korean UI; ✔️/✗ permission table | `411590d` |
| 2026-10-19 | v0.6 | No-invented-numbers rule for rewrites; review defects | `19f88db` |
| 2026-10-19 | v0.7 | Missing core skills decided by code from catalog match terms | `c825eb9` |
| 2026-10-19 | v0.8 | Outage matrix, AI circuit breaker, criterion-to-test check, personal data inventory, reverse traceability | `9bf4c39` |
| 2026-10-19 | v0.9 | Resume limit, script check at parse, exact error bodies, test fixtures, grievance erasure | `9a8ba53` |
| 2026-10-19 | v0.10 | Build test 4 fixes | `c68a96b` |
| 2026-10-19 | v0.11 | Build test 5 fixes | `5b73fe5` |

### Main decisions and their trade-offs

- **Analysis runs in a background worker (BullMQ), not in the request.** AI calls take up to a minute and file parsing is CPU-heavy and works on untrusted files. Trade-off: the web app has to poll, and there are two processes to run.
- **Quota is reserved at creation and committed or released at the end, under a row lock.** Failed analyses never cost the user anything, and parallel requests cannot overshoot the limit. Trade-off: more states to handle (stale janitor, erasure race, lock order).
- **The raw file is kept only until it is parsed.** Only redacted, encrypted text is stored. Trade-off: re-parsing with a better extractor later is not possible.
- **Missing core skills are decided by code, not by the AI (v0.7).** Each core skill has match terms, so the result is deterministic and testable. Trade-off: the match terms have to be maintained in the catalog.
- **AI output and the PDF are English only; the UI has English, Hindi and Korean.** `pdfmake` cannot shape Devanagari, and one report language keeps tests simple.
- **Claude `claude-opus-5-5` with structured output.** The JSON schema removes most invalid-output failures, and the backend computes the overall score so the number is always consistent.

### Facts I checked instead of trusting memory

- Claude model ID, pricing and the structured-output request shape were checked against Anthropic's current API documentation. Older request shapes (`budget_tokens`, `temperature` on this model) would now return errors.
- The DPDP Rules 2025 dates (notified 13 Nov 2025, most duties from 13 May 2027) and the breach timeline (Board informed without delay, detailed report within 72 h) were checked with a web search.

### Self-checks

- **Simulated build test.** A separate AI agent got only `spec.md` and acted as the builder, listing every place it would have to guess.

  | Round | Spec version | Estimated guesses | Main areas |
  |---|---|---|---|
  | 1 | v0.1 | 16–20 | Dockerfiles, CI jobs, per-environment URLs, test tokens, metrics pipeline |
  | 2 | v0.2 | 10–12 | env loading for the web app, the 503/500 rule, OIDC trust, ESM-only `pdfjs-dist`, CODEOWNERS |
  | 3 | v0.3 | about 8 | erasure racing a running analysis, Redis failures on caches, per-process env validation |
  | 4 | v0.9 | about 9 | new gaps opened by the v0.6–v0.9 additions: outage-test harness, fixture build hook, erase CLI start-up, criterion script rules |
  | 5 | v0.10 | about 6 | migration task image on deploy, log-scan false positives on IDs, header logging, JWKS refresh timing |

  The findings of each round were fixed in the next version.
- **Consistency audit** against `RUBRIC.md`: constants, enums, error codes, traceability links and JSON examples.
- **Rubric review (v0.6–v0.9).** I compared my spec with the other submitted specs at the level of ideas only, to find rubric areas where mine was weaker. No text was copied; every change was designed and written for this spec. For example, the no-invented-numbers rule is checked my own way (numbers outside `[…]` placeholders), and the missing-skill logic uses my own match-term catalog.
- **Script check** on every version: every JSON example parses, every table has a consistent column count, every referenced ID is defined, and every FR/NFR appears in the traceability matrix.

### Things the AI got wrong, which I corrected

- The first CSP (`script-src 'self'`) would have blocked Next.js hydration scripts. It now uses a per-request nonce (CR-022).
- The web health check pointed at `/`, which redirects. It now uses `/healthz` (CR-036).
- Build-time `NEXT_PUBLIC_*` values could not work in one image promoted to three environments. The web app now reads its config at runtime (§1.7).
- Erasing data reset the FREE monthly limit, and `used` meant two different things in two endpoints. Both are fixed (FR-007, FR-019).
- Erasure and a running analysis could release the same quota reservation twice. Erasure now deletes with `RETURNING`, and there is one lock order (§5.4).
- The missing-skill filter compared long descriptive phrases with the resume text and would almost never have matched. This was redesigned in v0.7.
- When Korean was added, one cache key (`ro:catalog:v1:ko`) was missed. This was fixed in v0.6.
- A SQL `||` inside a table cell broke the Markdown table. It is now escaped.

### Open items

- Every assumption in §3.3 names who must confirm it ("Confirm with"); none is confirmed yet.
- Round 1 submission: bump to v1.0 on 2026-10-20 after a last read-through.
