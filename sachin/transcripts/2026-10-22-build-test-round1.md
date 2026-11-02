# Build test, round 1 (spec v0.3 draft)

Exported transcript, 2026-10-22. Claude Code agent run on its own, reading only `sachin/spec.md`. Tool results (the spec text it read) are left out; each tool call is shown as one line.

## Prompt

```text
You are simulating the "build test" of a spec-writing homework. Read ONLY the file sachin/spec.md (do not read any other folder in spec-homework — other participants' folders are off limits; you may read RUBRIC.md for context). Do not edit any file.

Pretend you are Claude Code asked to build the backend API layer (Spring Boot) from this spec alone with no other context. Focus on section 6 (API design) and how it connects to sections 1–5 and 7. List every place where you would have to GUESS or ASSUME something the spec does not state, or where two parts of the spec contradict each other (e.g. a status code, field name, validation rule, error code, check order, response shape, header, or limit that differs between sections).

For each finding give: a short title, the exact location (section / ID / line text), what is missing or contradictory, and a concrete one-or-two-sentence fix (the exact wording or value to add). Rank by how likely a builder would guess wrong. Ignore purely cosmetic/style issues and anything outside the backend API (frontend styling, infra details) unless it directly affects an endpoint's behaviour. Be precise and terse; report at most 15 findings, most important first. If something is fine, don't mention it.
```

- *tool:* Read `spec.md`
- *tool:* Read `RUBRIC.md`
- *tool:* Read `spec.md` lines 453-902
- *tool:* Read `spec.md` lines 903-1352
- *tool:* Read `spec.md` lines 1353-1752
- *tool:* Read `spec.md` lines 1753-2022
## Final report

Build-test simulation of sachin/spec.md (v0.3). I read all 2013 lines and RUBRIC.md for context, and edited nothing. Section 6 is strong: check order, error envelope, pagination, per-endpoint error tables and ERR list are all present. Below are the 15 places where a builder would still have to guess, or would hit two sections that disagree. Most likely to go wrong comes first.

**1. Non-UUID `resumeId` in API-007 returns 400 in one place and 404 in three others**
- Where: API-007 request table (line 1608) says "valid UUID, else 400 `VALIDATION_FAILED` (`field: resumeId`)". ERR-015 (line 1921) says API-007 `resumeId` "not a UUID" gives 404 `RESUME_NOT_FOUND`. CR-014 (line 94) says the same 404 applies "wherever the id is sent". 6.1 (line 1360) says not-a-UUID means not found.
- Fix: In the API-007 table, write "valid UUID, else 404 `RESUME_NOT_FOUND` (CR-014); missing or blank → 400 `VALIDATION_FAILED` (`field: resumeId`)". Move the not-UUID case into the 404 row of the API-007 error table.

**2. The erasure "nothing stored" check: the diagram's query and the prose disagree**
- Where: 4.3.6 diagram (lines 674–677) decides "nothing stored" with `SELECT storage_key FROM resume WHERE user_id AND storage_key IS NOT NULL`. The prose (line 697) says "no resume, consent or counter row".
- Why it matters: if the diagram is followed, a user with only a consent row, or only EXPIRED resumes plus analyses, gets 202 and nothing is deleted. That breaks API-003, API-013 and API-014.
- Fix: In the diagram, replace the guard with "`NOT EXISTS` any `resume`, `consent` or `usage_counter` row for `user_id` → COMMIT, 202, no record". Then select the storage keys as a separate step (`storage_key IS NOT NULL`).

**3. The upload size limit is not fully configured, and "before the body is read" has no mechanism**
- Where: 4.7 `app.upload.max-bytes` (line 894) sets only `max-request-size`. API-004 (lines 1526 and 1532) says checks 1–3, including size, run before the body is read.
- Why it matters: Spring Boot's default `spring.servlet.multipart.max-file-size` is 1 MB. A builder following the spec literally rejects a 2 MB resume. It is also unstated how size is known before reading: is it `Content-Length`? What about chunked uploads? How does `MaxUploadSizeExceededException` map?
- Fix: Add "`spring.servlet.multipart.max-file-size` = `app.upload.max-bytes`; a `Content-Length` over max-bytes + 65,536 → 413 before reading; a chunked body or an oversized part → 413 when `MaxUploadSizeExceededException` is raised, mapped by `GlobalExceptionHandler`".

**4. FR-012 contradicts itself on which plan controls visibility**
- Where: FR-012 criteria 1–2 (line 318) say "analysis **started** on FREE / PREMIUM". Criterion 3 and API-009 (line 1731) use the reader's plan at read time.
- Why it matters: for an analysis started on PREMIUM and read later on FREE, criterion 2 says all suggestions and criterion 3 says ranks 1–3.
- Fix: Reword criteria 1–2 as "If the reader's current plan is FREE… / If the reader's current plan is PREMIUM…".

**5. Response field names in the section 4 diagrams differ from section 6, and no rule says which wins**
- Where: The 4.3.2 diagram (line 578) returns `202 {analysisId, status}`, but API-007 returns `id` plus five more fields. The 4.3.1 diagram (line 520) uses `size`, but API-004 uses `fileSizeBytes`.
- Fix: Change the diagrams to `{id, status, …}` and `fileSizeBytes`. Add to 6.1: "Section 6 is the authoritative wire contract; diagram labels are informal."

**6. The shape of `details[]` items clashes with the "never omit a field" rule**
- Where: 1.4 (line 116) and 6.1 (line 1398) put `resetAt` in the QUOTA_EXCEEDED item only. 6.1 (line 1368) says a field with an unknown value "is present with `null`, never left out".
- Why it matters: with one shared detail DTO, does a VALIDATION_FAILED item carry `"resetAt": null`?
- Fix: Add to 6.1: "`resetAt` appears only in the QUOTA_EXCEEDED detail item and is omitted elsewhere (the null-not-omitted rule applies to resource bodies, not to `details` items)."

**7. API-010's 404 list leaves out "older than 180 days"**
- Where: API-010 (line 1753) lists "no such id, another user's id, or not a UUID". ERR-017 (line 1923) applies "older than 180 days" to API-009, API-010 and API-011.
- Why it matters: deleting a 181-day-old row that has not been purged yet could return 204 or 404.
- Fix: Add ", or older than 180 days" to the API-010 404 row.

**8. The upload rate-limit hour has no time zone**
- Where: The Redis key `…:upload:<userId>:<yyyyMMddHH>` (line 920) and API-004 "`Retry-After` = seconds to the next hour" (line 1556).
- Why it matters: IST hours start 30 minutes off UTC hours. CR-025 (line 137) says the only IST logic is quota and retention, but `upload_rejection_daily` (line 1219), `analysis_outcome`, API-011's filename and API-015 all use IST too. So "only" is false and gives no guidance here.
- Fix: Add "rate-limit keys use UTC (`ZoneOffset.UTC`)" to CR-020. In CR-025, list every IST use (quota period, retention schedule, statistics day, report date).

**9. `Retry-After` for the 60-requests-per-minute limit is unspecified**
- Where: CR-020 (line 127) and 6.1 check 3 (line 1349). Only the upload limit defines its value.
- Fix: Add "`Retry-After` = seconds until the next UTC minute boundary, minimum 1".

**10. CORS headers and preflight are not defined**
- Where: CR-017 (line 124). The UI (`www`) and the API (`ro-api`) are on different origins.
- Why it matters: `X-Requested-With` is a non-simple header, so every POST or DELETE triggers an OPTIONS preflight. The browser also cannot read `X-Request-Id`, `Retry-After`, `Location` or `Content-Disposition` unless they are exposed.
- Fix: Add "Allowed methods GET, POST, DELETE; allowed headers `Content-Type`, `X-Requested-With`, `X-Request-Id`; exposed headers `X-Request-Id`, `Retry-After`, `Location`, `Content-Disposition`; `OPTIONS` preflight bypasses token, CSRF and rate-limit checks; max-age 600 s".

**11. The CSRF header rule says "present" in one place and "exact value" in another**
- Where: 6.1 check 2 (line 1348) says "header `X-Requested-With: SurfingBear` present". CR-018 (line 125) says requests "must carry" that header.
- Why it matters: would `X-Requested-With: XMLHttpRequest` pass?
- Fix: Write "value must equal `SurfingBear` exactly (case-sensitive); any other value or no header → 403 `FORBIDDEN`".

**12. API-002's 201 response may need a `Location` header**
- Where: 6.1 (line 1365) says 201 and 202 responses include `Location` "where a GET for it exists". `GET /api/v1/consents/current` exists, but API-002 (line 1498) does not mention the header.
- Fix: Add to API-002 either "201 includes `Location: /api/v1/consents/current`" or "no `Location` header".

**13. `erasure_record.notice_version` on withdrawal: current version or the version withdrawn?**
- Where: API-003 (line 1517) says `notice_version` = the current version. TBL-011 (line 1193) says "the version withdrawn".
- Why it matters: they differ when the user accepted only an older notice. If no consent row exists at all, the column is still NOT NULL by its CHECK.
- Fix: Write "`notice_version` = the user's most recent accepted version, or the current `app.privacy.notice-version` if none".

**14. Token claim edge cases have no status code**
- Where: CR-003 (line 53) and the 6.2 note (line 1426).
- The unstated cases:
  - What happens with `sub` missing, or `sub` over 64 characters?
  - What happens with a token whose `roles` lacks `USER`, such as a `PLATFORM_SERVICE` token on `/api/v1/**`: 401 or 403?
- Fix: Add "`sub` missing or over 64 characters → 401 `UNAUTHENTICATED`; on `/api/v1/**` a token without `USER` in `roles` → 403 `FORBIDDEN`".

**15. The `field` value is unspecified for several 400s**
- On API-004, a non-multipart request (for example JSON) could be 400 with `field: body`, 400 with `field: file`, or 415.
- On API-002 and API-013, an empty JSON body could name `body` or the missing field.
- On API-015 (line 1861), a `from > to` or over-90-day failure could name `from` or `to`.
- Fix:
  - API-004: "non-multipart request → 400 `VALIDATION_FAILED` (`field: file`)".
  - API-002 and API-013: "an empty body is treated as all fields missing".
  - API-015: "`from > to` and range over 90 days name `to`; a future `to` names `to`".

On the rubric's G scale this is 15 guesses, which falls in the 13–20 band (4 points). Items 1–7 are the ones a builder would most likely get wrong. Fixing those seven, plus 8 and 9, with the one-line edits above would bring the count down to about 6 (the 4–7 band, 12 points).
