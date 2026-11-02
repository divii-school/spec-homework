# Improvement round: writing and fixing the spec (v0.7 – v1.3)

Exported transcript, 2026-11-02. One Claude Code agent that wrote v0.7 and then fixed each build test's findings; it worked only in `sachin/`. The follow-up instructions are the build-test findings with the fix decided for each. Tool results (the spec text it read) are left out; each tool call is shown as one line.

## Prompt

```text
You are improving one Markdown spec for a scored homework: `sachin/spec.md` (SurfingBear Resume Optimizer, v0.6, ~2,700 lines, Spring Boot 3.4 / Java 21 backend, Next.js 15 frontend, PostgreSQL 16, Redis 7, Razorpay, AWS ap-south-1). It scored 96/100; this is the optional improvement round (deadline Wed 4 Nov 2026 23:59 IST). Today is Mon 2 Nov 2026.

## Hard rules
- Read and edit ONLY files inside `sachin\`. You may read `RUBRIC.md`, `BRIEF.md`, `PLAYBOOK.md`, `SCORES.md`. NEVER open any other participant's folder (copying scores zero) and do not grep the repo root recursively.
- IDs are permanent: never renumber or reuse; a removed item is marked **Deprecated**, not deleted. New IDs continue from the current maximum (CR-033 is the last CR, A-16 the last assumption, CMP-022, TBL-019, API-022, ERR-025, FR-030, NFR-021). Check the maxima yourself before adding.
- Every number/value lives once in 4.7 (configuration) and is referenced elsewhere; add new multi-place values to 7.3. Keep the spec's existing style: tables with ID | Area | Rule | How it is checked for CRs, plain direct English, *(v0.7)* markers on changed/added text like the existing *(v0.4)* / *(v0.6)* markers.
- Fix contradictions at the source; after each part, search the whole spec for old wording that the change makes stale (e.g. CR-001's cookie Domain, 1.7's "Local JWKS file + a dev token script", 4.4's `NEXT_PUBLIC_BILLING_FAKE` paragraph, NFR-005's "uptime check on /actuator/health/liveness", 4.5's Hosting/Observability rows, CR-011 log retention, A-10) and update it.
- Git: work in `spec-homework`. First `git pull --ff-only`. Commit after each part below with `git add sachin/` only and the message given (plain message, NO Co-Authored-By or any trailer, no mention of Claude/AI in the message). Do NOT push. Do not squash or amend.
- Mermaid: any diagram you add must parse. A checker exists: `node <scratchpad>\mmd\parse.mjs <file>` (look at it first to see its usage). ID checker: `python -I <scratchpad>\check_spec.py <file>` (read it first; known false positives: EXCELLENT/IGNORED/ORPHANED enum values). Run both at the end and fix what they report.

## What the scorer asked for (feedback, verbatim)
"Add a frontend deploy section (hosting, pipeline, env vars, 401 → platform login with return URL); name the logging/metrics/alert destinations, task sizes and IaC tool." The playbook checklist also requires: per-environment config (local/dev/staging/prod) written for frontend and backend; a change-log row for this round with affected IDs.

## Decisions already taken (write them in; each needs its reason in the spec)
Put platform-side facts as assumptions A-17.. with reasons. Put rules as new CRs (CR-034..) in sections 1.7/1.8 (or a new 1.9 "Infrastructure and operations" if cleaner), and put the detailed tables in a new section 4.8 "Deployment and operations" (subsections as you see fit: 4.8.1 environments and domains, 4.8.2 runtime configuration per environment, 4.8.3 frontend hosting, 4.8.4 pipeline, 4.8.5 infrastructure as code, 4.8.6 task sizes and scaling, 4.8.7 observability destinations and alert routing). Add CMP-023 "Web server" if the Next.js server needs its own component row (it does: it now runs as ECS service `ro-web`); update CMP-001 accordingly, the component diagram, the 2.2 frontend tree (add `src/middleware.ts`, `src/instrumentation.ts`, `src/app/api/health/route.ts` or the basePath equivalent, `Dockerfile`), and the traceability rows.

1. **Domains per environment (assumption).** prod: web `https://www.surfingbear.com/resume-optimizer`, API `https://ro-api.surfingbear.com`, platform cookie Domain `.surfingbear.com`. staging: `https://www.surfingbear-staging.com/resume-optimizer`, `https://ro-api.surfingbear-staging.com`, cookie `.surfingbear-staging.com`. dev: same pattern with `surfingbear-dev.com`. local: web `http://localhost:3000/resume-optimizer`, API `http://localhost:8080`, host-only cookie on `localhost` (cookies ignore the port; Chrome/Firefox treat http://localhost as a secure context, so `Secure` works). Reason: separate registrable domains keep each environment's `sb_access` cookie apart; a shared `.surfingbear.com` would let dev and prod overwrite each other's cookie. CR-001's cookie Domain and CR-017's CORS origin list become per environment (new property `app.cors.allowed-origins` / `CORS_ALLOWED_ORIGINS`, values per env in the config table).
2. **Platform contract per environment (assumption).** Issuer `https://auth.surfingbear.com` (prod), `https://auth.surfingbear-staging.com`, `https://auth.surfingbear-dev.com`; JWKS at `<issuer>/.well-known/jwks.json`; login page `https://www.<domain>/login` accepting `?returnTo=<path>`, where returnTo must be a relative path on the same host (the platform rejects anything else and lands on its home page); after login the platform redirects to returnTo. The platform also owns the CloudFront distribution for `www.<domain>` and adds the behaviour for `/resume-optimizer*` pointing at the module's origin (it gets the origin host name and the origin secret from the module team).
3. **Local login (replaces "Local JWKS file + a dev token script").** In `local` only (`@Profile("local")`, bean absent otherwise, and the task refuses `APP_ENV=local` against... keep it simple: the controller exists only in the local profile), the API serves `GET /local/login?returnTo=<path>&userId=<sub, default local-user-1>&admin=<true|false>` which signs an RS256 token with the local key pair `config/local-jwt/private.pem` (issuer `http://localhost:8080/local`, aud `resume-optimizer`, roles USER [+ADMIN], exp 8 h), sets cookie `sb_access` (host-only, Path=/, HttpOnly, SameSite=Lax, Secure) and 302s to `http://localhost:3000` + returnTo; and `GET /local/jwks.json` serving the public key. Local `PLATFORM_ISSUER=http://localhost:8080/local`, `PLATFORM_JWKS_URL=http://localhost:8080/local/jwks.json`, web `PLATFORM_LOGIN_URL=http://localhost:8080/local/login`. `scripts/dev-token.sh` stays for curl and for `PLATFORM_SERVICE` tokens (FR-021 tests) using the same key. Make sure `/local/**` is excluded from the JWT filter in local and does not exist elsewhere.
4. **Frontend hosting.** Next.js 15 with `output: 'standalone'`, `basePath: '/resume-optimizer'`, Node 22 LTS image (`node:22-alpine`), one Docker image `ro-web` per commit, run as ECS Fargate service `ro-web` on port 3000 behind the module's ALB. CDN path: platform CloudFront behaviour `/resume-optimizer*` → origin `ro-web-origin.<domain>` (an ALB host rule). The ALB forwards to `ro-web` only if the request carries header `X-Origin-Verify` equal to the secret `RO_WEB_ORIGIN_SECRET` (CloudFront adds it as a custom origin header), otherwise fixed 403 — so the origin cannot be used to bypass the CDN. Caching: `/resume-optimizer/_next/static/*` cached by the CDN for 1 year (Next.js content-hashes them and sends immutable); every HTML response sends `Cache-Control: private, no-store` (set in middleware), so no page is cached by the CDN. Health: `GET /resume-optimizer/api/health` → 200 `{"status":"UP"}`, used by the ALB target group (interval 15 s, healthy 2, unhealthy 3). The web server never calls the API from the server side; all API calls are made by the browser to the API host with `credentials: 'include'` (state why: the token is checked in one place, the API, and the web tier holds no secrets except the origin secret).
5. **Runtime configuration for the web (one image for every environment, CR-026).** No `NEXT_PUBLIC_*` values in deployed images. The root layout is a server component (`export const dynamic = 'force-dynamic'`) that reads at request time `APP_ENV`, `RO_API_BASE_URL`, `PLATFORM_LOGIN_URL`, `WEB_BILLING_FAKE` (default `false`) and, only when billing is fake, `WEB_FAKE_RAZORPAY_KEY_SECRET`; it passes them to a client `RuntimeConfigProvider`. `src/instrumentation.ts` `register()` fails startup if `APP_ENV` is missing or not one of the four values, if `RO_API_BASE_URL`/`PLATFORM_LOGIN_URL` are missing, or if `WEB_BILLING_FAKE=true` with `APP_ENV` ≠ `local`. This **deprecates** `NEXT_PUBLIC_BILLING_FAKE` and `NEXT_PUBLIC_FAKE_RAZORPAY_KEY_SECRET` in 4.4 (replace the text, mark the old names deprecated in the change log). Playwright e2e in CI runs against the docker compose stack with `APP_ENV=local`; the staging Playwright run (FR-024 trace row) uses real Razorpay Test Mode.
6. **401 handling (CR).** (a) `src/middleware.ts`: a page request under `/resume-optimizer` with no `sb_access` cookie → 307 to `PLATFORM_LOGIN_URL?returnTo=<encodeURIComponent(pathname + search)>` (pathname includes the basePath). The middleware does not validate the token. (b) The typed API client in `src/lib/api`: any API response 401 with `error.code` `UNAUTHENTICATED` → `window.location.assign(loginUrl + '?returnTo=' + encodeURIComponent(location.pathname + location.search))`; in-flight requests are dropped; nothing else in the UI handles 401. (c) Loop guard: before redirecting, read `sessionStorage['ro:auth-redirect-at']`; if a redirect happened less than 60 s ago, do not redirect again: show a full-page message "We could not confirm your sign-in." with a "Sign in again" button that clears the key and redirects; otherwise write the current time and redirect. Wrap sessionStorage access in try/catch (if unavailable, redirect). (d) 403 and every other code are not redirects. (e) returnTo is always a relative path starting with `/resume-optimizer`. Give the rule an ID and a check (Vitest on the client; Playwright with an expired cookie).
7. **Pipeline (CR + 4.8.4).** GitHub Actions in three repos: `resume-optimizer-api`, `resume-optimizer-web`, `resume-optimizer-infra`. AWS access by GitHub OIDC (role `ro-ci-deploy-<env>`), no stored AWS keys. On every merge to `main`: build, test (CR-028), push image to ECR repository `ro-api` / `ro-web` in the shared `sb-tools` account, tag = full git SHA, tag immutability on; then deploy to dev automatically. Staging and prod: `workflow_dispatch` with input `image_sha`, GitHub Environment `staging` / `prod` with one required reviewer (tech lead); the job refuses a SHA that has not been deployed to the previous environment (checks the running task definition image there). API deploy steps, in order: (1) run the one-off migration task (same image, `APP_MODE=migrate`: runs Flyway as `ro_owner` then exits 0; non-zero exit stops the pipeline) — add `migrate` to `app.mode` in 4.7 and reconcile with 4.4's existing "Flyway runs as ro_owner in a one-off ECS task" and add the owner secret; (2) update `ro-api`, wait for steady state (max 15 min); (3) update `ro-worker`, wait; (4) smoke: `GET https://<api host>/api/v1/job-roles` without a cookie must return 401 with `error.code` `UNAUTHENTICATED`. Web deploy: update `ro-web`, wait, smoke `GET https://www.<domain>/resume-optimizer/api/health` (through the CDN) = 200. ECS deployment circuit breaker with automatic rollback on for all three services. When an API change and a web change belong together, API first (CR-030 keeps the old web working). Reconcile with CR-028/CR-029/CR-031 (rollback by redeploying the previous SHA stays).
8. **Infrastructure as code (CR + 4.8.5).** Terraform 1.9 with the AWS provider ~> 5.70 in repo `resume-optimizer-infra`: `modules/` (alb, ecs-service, rds, redis, s3, observability, secrets) and `envs/{dev,staging,prod}`. One AWS account per environment (`sb-dev`, `sb-staging`, `sb-prod`) plus `sb-tools` for ECR. State: S3 bucket `sb-tfstate-<env>` (versioned, encrypted) key `resume-optimizer/terraform.tfstate`, lock table DynamoDB `sb-tf-locks`. `terraform plan` runs on every PR and is posted as a PR comment; `apply` only by `workflow_dispatch` through the same GitHub Environments. The module deploys into the platform's existing VPC per environment (private subnets for tasks, RDS and Redis; public subnets for the ALB) — VPC and subnet ids are Terraform inputs (assumption). Not in Terraform, done once by hand and listed: the Razorpay plan (dashboard, id into `envs/<env>/terraform.tfvars` as `razorpay_plan_id`), secret VALUES (Terraform creates empty Secrets Manager entries, ops set values), the platform's CloudFront behaviour, DNS for `ro-api.<domain>` and `ro-web-origin.<domain>` if the zone belongs to the platform (say who). Local is docker compose only.
9. **Task sizes and scaling (4.8.6).** Sizes are per task, including the observability sidecar. `ro-api` 1 vCPU / 2 GB, `ro-worker` 0.5 vCPU / 1 GB, `ro-web` 0.25 vCPU / 0.5 GB, migration task 0.5 vCPU / 1 GB. JVM `-XX:MaxRAMPercentage=75`. Desired/min/max — dev: api 1/1/2, worker 1/1/2, web 1/1/2; staging: api 2/2/4, worker 2/2/10, web 2/2/4; prod: api 2/2/6, worker 2/2/10, web 2/2/4. api and web: target tracking on average CPU 60 %. worker: step scaling on CloudWatch metric `ResumeOptimizer/<env>` `AnalysisBacklog` (count of `QUEUED` analyses, published once a minute by the sweeper CMP-012 with `cloudwatch:PutMetricData` on the worker task role): ≥ 20 for 2 consecutive minutes → +2 tasks, ≥ 100 → +4, = 0 for 15 minutes → −1 (never below min). Check 10 × `WORKER_CONCURRENCY` 5 = NFR-006's 50. RDS PostgreSQL 16: dev `db.t4g.micro` single-AZ 20 GB; staging and prod `db.t4g.medium` Multi-AZ, 100 GB gp3 with storage autoscaling to 500 GB, backups 7 days (PITR, NFR-005 RPO). Check connection budget against HikariCP pools (20 per api task, 10 per worker task, see 4.4): prod max 6×20 + 10×10 = 220 < the instance's default `max_connections` (state it roughly, ~400 for 4 GiB); dev 2×20 + 2×10 = 60 < ~100 for micro. ElastiCache Redis 7: dev `cache.t4g.micro` one node; staging and prod `cache.t4g.small` primary + 1 replica, Multi-AZ automatic failover, TLS and AUTH (as 4.4). Staging uses prod sizes (1.7 already says so).
10. **Observability destinations (CR + 4.8.7; update CR-011, NFR-005, NFR-013, NFR-014, 4.5, 7.2).** Logs: stdout JSON (CR-008) → ECS `awslogs` driver → CloudWatch Logs groups `/ro/<env>/ro-api`, `/ro/<env>/ro-worker`, `/ro/<env>/ro-web`, `/ro/<env>/ro-migrate`, retention 30 days (CR-011), KMS-encrypted; searched with Logs Insights. The web server logs JSON with pino, same fields as CR-008 where they apply, and CR-009 applies to it too. Browser-side errors are not collected (no third-party error SDK on pages that hold personal data and payment checkout) — say so in out of scope. Metrics: Micrometer Prometheus endpoint on the management port 8081 (`/actuator/prometheus`, not routed by the ALB) scraped every 30 s by an AWS Distro for OpenTelemetry (ADOT) collector sidecar in each `ro-api` and `ro-worker` task (128 CPU units / 256 MB of the task size), remote-written to an Amazon Managed Service for Prometheus workspace `ro-<env>` with labels `env`, `service`, `task_id`. Dashboards: Amazon Managed Grafana workspace `surfingbear-observability`, one dashboard `Resume Optimizer` per env data source, defined as JSON in the infra repo. Traces (NFR-014): OpenTelemetry Java agent → OTLP to the same sidecar (localhost:4317) → AWS X-Ray; sampling 100 % in dev and staging, 10 % in prod (parent-based); `requestId` is a span attribute. Alerts: Prometheus alerting rules in the AMP rule namespace `ro-alerts` (YAML in the infra repo), routed by AMP Alertmanager by severity to SNS topics `ro-<env>-alerts-critical` and `ro-<env>-alerts-warning`; SNS → AWS Chatbot → Slack `#ro-alerts-prod` (prod) / `#ro-alerts-nonprod` (dev, staging); prod critical also → PagerDuty service "Resume Optimizer" through its SNS HTTPS integration. Non-prod never pages. Write ONE alert table (name, PromQL or CloudWatch condition, for, severity, destination, requirement) covering every alert the spec already promises (NFR-011 retention violations, NFR-012 storage_deletion/subscription_cancellation rows older than 24 h — define the gauges that make it measurable, NFR-013's list including billing ones, JWKS fetch failures from 4.4, rate limiter unavailable from 4.4, NFR-015 AI spend if it is an alert) and make NFR-013 point to it instead of restating numbers (keep numbers in one place). Uptime (NFR-005): a CloudWatch Synthetics canary `ro-<env>-uptime` every minute calls `GET https://<api host>/api/v1/job-roles` without a cookie and passes on HTTP 401 with `error.code` `UNAUTHENTICATED` (proves DNS, TLS, ALB, task and filter chain; the management port is not public); availability = canary success rate per calendar month; canary failing 3 runs in a row → critical alarm to the same SNS topic. Update NFR-005 and its 7.2 row.
11. **Per-environment configuration table (4.8.2).** One table, one row per environment variable for backend AND frontend (every variable in 4.7 plus the new ones), columns: variable | service(s) | local | dev | staging | prod | source (task definition env via Terraform / Secrets Manager secret name / docker compose). Secret names: `ro/<env>/db-app` (username+password of `ro_app`), `ro/<env>/db-owner` (`ro_owner`, migration task only), `ro/<env>/redis`, `ro/<env>/anthropic`, `ro/<env>/razorpay` (JSON keys `key_secret`, `webhook_secret`), `ro/<env>/web-origin-verify`. Local values come from `docker-compose.yml` and a git-ignored `.env` created from a committed `.env.example`. Local S3 is LocalStack: if the spec has no S3 endpoint property yet, add `app.storage.endpoint` / `S3_ENDPOINT` (empty = AWS default; `http://localhost:4566` locally, path-style on). Non-secret values that are per-env inputs (RDS/Redis endpoints) come from Terraform outputs wired into the task definition. Keep 4.7 as the single place for defaults and add a sentence that 4.8.2 gives the per-environment values.

## Steps and commits
Part A (commit "improvement: environments and domains, platform contract per environment, local login (A-17 – A-19, CR-001, CR-017)" or similar accurate message): items 1–3 and the stale-text cleanup they cause.
Part B (commit "improvement: frontend hosting, runtime configuration and 401 redirect to platform login (CMP-023, CR-0xx)"): items 4–6.
Part C (commit "improvement: deploy pipeline, Terraform, task sizes and scaling (CR-0xx, 4.8.4 – 4.8.6)"): items 7–9.
Part D (commit "improvement: log, metric, trace and alert destinations; uptime canary; one alert table (NFR-005, NFR-013, NFR-014)"): item 10.
Part E (commit "improvement: configuration per environment for backend and frontend (4.8.2)"): item 11.
Part F (commit "improvement: v0.7 change-log row, traceability and shared values updated; consistency checks clean"): header Version v0.7 and Last updated 2026-11-02; 7.1/7.2 rows for anything new (new CRs are not in the matrix unless the matrix lists CRs — follow the existing pattern); 7.3 rows for new multi-place values (e.g. 30-day log retention, 60 s redirect guard, task counts if repeated); change-log row v0.7 with what / why (the scorer's feedback and playbook gaps) / affected IDs (list every ID and section touched; mark deprecated names). Run both checkers and the stale-wording search; fix and commit.

Be exact and concrete everywhere: a builder with no other context must be able to configure every environment from the spec alone. Do not invent features beyond these decisions; where a detail is needed to make a decision buildable and is not given above, choose the simplest option, write it with its reason, and list it in your final report.

## Final report (your last message)
- the commit SHAs and messages;
- every new ID with a one-line description;
- every place you changed because of stale wording;
- any detail you had to choose yourself beyond the decisions above;
- checker results (paste the summary lines).
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1 | tail -3; git status --short | head; git log --oneline -5; ls sachin; wc -l sachin/spec.md`
- *tool:* Bash `cat <scratchpad>/mmd/parse.mjs | head -60; echo ----; c…`
- *tool:* Bash `cd /c/Users/user/spec-homework && ls; git log --oneline -3 -- sachin/; grep -n '^#' sachin/spec.md`
- *tool:* Bash `cd /c/Users/user/spec-homework && cat RUBRIC.md; echo ======; cat PLAYBOOK.md | head -200`
- *tool:* Read `spec.md` lines 1-300
- *tool:* Read `spec.md` lines 300-549
- *tool:* Bash `cd sachin && grep -n -i -E 'actuator|health|liveness|readiness|prometheus|micrometer|cloudwatch|alert|NEXT_PUBLIC|cookie|CORS|origin…`
- *tool:* Read `spec.md` lines 548-787
- *tool:* Read `spec.md` lines 786-1015
- *tool:* Read `spec.md` lines 1150-1209
- *tool:* Read `spec.md` lines 1732-1843
- *tool:* Read `spec.md` lines 2560-2699
- *tool:* Bash `cd sachin && grep -n -o -E '.{100}(Playwright|e2e|NEXT_PUBLIC|8081|8080|management|spring.profiles|application-<env>|ro_owner|ro_app…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{120}(temurin|distroless|JAVA_TOOL|javaagent|otel|image|Docker image|healthCheck|container).{100}' spec…`
- *tool:* Bash `cd sachin && grep -n -o -E '`[a-z_]+(_total|_seconds)(\{[^}]*\})?`' spec.md | sort -t: -k2 -u | sort -n`
- *tool:* Bash `cd sachin && grep -n -o -E '.{80}(plan_[A-Za-z0-9]+|GRANT|ro_owner|CREATE ROLE|BILLING_PROVIDER|AI_PROVIDER|ANTHROPIC_MODEL).{60}' s…`
## Agent

Now Part A edits. Starting with CR-001, CR-017 and the new CR-034.

- *tool:* Edit `spec.md`
- *tool:* Edit `spec.md`
- *tool:* Edit `spec.md`
- *tool:* Bash `cd sachin && python - <<'EOF' import io p='spec.md'; s=open(p,encoding='utf-8').read() a="Domain = the environment's cookie domain: …`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() anchor="| `ro:<env>:queue:analysis` | stream, group `ro…`
- *tool:* Bash `cd sachin && awk '/^#### API-014/,/^#### API-015/' spec.md | head -30`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && sed -i 's/this module'"'"'s Next.js service, CMP-023); the API/this module'"'"'s Next.js service); the API/' spec.md &&…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/ && git commit -q -m "improvement: environments and domains, platform contract per environment, local sign-in (…`
## Agent

Part A committed. Now Part B (frontend hosting, runtime config, 401 handling).

- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
## Agent

Now CMP-001, CMP-023, the component diagram and A-10.

- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() anchor="Integration tests do not use this key: each tes…`
- *tool:* Bash `cd sachin && python -I <scratchpad>…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/ && git commit -q -m "improvement: frontend hosting, runtime configuration and 401 redirect to platform login (…`
## Agent

Part B committed. Now Part C: pipeline, Terraform, sizes. First the CRs, A-19, and run-mode changes.

- *tool:* Bash `cd sachin && python - <<'EOF' p='spec.md'; s=open(p,encoding='utf-8').read() def rep(a,b):     global s     assert s.count(a)==1,(s.…`
- *tool:* Bash `cd sachin && grep -n 'Infrastructure-as-code review' spec.md | cut -c1-80`
## Agent

I'll save the script to a file so I can re-run it with the fix.

- *tool:* Bash `mkdir -p <scratchpad> && cd /c/Users/user/spec-homework…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
## Agent

The command is too long for the shell; I'll write the edit script with the Write tool.

- *tool:* Write `c2.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{60}(razorpay_calls_total|outcome\b|jwks_fetch|ratelimit_unavailable|retention_violations_total).{80}' …`
## Agent

Part C committed. Now Part D (observability). Writing the edit script to a file.

- *tool:* Write `d1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Write `d2.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && awk '/^### 4.7 Configuration/,/^Redis keys/' spec.md | grep '^|' | cut -d'|' -f2-4 | cut -c1-200`
## Agent

Part D committed. Now Part E: the per-environment configuration table.

- *tool:* Write `e1.py`
- *tool:* Edit `e1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{70}(liveness|[Uu]ptime check|spend alarm|same host|load balancer listener|src/app/resume-optimizer|dev…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{60}surfingbear\.com.{40}' spec.md | awk -F: '$1<1240 || $1>1300' | cut -c1-200`
- *tool:* Write `f1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/ && git commit -q -m "improvement: v0.7 change-log row, traceability and shared values updated; consistency che…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git branch --show-current && git status --short | head`
- *tool:* Agent: Build-test review of v0.7 deploy sections (separate build-test agent, transcript exported on its own)
## Agent

The review found 25 items; most are real. Fixing them at their sources now.

- *tool:* Bash `cd sachin && grep -n -o -E '.{0,40}(GitHub Environments `dev`|ro-ci-ecr-pull` \| `sb-tools`|ro-ci-tf-plan-<env>` \||random values of…`
- *tool:* Bash `cd sachin && for n in 1430 1441 1457 1487 1493 1501 1516 1591 1632; do sed -n "${n}p" spec.md; echo; done | cut -c1-1500; grep -n 'C…`
- *tool:* Write `g1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git log --format='%h %an %s' -8 && git status --short && git log -1 --format=%B | tail -3 && wc -l sachin/spec.md`
## Final report

## Result
The spec is now v0.7 (updated 2026-11-02, 2,698 → 3,166 lines). It covers everything the scorer asked for: frontend hosting, the pipeline, variables per environment for backend and frontend, the 401 redirect to the platform login with a return URL, named log/metric/alert destinations, task sizes, and Terraform as the IaC tool. All 7 commits are on `main` in `spec-homework`, staged with `git add sachin/` only, plain messages with no trailers, and **not pushed**. I edited only `sachin/spec.md`. I did not touch `process.md`; it has no entry for this round yet (it carries points in area I).

## Commits
1. `67af97f` improvement: environments and domains, platform contract per environment, local sign-in (A-17, A-18, CR-034, CR-001, CR-017, 4.8.1)
2. `cea4c8d` improvement: frontend hosting, runtime configuration and 401 redirect to platform login (CMP-023, CR-035 - CR-037, 4.8.3)
3. `e843e71` improvement: deploy pipeline, Terraform, task sizes and scaling (A-19, CR-038, CR-039, CR-029, CR-031, 4.8.4 - 4.8.6)
4. `ac901a9` improvement: log, metric, trace and alert destinations; uptime canary; one alert table (CR-040, A-20, NFR-005, NFR-013, NFR-014, 4.8.7)
5. `a51ef13` improvement: configuration per environment for backend and frontend (4.8.2)
6. `fc086d5` improvement: v0.7 change-log row, traceability and shared values updated; consistency checks clean
7. `59a4c79` improvement: build-test fixes to v0.7 deployment and operations (OIDC subjects, ECR check, plan secrets, counters at zero, HSTS, worker stop, X-Ray index)

Commit 7 is extra. After part F, a fresh agent read only `sachin/spec.md` as a build test of the v0.7 parts. It logged 25 items (4 high, 11 medium, 10 low). All 25 are fixed, and the v0.7 change-log row says so.

## New IDs
- **CR-034** Local sign-in: `/local/login` and `/local/jwks.json`, `@Profile("local")` only; 404 in every other environment.
- **CR-035** 401 handling:
  - middleware sends 307 to the login page with `returnTo` when there is no cookie;
  - the API client is the only code that handles a 401 `UNAUTHENTICATED`;
  - a 60 s `sessionStorage` loop guard shows a "Sign in again" page;
  - 403 and other codes are not redirects; `returnTo` always starts with `/resume-optimizer`.
- **CR-036** One web image: no `NEXT_PUBLIC_*`; values read at request time; `instrumentation.ts` refuses to start on bad values.
- **CR-037** Web origin locked: only requests with `X-Origin-Verify` pass, otherwise 403; HTML is `private, no-store`; only `_next/static` is cached.
- **CR-038** Pipeline: GitHub OIDC, full-SHA images, dev deploys on merge, staging and prod by `workflow_dispatch` with a reviewer and a promotion check, migration first, smoke checks, API before web.
- **CR-039** Infrastructure as code: Terraform for everything except a listed set of manual steps.
- **CR-040** Observability destinations: CloudWatch Logs (30 days), AMP, X-Ray, SNS → Slack, PagerDuty for prod critical only.
- **A-17** Domains per environment. **A-18** Platform contract per environment (issuer, JWKS, login page, CloudFront, `ro-internal` host). **A-19** AWS accounts, the platform's VPC, DNS zones, GitHub org, test accounts. **A-20** Slack, PagerDuty, Grafana.
- **CMP-023** Web server, ECS service `ro-web`.
- New sections: 1.9, and 4.8 with 4.8.1 – 4.8.7. No new FR, NFR, TBL, API or ERR. The `/local/**` endpoints have no API ID on purpose; they exist only locally.

## Places changed because of stale wording
- CR-001, CR-017, A-1, A-10: domains and origins now come from the per-environment table.
- 1.7: the "Local JWKS file + a dev token script" row, plus a pointer to 4.8.
- 4.4: the `NEXT_PUBLIC_BILLING_FAKE` paragraph (both old names marked Deprecated, also in 4.7), and the PostgreSQL, JWKS, Redis and Anthropic rows.
- NFR-005: the liveness uptime check was on a non-public port; it is now a canary. Its 7.2 row too.
- 4.5: Hosting, Observability, Frontend and Run modes rows ("API by request rate" now measured as CPU).
- CR-011 log retention; CR-008 and CR-009 now cover the web server.
- NFR-006, NFR-010, NFR-011, NFR-012, NFR-013 (thresholds now live only in the alert table), NFR-014, NFR-015 ("spend alarm" became the workspace spend limit), NFR-016 and NFR-018 rows in 7.2.
- 4.3.7: the 24-hour overdue rows moved to per-minute metrics.
- 2.2: pages moved to `src/app/`; under `basePath` the old folder would have served `/resume-optimizer/resume-optimizer`. Backend tree and compose comment updated.
- 4.7 `app.env`: the bucket now comes from `S3_BUCKET`. `app.mode` gains `migrate`.
- 6.1: internal host, `/local/**`, `/actuator/prometheus`, and the "no other HTTP endpoints" line.
- 4.3.9 webhook URL per environment; CR-016 HSTS; CR-029 and CR-031; CMP-001 and CMP-012; the section 4 intro; 7.1 note; 7.3 (12 new rows, plus a note that infrastructure values live in 4.8).

## Details I chose myself (each has its reason in the spec)
- **Network and load balancers:**
  - The internal API moves to a separate internal ALB at `ro-internal.<domain>`.
  - The public ALB returns an empty 404 for `/internal/*`.
  - ALB rule order and the `ELBSecurityPolicy-TLS13-1-2-2021-06` TLS policy.
  - API target groups health-check `/actuator/health/readiness` on port 8081 (15 s / 2 / 3).
  - HSTS is set on the ALB listener.
- **CDN and origin secret:**
  - Two CloudFront behaviours: `CachingOptimized` for static files; `CachingDisabled` with `AllViewerExceptHostHeader` for the rest.
  - The origin secret is JSON `{current, next}` so it can be rotated without refusing a request.
- **Local and frontend:**
  - The local key pair is git-ignored and created at first start, because gitleaks would block a committed private key.
  - `/local/login` has defaults and validation for `userId` and `admin`; there is a `LOCAL_LOGIN_REDIRECT_ORIGIN` property.
  - Next.js must be 15.5 or later, with middleware on the Node runtime.
  - Playwright on the local stack runs Chromium and Firefox; WebKit runs on staging.
  - CI Playwright uses the API image SHA in `e2e/api-image-sha.txt`.
  - Staging Playwright signs in with platform test accounts, stored as Environment secrets.
- **Database:**
  - `migrate` mode creates or updates `ro_app` with its password, then runs Flyway.
  - New variables `DB_OWNER_USERNAME` / `DB_OWNER_PASSWORD`.
  - `sslmode=require`; database `resume_optimizer`, with `ro_owner` as the RDS master user.
  - Dev RDS keeps backups for 1 day; staging and prod have deletion protection.
- **Images and Terraform:**
  - The API base image is `eclipse-temurin:21-jre`.
  - Terraform task definitions use `track_latest`, and an SSM image-SHA parameter that the pipeline writes.
  - An `envs/tools` state for ECR; GitHub org `surfingbear`.
  - Role names for ECR push/pull/read, promotion read and Terraform plan/apply.
  - A three-pass first apply (secrets and certificates, then values and DNS records, then the rest).
  - An apply that changes both API and worker runs in two steps, to keep the connection budget.
  - A rollback skips the promotion check and the migration, and only accepts a SHA used before.
  - Grafana tokens are minted per run, because they expire after 30 days.
- **Workers:** no container health checks. Graceful worker stop: `stopTimeout` 120 s and a 110 s drain.
- **Metrics:**
  - The sweeper also publishes `StorageDeletionOverdue` and `CancellationOverdue` as CloudWatch metrics. A Prometheus gauge would go stale on tasks that don't hold the lock.
  - New counters `analyses_finished_total{status}` and `api_errors_total{code}`. All counters start at 0.
  - Label values defined for `razorpay_calls_total`.
- **Alerts:**
  - Severities chosen for every alert.
  - Minimum event counts on the three ratio alerts.
  - Extra alerts `WebUnavailable` and `ProHeavyUse`.
  - The AI spend limit becomes the environment's Anthropic workspace limit, with no separate alert.
  - PagerDuty has its own topic, `ro-prod-alerts-page`.
  - The Chatbot custom-notification template for Alertmanager.

## Checker results (final run)
- ID checker: CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 25, A 20 defined.
  - `error codes listed: 25 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`. The component diagram now has the CloudFront node, the `ro-web` service and the login-redirect edge.
- Stale-wording search: no `NEXT_PUBLIC_*` left outside the deprecation notes, no "Local JWKS file", and no other liveness-check mention except the note on NFR-005 explaining why it was replaced.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
An independent build test of v0.7 (whole spec, the scorer's playbook prompt) found 16 items. Fix all of them at their source as v0.8, same rules as before (only sachin/, IDs never renumbered, values once in 4.7, stale-wording search, both checkers clean, commit locally with plain messages, NO trailers, do NOT push). Decisions per item:

1. CONTRADICTION FR-008 c1 vs c7 / 4.2 diagram / 4.3.4: c1 allows only QUEUED→RUNNING→COMPLETED|FAILED, but the sweeper needs QUEUED→FAILED. Fix c1: add "or `QUEUED → FAILED` (sweeper only, `AI_TIMEOUT`, criterion 7)". Check the 4.2 transition sentence too.
2. GUESS 4.3.9 / FR-029 c8 / 4.3.8 `last_event_at`: verification (API-018) and reconciliation change `status` without setting `last_event_at`, so a stale redelivered `pending` older than the charge can move a paid row back to PENDING, and the Pro page would offer an immediate cancel. Fix: whenever verification or reconciliation applies a status change, set `last_event_at` = the time of the Razorpay fetch (UTC now at fetch); events whose payload `created_at` is older are then skipped for status (payment + GREATEST still apply). Update TBL-016 column note, 4.3.8, 4.3.9, FR-029 c5/c8 and the test rows.
3. GUESS what each APP_MODE starts: state per mode, in 4 intro or 4.7 `app.mode`: `api` = servlet server on 8080 (controllers, request pipeline CMP-002, webhook) + management on 8081; `worker` = no API controllers, no JWT/CORS/CSRF/rate-limit beans, servlet server only for management on 8081 (`server.port` 8081, actuator only, every other path 404), queue consumer + scheduled jobs; `migrate` = `spring.main.web-application-type=none`, Flyway (and the ro_app role step) then exit. Name the condition (`@ConditionalOnProperty(name="app.mode", havingValue=...)` on each bean group) and say which env vars each mode requires, consistent with 4.8.2.
4. GUESS local host run: Spring reads neither docker-compose.yml nor .env on `./mvnw spring-boot:run`. Fix: all local values (secret and non-secret) live in `.env` copied from committed `.env.example`; the host run is `scripts/run-local.sh`, which does `set -a; . ./.env; set +a` and then `./mvnw spring-boot:run` (so they are real environment variables, which also makes `APP_ENV`→profile work). docker compose only runs postgres/redis/localstack. Update 4.8.2 "where values come from", 1.7 local row, 2.2 tree (scripts/), anywhere else that says otherwise.
5. GUESS CMP-021 cancellation retry after a 4xx: define the outcome per fetched status: `cancelled`, `completed`, `expired`, `halted` → done (row closed; halted subscriptions are not charged again); `created` → done if `expire_by` has passed, else retry next run; `authenticated`, `active`, `pending` → retry next run (the 24 h overdue alert then pages, which is intended: money may move); fetch 404 → done. Put it in one place (4.4 Razorpay row or a small table in 4.3.6) and reference it.
6. GUESS staging load tests: add `app.ai.fake-latency` / `AI_FAKE_LATENCY` to 4.7 (default 0) and 4.8.2. NFR-006's staging run: Terraform variable `load_test_mode` (default false) in envs/staging sets `AI_PROVIDER=fake` and `AI_FAKE_LATENCY=2s` for the duration of the run, applied and reverted through the normal apply workflow; it is refused in prod (validation in the variable). The fake-AI guard must allow staging only when that mode is on — check what the spec currently says about AI fake guards and make it consistent. k6 tokens: assumption (extend the platform assumption) that the platform's staging admin tool issues 24-hour user tokens for 25 test accounts `ro-load-01`…`ro-load-25`; k6 sends each as the `sb_access` cookie, 20 req/s spread over the 25 users stays under 60/min per user. State that NFR-001 k6 also uses those accounts.
7. GUESS HikariCP pool per mode: add `app.db.pool-size` / `DB_POOL_SIZE` to 4.7 (no default; required) with values per service in 4.8.2: api 20, worker 10, migrate 2; 4.4 and 4.8.6 reference it.
8. GUESS FR-030 c3: plan PRO with `subscription: null` (a late charge on an EXPIRED/ORPHANED row raised `access_until`). Add a page state, placed before (b): "Resume Pro until <accessUntil>", no buttons. Keep the state letters stable (use (b0) or (a3) style, don't reletter existing ones).
9. GUESS FR-001 / A-14 notice text: give the exact English text of the privacy notice (purpose, data, AI processing by Anthropic outside India on redacted text, retention 30/180 days, rights, how to withdraw/erase, grievance contact) and of the Pro payment notice, as `messages/en.json` keys with values, in a new subsection (e.g. 3.4 "User-facing notice texts"). Grievance contact: "Grievance Officer, SurfingBear, grievance@surfingbear.com, replies within 30 days" (mark it as an assumption to be confirmed by legal). Reference numbers from 4.7 by name rather than repeating literals where the text must show them, or note they must match 4.7.
10. GUESS FR-022 admin screen: no UI in this module; admins read API-015 directly (JSON) — say it in FR-022 and in out of scope.
11. CONTRADICTION NFR-003 vs FR-008 c6–7: reword NFR-003: p95 ≤ 45 s from QUEUED to COMPLETED; hard limit 120 s after RUNNING starts (FR-008 c6); an analysis can wait in QUEUED at most about 10 minutes (FR-008 c7) before it fails.
12. CONTRADICTION FR-011 c3 vs 4.3.3 / 4.6.1 / TBL-009: the check is against the redacted text that was sent, as a substring after whitespace normalisation. Make FR-011 c3 say exactly that and align the other three places.
13. CONTRADICTION 4.3.1 `String.length()` vs TBL-005 `char_length` (code points): Java must use `text.codePointCount(0, text.length())`; say so in 4.3.1, FR-004 wording if it defines "characters", and anywhere text_length is computed.
14. GUESS FR-008 c3–4 two different failures: the stored `failureReason` is the last attempt's; `input_tokens` / `output_tokens` are summed over every attempt that returned usage (null if none did). Update FR-008, 4.6.4 and TBL-006 notes.
15. CONTRADICTION TBL-011 `db_completed_at` NOT NULL vs 4.3.6 "COMMIT, set db_completed_at": set it to `clock_timestamp()` in the INSERT of the erasure record, the last statement before COMMIT. Fix the diagram label and prose.
16. GUESS 4.3.1 extractor pool saturation: bounded pool of 4 threads per API task with a queue of 16; a request waits at most 10 s for a slot; if the queue is full or the wait expires → 503 with new code `SERVER_BUSY` (ERR-026) and `Retry-After: 5`, nothing stored. Add ERR-026 to 6.3, API-004 error table, CR-012/13 enum note if any, the frontend message mapping, and the shared-values table if 4/16/10 s appear in more than one place (put them in 4.7).

Then: add a v0.8 change-log row ("Build test 1 of the improvement round: an independent agent reading only this file with the scorer's playbook prompt logged 16 items (5 contradictions, 11 guesses); all fixed", what/why/affected IDs), bump header to v0.8, run both checkers, commit as "improvement: build test 1 fixes (16 items: status guard, last_event_at, run modes, local env, cancel retry outcomes, load test, pool size, Pro page state, notice texts, ERR-026 SERVER_BUSY)" or similar accurate message. Report commit SHA(s), new IDs, checker summary lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1 | tail -1; cd sachin && grep -n -o -E '.{0,80}(Status moves only|Every transition is one conditional|l…`
- *tool:* Bash `cd sachin && sed -n 1150,1200p spec.md | cut -c1-600; grep -n 'TBL-009\b' spec.md | head -3; awk '/^#### TBL-009/,/^#### TBL-010/' s…`
- *tool:* Bash `cd sachin && awk '/^#### 4.6.1/,/^#### 4.6.2/' spec.md | cut -c1-400; grep -n -o -E '.{0,200}(Cancel from TBL-019).{0,300}' spec.md;…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{0,10}(\| NFR-003 \|.{0,250}|PV->>DB: INSERT erasure_record.{0,80}|\| db_completed_at.{0,200}|\| last_e…`
- *tool:* Bash `cd sachin && awk '/^#### TBL-011/,/^#### TBL-013/' spec.md | cut -c1-260; sed -n 780,800p spec.md`
- *tool:* Bash `cd sachin && grep -n -o -E '(the API listens on `server.port`.{0,90}|The local column is for the API run.{0,700}|\*\*Where values co…`
- *tool:* Bash `cd sachin && grep -n -o -E '(\*\*Local stack\.\*\*.{0,900}|1\. Status moves only.{0,140}|4\. If the retry also fails, status = `FAIL…`
- *tool:* Write `h1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '(A status change from an event whose payload `created_at` is older than the row.s `last_event_at` is ski…`
- *tool:* Write `h2.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Write `h3.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{80}(mvnw spring-boot:run|503 only|HikariCP pool|fake-latency|ERR-025\b[^|]).{60}' spec.md | cut -c1-24…`
- *tool:* Write `h4.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/ && git commit -q -m "improvement: build test 1 fixes, v0.8 (16 items: status guard, last_event_at, run modes, …`
## Final report

## Result
All 16 build-test items are fixed at their source, and the spec is now v0.8 (dated 2026-11-02). One commit on `main` in `spec-homework`, staged with `git add sachin/` only, plain message, no trailers, **not pushed**:

- `f6870ed` improvement: build test 1 fixes, v0.8 (16 items: status guard, last_event_at, run modes, local env, cancel retry outcomes, load test, pool size, Pro page state, notice texts, ERR-026 SERVER_BUSY)

Earlier commits in this round are unchanged: `67af97f`, `cea4c8d`, `e843e71`, `ac901a9`, `a51ef13`, `fc086d5`, `59a4c79`.

## New IDs and sections
- **ERR-026** `SERVER_BUSY`, 503, with `Retry-After`. It is in 6.3, the API-004 error table, the new FR-003 criterion 10, the 6.1 note on 503s, the FR-003 test row, and the frontend message key `errors.SERVER_BUSY`.
- **A-21** Grievance contact "Grievance Officer, SurfingBear, grievance@surfingbear.com, replies within 30 days", marked as to be confirmed by legal.
- **Section 3.4** User-facing notice texts: the exact `messages/en.json` keys and values for the privacy notice and `pro.paymentNotice`. Numbers come from `src/lib/config/limits.ts`, which must equal the named 4.7 properties.
- **FR-030 page state (a3):** plan `PRO` with `subscription` null shows "Resume Pro until <accessUntil>" and no buttons. It sits before (b); no state was relettered.
- **New 4.7 properties:** `app.upload.extractor-threads` / `extractor-queue` (4 / 16), `app.upload.extractor-wait` / `busy-retry-after` (10s / 5s), `app.ai.fake-latency` / `AI_FAKE_LATENCY`, `app.load-test-mode` / `LOAD_TEST_MODE`, `app.db.pool-size` / `DB_POOL_SIZE` (required).
- **Terraform variable** `load_test_mode` (staging only). **Script** `scripts/run-local.sh [api|worker|migrate]`.

## Two places I did not follow the decision exactly
- **Item 15 (`db_completed_at`):** `storage_deletion.erasure_record_id` is a foreign key to the erasure record, so that record cannot be the last INSERT. The INSERT stays early and writes `clock_timestamp()`, which satisfies NOT NULL. An `UPDATE … SET db_completed_at = clock_timestamp()` is then the last statement before `COMMIT`. The diagram, TBL-011 and the change log say so.
- **Item 6 (fake-AI guard):** the spec had no fake-AI guard before. I added one: `AI_PROVIDER=fake` is refused unless `APP_ENV=local`, or `APP_ENV=staging` with `LOAD_TEST_MODE=true`. `LOAD_TEST_MODE=true` itself is refused outside staging.

## Other details I chose
- The 25 load-test accounts hold Resume Pro through Razorpay Test Mode subscriptions bought once by the test team; otherwise the free quota would stop the run. This is added to A-19.
- The local `.env` sets `DB_POOL_SIZE=10` for every mode.
- In worker mode one security chain permits every request, so the actuator answers and every other path is 404.
- The API repo's compose file no longer has the `app` profile. Only the web repo's Playwright stack runs the API in containers.

## Checker results
- ID checker: CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — only the known false positives.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.
- Stale-wording search for `String.length`, the old "extracted text (exact…", the compose `app` profile, `COMMIT, set db_completed_at` and "503 only": nothing stale is left.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
Build test 2 (fresh independent agent, whole v0.8, scorer's prompt) logged 9 items. Fix all at their source as v0.9, same rules (only sachin/, no renumbering, values once in 4.7, stale-wording search, both checkers, one local commit with a plain message and NO trailers, do NOT push). Decisions:

1. CONTRADICTION TBL-019 note and 5.4 ("deleted when Razorpay confirms" / "or when a fetch shows cancelled, completed or expired") vs the 4.3.6 outcome table (also halted, created past expire_by, not found). Fix: TBL-019 and 5.4 say "deleted on any 'done' outcome of the 4.3.6 table" and nothing else; the 4.3.6 table is the single source.
2. CONTRADICTION/GUESS Razorpay "no such subscription": Razorpay answers a fetch/cancel for an id it does not hold with HTTP 400 (`error.code` `BAD_REQUEST_ERROR`, description "The id provided does not exist"), not 404. Define once (4.4 Razorpay row, as a named outcome NOT_FOUND): HTTP 400 or 404 on a fetch or cancel of a subscription id = NOT_FOUND; 401/403 = a configuration fault, treated as "fetch failed" plus an ERROR log `event=razorpay.auth_failed` (counts in razorpay_calls_total outcome label — check its defined label values); timeout/connection/5xx = fetch failed. Replace every "HTTP 404" rule (FR-024 c3, FR-029 c8, 4.3.6 table, 4.3.8, reconciliation mapping, tests) with NOT_FOUND. FakeRazorpayClient: make its state shared between the separate api and worker JVMs by keeping it in the local Redis (hash per subscription under `ro:local:fake-razorpay:<subId>`, no TTL); unknown ids → HTTP 400 with that body, as Razorpay does. Keep its other behaviour.
3. GUESS erasure racing upload / start analysis: the upload's DB transaction (INSERT resume + resume_text) and the start-analysis transaction first take the same per-user `pg_advisory_xact_lock` key as 4.3.6/4.3.8 and re-check consent to the current notice inside the lock; if consent is gone → 403 `CONSENT_REQUIRED`, transaction rolled back, and for the upload the already-written S3 object is deleted right away (on failure a `storage_deletion` row without an erasure record — check TBL-012 allows that; if not, choose the simplest consistent option and say it). Update 4.3.1, 4.3.2, 4.3.6 (mention that it serialises with them), API-004/API-007 check notes if they list order.
4. CONTRADICTION what verification (API-018) writes for non-active statuses: make ONE mapping table "Razorpay fetched status → local effect" used by both verification (API-018) and reconciliation (FR-029 c8), with the per-path differences as columns or notes: active → ACTIVE + cycle dates + GREATEST (no payment row); authenticated → AUTHENTICATED only from CREATED; pending → PENDING; halted → HALTED + ended_at + TBL-019 HALTED row; cancelled/completed/expired → that status + ended_at; created → unchanged (reconciliation: EXPIRED if checkout_expires_at passed, INFO, not a fix); NOT_FOUND → EXPIRED + ended_at for CREATED rows (decide and state what it means for non-CREATED rows: treat as reconciliation fix to EXPIRED + ended_at + WARN, no TBL-019 row since Razorpay holds nothing); fetch failed → unchanged. Terminal rows never change; the ORPHANED backstop (FR-029 c9) and the last_event_at rule apply on both paths. API-018 response: 200 if plan is PRO after the update, else 202. Point FR-024 c8, FR-029 c5/c8/c9, 4.3.8, 4.3.9 and the state note at the table instead of restating.
5. GUESS erasure_record.completed_at: define it as "the time the last storage_deletion row and the last subscription_cancellation row of this erasure are gone" (check NFR-012's wording and keep them consistent). Set by whichever step removes the last one: the erasure transaction itself at commit when there were no keys and no open subscriptions; the after-commit listener when its immediate S3 deletes and Razorpay cancels all succeed; otherwise the retry job (CMP-015) or the cancellation job (CMP-021) when it removes the last row. Each checks "no rows left for this erasure_record_id" in the same transaction as its delete. Make sure subscription_cancellation rows can be linked to the erasure record (check TBL-019; if they have no erasure_record_id, add a nullable FK column `erasure_record_id` in V3 since V3 is unreleased, ON DELETE SET NULL, and say why).
6. GUESS ShedLock durations: add to 4.7 one row per lock name with lockAtMostFor / lockAtLeastFor: analysis sweeper (every minute) 50s / 10s; storage deletion retry (10 min) 9m / 30s; Razorpay cancellation retry (10 min) 9m / 30s; retention (daily 02:00) 2h / 5m; retention audit (daily 06:00) 30m / 5m; reconciliation (daily 03:00) 2h / 5m. Use the lock names the spec already gives (check 4.3.x and 5.2 shedlock). State that a run longer than lockAtMostFor must stop between batches when its remaining lock time is under one batch (retention batches, reconciliation rows) and continue next run, so two tasks never run it together.
7. CONTRADICTION PagerDuty integration (4.8.5 step 5 "Amazon SNS integration" vs 4.8.7 "Amazon CloudWatch integration URL"): replace both with one design: a small Lambda `ro-prod-pagerduty-forwarder` (Terraform observability module, Node 22, 128 MB) subscribed to SNS topic `ro-prod-alerts-page`; it turns each message (CloudWatch alarm JSON, or Alertmanager's text message) into a PagerDuty Events API v2 `trigger` or `resolve` with `dedup_key` = alert name + env; routing key from Secrets Manager `ro/prod/pagerduty`; failures retried by Lambda's async retry (2) then to a DLQ SQS `ro-prod-pagerduty-dlq` with a CloudWatch alarm on DLQ depth > 0 to the Slack warning topic. Update the manual-steps list (ops creates the PagerDuty service and an Events API v2 integration, routing key into the secret), the secrets table in 4.8.2, A-20 if needed, and remove `pagerduty_endpoint`.
8. GUESS "Continue payment" key: state (d)'s button calls API-017 (which returns 200 with the same CREATED row and its `razorpayKeyId`, FR-024 c3) and opens Checkout with that response. No change to API-019.
9. CONTRADICTION upload check order: magic-byte detection (PDF prefix; DOCX = ZIP magic + `word/document.xml` entry) runs on the request thread before a pool slot is taken, so a PNG on a busy server gets 415; the extractor slot (6a) is taken after it, then protection, pages, text. Make 4.3.1, API-004's check list (add the slot step, section 6 wins) and CMP-006's description agree.

Then add a v0.9 change-log row ("Build test 2 of the improvement round: an independent agent logged 9 items (5 contradictions, 4 guesses); all fixed"), bump the header to v0.9, run both checkers, commit as "improvement: build test 2 fixes, v0.9 (9 items: cancel-row done rule, Razorpay not-found, erasure lock on upload and start, one fetched-status table, erasure completed_at, ShedLock durations, PagerDuty forwarder, continue payment, upload check order)". Report SHA and checker lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1|tail -1; cd sachin && grep -n -o -E '.{0,120}(HTTP 404|404\b[^0-9`]{0,3}→|deleted when Razorpay|when a…`
- *tool:* Bash `cd sachin && awk '/^#### TBL-015/,/^#### TBL-016/' spec.md | cut -c1-250; awk '/^#### TBL-019/,/^### 5.3/' spec.md | cut -c1-300; se…`
- *tool:* Bash `cd sachin && grep -n -o -E '(8\. With a valid signature.{0,700}|8\. Reconciliation, daily at 03:00 IST.{0,1300}|9\. \*\(v0.6\)\* Bac…`
- *tool:* Bash `cd sachin && grep -n -o -E '9\. \*\(v0.6\)\* Backstop.{0,900}' spec.md | cut -c1-1000; sed -n 868,884p spec.md; grep -n -o -E '(When…`
- *tool:* Bash `cd sachin && sed -n 905,922p spec.md; grep -n 'API->>DB: INSERT resume + resume_text (one transaction)' spec.md; grep -n -o -E '.{0,…`
## Agent

Writing the v0.9 edit script.

- *tool:* Write `i1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{0,100}(HTTP 404|or 404\b|expired or 404|in memory|in-memory|pagerduty_endpoint|SNS integration|CloudWa…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
## Agent

Now the change-log row, checkers, and commit.

- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{20}[0-9]{3} `BAD_REQUEST_ERROR`.{10}' spec.md; sed -i 's/HTTP 400 `BAD_REQUEST_ERROR` or 404/HTTP 400 …`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/ && git commit -q -m "improvement: build test 2 fixes, v0.9 (9 items: cancel-row done rule, Razorpay not-found,…`
## Final report

## Result
All 9 build-test-2 items are fixed at their source, and the spec header now reads v0.9. One local commit on `main` in `spec-homework`, staged with `git add sachin/` only, plain message, no trailers, **not pushed**:

- `9206e89` improvement: build test 2 fixes, v0.9 (9 items: cancel-row done rule, Razorpay not-found, erasure lock on upload and start, one fetched-status table, erasure completed_at, ShedLock durations, PagerDuty forwarder, continue payment, upload check order)

## What changed, by item
1. **Cancellation row "done" rule:** TBL-019 and 5.4 now say only "deleted on any 'done' outcome of the 4.3.6 table". FR-019 criterion 7's "until Razorpay confirms" was updated to match.
2. **Razorpay not-found:**
   - 4.4 now defines three named outcomes once: NOT_FOUND (HTTP 400 with Razorpay error code `BAD_REQUEST_ERROR`, or 404), auth failure (401/403, handled as fetch failed plus ERROR `event=razorpay.auth_failed`), and fetch failed.
   - Every old "HTTP 404" rule now says NOT_FOUND: FR-024 criterion 3, FR-029 criterion 8, the 4.3.6 table and the 4.3.8 diagram.
   - `razorpay_calls_total{outcome}` gains `not_found` and `auth_failed`. The Razorpay failure-rate alert now counts only `timeout|error|auth_failed`.
   - `FakeRazorpayClient` keeps its state in local Redis under `ro:local:fake-razorpay:<subId>` (added to the Redis keys table). Unknown ids get HTTP 400 with Razorpay's error body.
3. **Erasure racing upload / start:**
   - The upload transaction and the start-analysis transaction now take the per-user advisory lock (`ro-sub:` key) and re-check consent inside it; if consent is gone, the answer is 403 `CONSENT_REQUIRED`.
   - The upload then deletes its S3 object through an `UPLOAD_ROLLBACK` `storage_deletion` row. TBL-012 already allows a null erasure record, so no schema change was needed.
   - 4.3.1 diagram, the 4.3.2 SQL and 4.3.6 prose are updated.
4. **One fetched-status table:** a new "Razorpay fetched status → local effect" table in 4.3.9 covers verification, reconciliation and the expiry fetch of FR-024 criterion 3.
   - A NOT_FOUND on a CREATED row is an abandoned checkout (INFO, not a fix). On any other non-terminal row it becomes EXPIRED, logs WARN and counts as a fix, with no TBL-019 row.
   - The ORPHANED backstop and the `last_event_at` rule apply on every path.
   - FR-024 criterion 8, FR-029 criteria 8–9, the 4.3.8 diagram and prose, and API-018's check list and response now point to the table.
5. **`completed_at`:** it now means the time both the S3 deletes and the Razorpay cancels of that erasure are done. It is set by whichever step removes the last row.
   - **New column `subscription_cancellation.erasure_record_id`:** nullable foreign key to the erasure record, ON DELETE SET NULL. It goes into V3, which is unreleased.
   - **New index `ix_subscription_cancellation_erasure_record_id`:** added to 5.3 and to V3's step 5.
6. **ShedLock:** a 4.7 table gives `lockAtMostFor` / `lockAtLeastFor` for the six existing lock names in TBL-015, with your durations.
   - It also states the rule that a job stops between batches when its remaining lock time is short.
   - I picked "less than one batch" as one batch's statement timeout, and 15 s per reconciliation row (the Razorpay timeouts plus margin).
7. **PagerDuty:** prod critical alerts now go through a Lambda `ro-prod-pagerduty-forwarder` (Events API v2, `dedup_key` = alert name + env).
   - It reads the secret `ro/prod/pagerduty` (`routing_key`). Failures land in the queue `ro-prod-pagerduty-dlq`, which has a new warning alert `PagerDutyForwarderFailed`.
   - Updated: 4.8.2 secrets table, 4.8.5 manual step 5 and secret counts, the observability module comment, and the routing table. `pagerduty_endpoint` is removed everywhere.
8. **Continue payment:** FR-030 state (d)'s button calls API-017 and opens Checkout with its answer.
9. **Upload check order:** magic-byte detection runs on the request thread before the extractor slot. 4.3.1, API-004's check list (new step 5a) and CMP-006 now agree.

Test rows in 7.1 for FR-019, FR-024 and FR-029 are extended to cover these changes. A v0.9 change-log row lists the what, the why (9 items: 5 contradictions, 4 guesses) and the affected IDs.

## New IDs
None. No ID was renumbered or reused.

## Checker results
- ID checker:
  - CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1. Razorpay's own `BAD_REQUEST_ERROR` was flagged at first; I reworded those two mentions.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.
- Stale-wording search: "HTTP 404", "expired or 404", the in-memory fake, `pagerduty_endpoint`, "SNS integration", "CloudWatch integration" and "Razorpay confirms" no longer appear as rules.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
Build test 3 (fresh independent agent, whole v0.9, scorer's prompt) logged 10 items. Fix all at their source as v1.0, same rules as before (only sachin/spec.md — note sachin/process.md and sachin/transcripts/ were updated and committed by me, do not touch them; `git add sachin/spec.md` only this time; no renumbering; values once in 4.7; stale-wording search; both checkers; one local commit, plain message, NO trailers, do NOT push). Decisions:

1. CONTRADICTION FR-016 c3 "no contact details and no name" vs c2 (visible suggestions quote `original_line`) / TBL-009 / 4.6.1 (only email, phone, URL redacted; module never knows the name). Fix FR-016 c3: the PDF has no field of its own for a name or contact details; quoted resume lines (`original_line`) appear exactly as sent to the AI, i.e. with `[EMAIL]`, `[PHONE]`, `[URL]` already in place, so a line that contained a name still shows it; the report is the user's own data and is only downloaded by them. Update `ReportRendererTest` in 7.1 (no email/phone/URL pattern anywhere in the extracted PDF text; no name field) and any other place repeating "no name".
2. GUESS FR-018/API-006 vs API-007 race: API-006 (delete a resume) takes the per-user advisory lock (same key) and `SELECT … FROM resume WHERE id = ? AND user_id = ? FOR UPDATE` before its QUEUED/RUNNING check and delete; API-007's locked transaction re-reads the resume (`FOR SHARE`) inside the lock and returns 404 `RESUME_NOT_FOUND` / 410 `RESUME_EXPIRED` from that read, before reserving quota. Update 4.3.2, 4.3.6 (resume delete), API-006/API-007 notes, FR-018 tests.
3. CONTRADICTION run-mode ports: locally api and worker both run on the host. `scripts/run-local.sh worker` sets `SERVER_PORT=8082` and `MANAGEMENT_SERVER_PORT=8082`; api keeps 8080/8081. Add the rows (local column) to 4.8.2 and mention in the run-mode table; deployed tasks are unchanged (one container per task).
4. GUESS erasure_record.completed_at race between CMP-015 and CMP-021: every path that may set `completed_at` first runs `SELECT … FROM erasure_record WHERE id = ? FOR UPDATE` in the same transaction as its delete, then checks for remaining rows. Say it once in TBL-011 and point to it from 4.3.6 and the job descriptions.
5. GUESS frontend text per error code: in 3.4 add a table: every ERR code (ERR-001…ERR-026) → `errors.<CODE>` key, English text, UI action (e.g. CONSENT_REQUIRED → show the consent panel inline; RESUME_EXPIRED → "Upload it again" button to the upload page; QUOTA_EXCEEDED → message with reset date + Resume Pro button; RATE_LIMITED → message with Retry-After seconds, button disabled until then; UNAUTHENTICATED → handled by CR-035 redirect, no text; INTERNAL_ERROR → generic text with the requestId; etc.). Also `analysis.failure.<REASON>` for the three failure reasons; `pro.benefits.*` (the four Pro benefits: unlimited analyses, all suggestions, history filters and scores, compare two analyses); the FREE PDF footer line text (FR-016 c4) with its count placeholder; category display names for 5.5's categories (use the category codes the seed defines). Keep numbers referenced to 4.7 names as 3.4 already does.
6. GUESS screen composition: add to 2.2 (or CMP-001) a per-page table: page path | what it shows | API calls | user actions. Decisions: the upload page `/resume-optimizer` shows usage (API-012), the consent panel inline when API-016 says not current (Accept → API-002), the "Your resumes" list (API-005) with "Analyse" (opens role/level choice, API-007) and "Delete" (API-006, confirm dialog), and the upload control (API-004); the result page `/analyses/[id]` has "Download PDF" (API-011, only when COMPLETED) and "Delete analysis" (API-010, confirm dialog); history `/analyses` lists (API-008) with a per-row delete; privacy page `/privacy` shows the notice, "Withdraw consent" (API-003, confirm dialog) and "Delete all my data" where the user types DELETE into a text field that enables the button, and the UI sends `"confirm": "DELETE"` (API-013) — plus the FR-019 c8 PRO line; Pro page and compare page as already specified.
7. CONTRADICTION notice versioning: key the notice texts by version: `privacy.notice.<version>.*` (e.g. `privacy.notice.2026-10-v1.purpose`). The UI shows the texts for the version API-016 returns; if it has no texts for that version it shows "The privacy notice is being updated. Please try again shortly." and no Accept button (so upload stays blocked by CONSENT_REQUIRED). Release rule: the web with the new texts is deployed before the API's `PRIVACY_NOTICE_VERSION` changes (add to the 4.8.4 pipeline/release notes and CR-038 if it lists ordering).
8. GUESS job timeouts: add to 4.7 `app.jobs.statement-timeout` = 60s (applied with `SET LOCAL statement_timeout` in each job transaction; this is "the time one batch may take" in the ShedLock stop rule) and the S3 client timeouts `app.storage.api-call-timeout` = 10s / `api-call-attempt-timeout` = 5s (used by CMP-015 for the per-row budget and by uploads). Point the ShedLock rule at them.
9. GUESS requestId on re-queue: no schema change; the sweeper's re-queue XADD carries a new UUID as `requestId` and logs `event=analysis.requeued` with the analysisId and both the new requestId; say so in CMP-009/4.3.4 and CR-010's background-job note.
10. CONTRADICTION V1 contents: state precisely in 5 intro / 5.6 that V1 (`V1__init.sql`) creates TBL-001–TBL-015 exactly as 5.2 shows them except (a) the two plan CHECKs on TBL-006 and TBL-014 are `('FREE','PREMIUM')` and (b) whatever else V3 adds or changes to those tables (check 5.6 and list each item), so a builder can write V1 and V3 from 5.2 + 5.6 alone. Do not fold V3 into V1.

Also: the FakeRazorpayClient's unknown-id body and the 4.4 NOT_FOUND definition must use Razorpay's exact description text "The id provided does not exist or access is unauthorised" (checked today against razorpay.com/docs/errors/common: code `BAD_REQUEST_ERROR`). Match on the code, not the text.

Then a v1.0 change-log row ("Build test 3 of the improvement round: an independent agent logged 10 items (4 contradictions, 6 guesses); all fixed" + what/why/affected IDs), header v1.0, checkers, commit "improvement: build test 3 fixes, v1.0 (10 items: PDF name rule, resume delete vs start lock, local worker port, completed_at lock, UI texts per error code, page composition, versioned notice texts, job timeouts, re-queue requestId, V1 contents)". Report SHA and checker lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1|tail -1; git log --oneline -3; cd sachin && grep -n -o -E '.{0,60}(no contact details and no name|no n…`
- *tool:* Bash `cd sachin && awk '/^#### TBL-001/,/^#### TBL-016/' spec.md | grep -n -E 'v0\.[4-9]' | cut -c1-260; sed -n 1753p spec.md; awk '/^### …`
- *tool:* Bash `cd sachin && awk '/^### 5.3/,/^### 5.4/' spec.md | grep -E 'v0\.[4-9]|^\| Table|^\| Index' | cut -c1-200; awk '/^#### API-006/,/^###…`
- *tool:* Bash `cd sachin && awk '/^### 5.3/,/^### 5.4/' spec.md | cut -c1-140 | head -30`
- *tool:* Bash `cd sachin && grep -n -o -E '(\| Lost message \|.{0,200}|`ReportRendererTest` \(text extracted.{0,80}|`ResumeDeleteIT` \(.{0,120}|\| …`
- *tool:* Bash `cd sachin && grep -n -o -E 'scripts/run-local.sh \[api\|worker\|migrate\]` \(default `api`\): it runs.{0,120}' spec.md; grep -n -o '…`
- *tool:* Write `j1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Write `j2.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{0,80}(no name|no contact|The id provided does not exist"|`privacy\.notice\.(title|purpose)`).{0,60}' s…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/spec.md && git commit -q -m "improvement: build test 3 fixes, v1.0 (10 items: PDF name rule, resume delete vs s…`
## Final report

## Result
All 10 build-test-3 items are fixed at their source, plus the exact Razorpay not-found text. The spec header now reads v1.0. One local commit on `main` in `spec-homework`, staged with `git add sachin/spec.md` only. I did not touch `process.md` or `transcripts/`. The message is plain with no trailers, and it is **not pushed**:

- `78d2cf5` improvement: build test 3 fixes, v1.0 (10 items: PDF name rule, resume delete vs start lock, local worker port, completed_at lock, UI texts per error code, page composition, versioned notice texts, job timeouts, re-queue requestId, V1 contents)

## What changed, by item
1. **PDF name rule (FR-016 criterion 3):** the PDF has no name or contact field of its own. Quoted lines appear as they were sent to the AI, with the placeholders in place, so a name inside a line still shows. `ReportRendererTest` in 7.1 now checks for no email, phone or URL pattern and no name field.
2. **Resume delete vs start:**
   - API-006 (delete a resume) now takes the per-user advisory lock and locks the resume row `FOR UPDATE` before its in-progress check.
   - API-007 (start an analysis) re-reads the resume `FOR SHARE` inside its locked transaction. It answers 404 or 410 from that read, before reserving quota.
   - Updated: 4.3.2 (prose and SQL), 4.3.6, the API-006 and API-007 notes, and the delete-resume test row.
3. **Local worker port:** `scripts/run-local.sh worker` sets `SERVER_PORT=8082` and `MANAGEMENT_SERVER_PORT=8082`. A new 4.8.2 row covers this, and it is noted in the run-mode table and in 7.3.
4. **`completed_at` race:** every path that may set it first locks the erasure record (`FOR UPDATE`), then deletes its row, then checks what is left. This is written once in TBL-011, and the 4.3.6 diagram and the two job components point to it.
5. **UI texts in 3.4:**
   - Text and UI action for every error code ERR-001 – ERR-026, with numbers referenced to 4.7 through `limits.ts`.
   - The three failure-reason texts, the four Pro benefits, and display names for the five catalogue categories.
   - A small table of the PDF report's own texts. They live in the backend template, not in `en.json`.
6. **Page table in 2.2:** for each page, what it shows, which APIs it calls and what the user can do, with your decisions (inline consent panel, resume list with Analyse and Delete, typed `DELETE` to erase).
7. **Versioned notice texts:** keys are now `privacy.notice.<version>.*` (the existing rows are under `2026-10-v1`).
   - If no texts exist for the version API-016 returns, the page shows the new `privacy.noticeUnavailable` text with no Accept button.
   - A new notice version is released web first. This exception is now in CR-038 and 4.8.4.
8. **Job timeouts (4.7):** `app.jobs.statement-timeout` is 60s. The S3 client timeouts are `app.storage.api-call-timeout` / `api-call-attempt-timeout` (10s / 5s). The ShedLock stop rule and the deletion job now refer to them.
9. **Re-queue request id:** the sweeper's re-queue sends a new UUID as `requestId` and logs INFO `event=analysis.requeued`. CR-010, the queue component (CMP-009) and 4.3.4 say so.
10. **V1 contents:** the 5 intro and 5.6 now state what V1 has instead. The two plan CHECKs are `('FREE','PREMIUM')`, and the Pro-history index on `analysis` is missing; no other part of TBL-001 – TBL-015 changed after v0.3. Tables TBL-016 – TBL-019 come from V3 only. V3 is not folded into V1.

**Razorpay text:** 4.4 and the fake client's unknown-id body now use "The id provided does not exist or access is unauthorised". The spec says the module matches on HTTP status and code, never on the text.

**One extra fix:** the v0.9 index row `ix_subscription_cancellation_erasure_record_id` had only three cells in the four-column 5.3 table. It is now a proper row in its place.

A v1.0 change-log row lists what changed, why (10 items: 4 contradictions, 6 guesses) and the affected IDs. No ID was renumbered or reused, and there are no new IDs.

## Checker results
- ID checker:
  - CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.
- Stale-wording search: no unversioned `privacy.notice.*` keys and no old Razorpay text are left. The only "no name" matches are the new FR-016 wording, the test row and the change log, plus FR-024's unrelated "notes" field.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
Build test 4 (fresh independent agent, whole v1.0, scorer's prompt) logged 7 items, none high. Fix all as v1.1, same rules (`git add sachin/spec.md` only; no renumbering; values once in 4.7; stale-wording search; both checkers; one local commit, plain message, NO trailers, do NOT push). Decisions:

1. GUESS pages need the reader's plan: add API-012 (returns `plan`) to the calls of `/analyses`, `/analyses/[analysisId]` and `/privacy` in the 2.2 page table, and say what each uses it for (Pro filters/compare boxes; upgrade banner; FR-019 c8 line in the delete dialog). No API change.
2. GUESS grants for V1 tables: state that V1 ends with `GRANT SELECT, INSERT, UPDATE, DELETE ON` every TBL-001–TBL-015 table (including `shedlock`) `TO ro_app` (the role exists because the migrate step creates it before Flyway); make 5 intro, 5.6 and the V1 description agree.
3. GUESS log redaction filter (CR-009, 2.2 `common/logging`): define it once: a Logback masking step in the JSON encoder (logstash-logback-encoder `MaskingJsonGeneratorDecorator` or equivalent) applied to the message, every MDC/structured field value and exception messages; it REPLACES matches (does not drop the line): the 4.6.1 email/phone/URL regexes → `[EMAIL]` / `[PHONE]` / `[URL]`; JWT-shaped strings `eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+` → `[TOKEN]`; `rzp_(live|test)_[A-Za-z0-9]+` → `[RAZORPAY_KEY]`. Request logging never logs headers or bodies, so cookie/Authorization values never reach it. The web server's pino logger uses `redact` for the same field names. Point CR-009's check at the named unit test with these samples.
4. GUESS `analysis_duration_seconds`: define it as `finished_at − created_at` (QUEUED to COMPLETED, matching NFR-003), observed by CMP-010 when an analysis reaches COMPLETED only; FAILED ones are counted by `ai_failures_total`. State that TBL-014 `duration_ms` / API-015 measure processing time (`finished_at − started_at`) on purpose, and keep both definitions in one place each.
5. GUESS `scripts/send-test-webhook.sh <event>` bodies: the script reuses API-022's `subscription.charged` example with `event` replaced, `payload.subscription.entity.status` set per event (authenticated→`authenticated`, activated→`active`, pending→`pending`, halted→`halted`, cancelled→`cancelled`, completed→`completed`, charged→`active`), and `payload.payment` removed for every event except `subscription.charged`; `created_at` = now. Write it in 4.4 next to the script.
6. CONTRADICTION FR-016 c4 vs 3.4: add "when `hiddenSuggestionCount` > 0" to FR-016 c4.
7. CONTRADICTION NFR-009 / 7.2 "50 samples" vs 4.6.1's 14: change NFR-009 and its 7.2 row to "the sample set in 4.6.1 (14 values: 9 must be replaced, 5 must not)" — check the exact counts in 4.6.1 and use them.

Then v1.1 change-log row ("Build test 4 of the improvement round: an independent agent logged 7 items (5 guesses, 2 contradictions, none high); all fixed"), header v1.1, checkers, commit "improvement: build test 4 fixes, v1.1 (7 items: plan on three pages, V1 grants, log redaction filter, analysis duration metric, test webhook bodies, PDF hidden-count line, redactor sample count)". Report SHA and checker lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1|tail -1; cd sachin && grep -n -o -E '.{0,80}(50 sample|log redaction|redaction filter|analysis_duratio…`
- *tool:* Bash `cd sachin && awk '/^#### TBL-006/,/^#### TBL-007/' spec.md | grep -E '_at \|' | cut -c1-160; grep -n -o -E 'posts the 4.3.9 example …`
- *tool:* Write `k1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
## Final report

## Result
All 7 build-test-4 items are fixed, and the spec header now reads v1.1. One local commit on `main` in `spec-homework`, staged with `git add sachin/spec.md` only. The message is plain with no trailers, and it is **not pushed**:

- `d6ee2f5` improvement: build test 4 fixes, v1.1 (7 items: plan on three pages, V1 grants, log redaction filter, analysis duration metric, test webhook bodies, PDF hidden-count line, redactor sample count)

## What changed, by item
1. **Plan on three pages:** the 2.2 page table now has `/analyses`, `/analyses/[analysisId]` and `/privacy` call API-012 for the plan. Each row says what it is used for: the Pro filters and compare boxes, the upgrade banner, and the FR-019 criterion 8 line in the delete dialog. No API change.
2. **V1 grants:** the section 5 intro, the "What V1 has instead" paragraph and 5.6 step 6 all say V1 ends with the `SELECT, INSERT, UPDATE, DELETE` grant to `ro_app` on TBL-001 – TBL-015, `shedlock` included. They note that `ro_app` exists because the migration task creates it before Flyway runs.
3. **Log redaction filter:** now defined once, in a new paragraph under the 1.3 table.
   - It masks in the JSON encoder and covers the message, MDC and structured fields, and exception text. It replaces matches and never drops the line.
   - What it replaces: the four 4.6.1 regexes, JWT-shaped strings → `[TOKEN]`, and `rzp_(live|test)_…` → `[RAZORPAY_KEY]`.
   - It is described as a safety net: headers and bodies are never logged in the first place.
   - The web server's pino logger uses `redact` on the same field names and the same regexes.
   - CR-009's check now names `LogRedactionTest`, and the 2.2 tree comment points to 1.3.
   - The task said "the 4.6.1 email/phone/URL regexes". 4.6.1 actually has four regexes (two of them for URLs), so the spec says four.
4. **`analysis_duration_seconds`:** defined once in 4.8.7 as `finished_at − created_at`, observed only when an analysis reaches COMPLETED. FAILED analyses are counted by `ai_failures_total`. The TBL-014 `duration_ms` note says it is processing time (`finished_at − started_at`) on purpose, for API-015, and NFR-003 points to the 4.8.7 definition.
5. **Test webhook bodies:** 4.4 now describes the body `scripts/send-test-webhook.sh` builds from API-022's `subscription.charged` example. It sets the event, id and per-event status, removes `payload.payment` except for `charged`, and uses the current time for `created_at`. The old reference to a "4.3.9 example body" was wrong (no per-event example exists there) and is replaced.
6. **PDF hidden-count line:** FR-016 criterion 4 now adds the line only when `hiddenSuggestionCount` > 0, matching 3.4.
7. **Redactor sample count:** I counted the 4.6.1 set as 14 values: 9 must be replaced, 5 must not. NFR-009 and its 7.2 row now say so instead of "50".

A v1.1 change-log row lists what changed, why (7 items: 5 guesses, 2 contradictions, none high) and the affected IDs. No ID was added, renumbered or reused.

## Checker results
- ID checker:
  - CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.
- Stale-wording search: no "50 sample" and no "4.3.9 example body" left.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
Build test 5 (fresh independent agent, whole v1.1, scorer's prompt) logged 6 items, none high. Fix all as v1.2, same rules (`git add sachin/spec.md` only; no renumbering; values once in 4.7; stale-wording search; both checkers; one local commit, plain message, NO trailers, do NOT push). Decisions:

1. GUESS who owns the "more than half of the suggestions dropped" retry (4.3.3 / 4.2 / 4.6.1 / 4.6.4): CMP-011 (AI client) runs EVERY response check — schema (4.6.3), the 4.6.4 checks including length limits, and the FR-011 suggestion checks (substring of the redacted text it built, placeholder in rewrite) and the more-than-half rule — and decides the single retry itself, so one analysis makes at most 2 provider calls (2 × 45 s + 2 s < the 120 s sweeper limit). It returns to CMP-010 the validated result with the surviving suggestions and, for ranking, each suggestion's position (character offset of `original_line` in the redacted text); CMP-010 only ranks (FR-011 c4), computes the overall score and saves. Update 4.3.3 diagram/prose, CMP-010/CMP-011 rows, 4.6.4, FR-011 c3–c4 wording if they name the component, and tests (`SuggestionCheckTest` belongs to `ai/`).
2. GUESS connection-level failures to Anthropic: connection refused, DNS failure, TLS error, connection reset → treated like 5xx: retried once after 2 s, then `AI_PROVIDER_ERROR`. Write it in FR-008 c3–c4 and 4.4's Anthropic row (one place for the classification, the other points to it).
3. CONTRADICTION cancellation retry vs `RazorpayCancellationOverdue`: for a fetched `created` whose `expire_by` has not passed, set the TBL-019 row's `next_attempt_at` = `expire_by` + 1 minute and exclude rows whose `next_attempt_at` is in the future and whose last fetched status was `created` from the overdue metric — simplest consistent form: the overdue metric counts rows older than 24 h whose `next_attempt_at` ≤ now. Check TBL-019 has `next_attempt_at` (if it has another retry column, use it; if none, add `next_attempt_at timestamptz NOT NULL DEFAULT now()` in V3, unreleased) and make the job select only due rows. Update the 4.3.6 table, TBL-019, the metric definition in 4.8.7 and the alert row.
4. GUESS unlisted Razorpay statuses: add a row to the 4.3.9 fetched-status table: "`paused` or any status not listed: row unchanged, ERROR log `event=billing.unknown_status` with ids, not counted as a fix; verification answers 202". The same applies to webhook event types already handled as IGNORED (no change there).
5. GUESS FakeRazorpayClient after cancel: the fake stores status and a `cancel_at_cycle_end` flag. `fetch` returns the stored state: `created` until the first verification fetch (API-018) for that id, then `active` with `current_start`/`current_end` as already specified; after `cancel` with `cancel_at_cycle_end: 0` → `cancelled`; after `cancel` with `cancel_at_cycle_end: 1` → still `active` (with Razorpay's `has_scheduled_changes`/end-at-cycle-end flag if the spec models one; otherwise just keep `active`) until `current_end`, then `cancelled`. So local reconciliation leaves an unpaid checkout `created` and does not end a cycle-end cancel early. Replace the old "fetch returns active" sentence.
6. CONTRADICTION 4.8.2 local cells for `APP_MODE` and `JAVA_TOOL_OPTIONS` mention the API repo's compose: change both to "in the web repo's `e2e/docker-compose.yml` services (4.8.4)" for the containerised case and the host values for `scripts/run-local.sh`; search for any other cell or sentence implying the API repo's compose runs the app.

Then v1.2 change-log row ("Build test 5 of the improvement round: an independent agent logged 6 items (4 guesses, 2 contradictions, none high); all fixed"), header v1.2, checkers, commit "improvement: build test 5 fixes, v1.2 (6 items: AI checks and retry in CMP-011, connection failures, cancel-retry schedule vs overdue alert, unknown Razorpay status, fake client after cancel, local compose cells)". Report SHA and checker lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1|tail -1; cd sachin && grep -n -o -E '.{0,10}(\| CMP-010 \|.{0,400}|\| CMP-011 \|.{0,400}|3\. The AI ca…`
- *tool:* Bash `cd sachin && sed -n '/^#### 4.3.3 Process an analysis/,/^#### 4.3.4/p' spec.md | cut -c1-1500; grep -n -o -E '.{0,30}(StorageDeletio…`
- *tool:* Write `l1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && grep -n -o -E '.{0,60}(in those compose services|in compose `|validates and saves|skill filter, suggestion checks|compo…`
- *tool:* Bash `cd /c/Users/user/spec-homework && git add sachin/spec.md && git commit -q -m "improvement: build test 5 fixes, v1.2 (6 items: AI checks and retry in CMP-011, co…`
## Final report

## Result
All 6 build-test-5 items are fixed, and the spec header now reads v1.2. One local commit on `main` in `spec-homework`, staged with `git add sachin/spec.md` only. The message is plain with no trailers, and it is **not pushed**:

- `13d3050` improvement: build test 5 fixes, v1.2 (6 items: AI checks and retry in CMP-011, connection failures, cancel-retry schedule vs overdue alert, unknown Razorpay status, fake client after cancel, local compose cells)

## What changed, by item
1. **Checks and retry in CMP-011:**
   - The AI client now runs every check on a response: the 4.6.3 schema, the 4.6.4 checks, the missing-skill filter, the suggestion checks, and the more-than-half rule. It also decides the single retry, so an analysis makes at most 2 provider calls (2 × 45 s + 2 s = 92 s, under the 120 s sweeper limit).
   - It returns the surviving suggestions with their character offset in the redacted text. CMP-010 now only computes the score and band, ranks by severity then offset, and saves.
   - Updated: the 4.3.3 diagram and prose, the CMP-010 and CMP-011 rows, FR-011 criterion 3 (names CMP-011), and the 7.1 tests. `SuggestionCheckTest` now sits in `ai/`, and a new `SuggestionRankingTest` sits in `analysis.worker`.
   - I also moved the missing-skill filter into CMP-011, because CMP-010 was to "only rank, score and save".
2. **Connection failures to Anthropic:** the classification lives once, in 4.4's Anthropic row:
   - timeout → `AI_TIMEOUT`;
   - 429, 5xx, 529, connection refused, DNS failure, TLS error, connection reset → retried like a 5xx, then `AI_PROVIDER_ERROR`;
   - any other 4xx → `AI_PROVIDER_ERROR` at once;
   - an invalid response → `AI_INVALID_RESPONSE`.
   FR-008 criterion 3 points to 4.4.
3. **Cancel retry vs the overdue alert:** TBL-019 already had `next_attempt_at`, so no schema change.
   - A fetched `created` whose `expire_by` has not passed now sets `next_attempt_at` = `expire_by` + 1 minute.
   - The job takes only due rows (`next_attempt_at <= now()`), and "retried at the next run" means now + 10 minutes.
   - `CancellationOverdue` counts only due rows older than 24 h.
   - Updated: the 4.3.6 table and its note, the TBL-019 column note, the 4.3.4 metric definition, and the alert row.
4. **Unknown Razorpay status:** a new row in the 4.3.9 fetched-status table covers `paused` or any status not listed. The row is unchanged, an ERROR `event=billing.unknown_status` is logged, it is not counted as a fix, and verification answers 202. Webhook types not in the table stay `IGNORED`.
5. **Fake client after cancel:** `FakeRazorpayClient` now stores a status and a `cancel_at_cycle_end` flag per subscription.
   - `fetch` returns `created` until the first verification fetch, then `active` with the existing cycle dates.
   - A cancel with `cancel_at_cycle_end: 0` gives `cancelled` at once. With `1` it stays `active` until `current_end`, then becomes `cancelled`.
   - The spec models no separate scheduled-changes flag, so it simply stays `active`. The old "fetch returns active" sentence is replaced.
6. **Local compose cells:** the 4.8.2 local cells for `APP_MODE` and `JAVA_TOOL_OPTIONS` now name `scripts/run-local.sh` for the host run and the web repository's `e2e/docker-compose.yml` for containers. A stale-wording search found no other place implying the API repository's compose runs the app.

A v1.2 change-log row lists what changed, why (6 items: 4 guesses, 2 contradictions, none high) and the affected IDs. No ID was added, renumbered or reused.

## One detail I chose
The expiry fetch of FR-024 criterion 3 (the API-017 path) on an unknown status keeps the row and answers 409 `SUBSCRIPTION_EXISTS`, because that subscription may still be chargeable.

## Checker results
- ID checker:
  - CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.

## Follow-up instruction

```text
The coordinator sent a message while you were working:
Build test 6 (fresh independent agent, whole v1.2, scorer's prompt) logged 12 items, none high. Fix all as v1.3 — this is the last round. Same rules (`git add sachin/spec.md` only; no renumbering; values once in 4.7; stale-wording search; both checkers; one local commit, plain message, NO trailers, do NOT push). Decisions:

1. CONTRADICTION FR-030 c3 (b) vs (c): (b) = plan `PRO`, `subscription.status` = `ACTIVE` and `cancelAtPeriodEnd` false; (c) = plan `PRO` and (`cancelAtPeriodEnd` true or status `CANCELLED`). Also fix the "six states" count to the real number of states.
2. GUESS `Api5xxRateHigh` pages on deliberate 503s: the critical alert counts only unexpected errors: rate of `api_errors_total{code="INTERNAL_ERROR"}` over all requests (plus the ALB's own `HTTPCode_ELB_5XX_Count` if the alert table already uses it — keep whatever second source exists consistent). `PAYMENT_PROVIDER_UNAVAILABLE` stays covered by the Razorpay failure-rate alert; add a warning `ServerBusy`: `increase(api_errors_total{code="SERVER_BUSY"}[5m]) > 20`. Update NFR-013's pointer if it names the alert.
3. GUESS cancel dialog when `accessUntil` is null (states (a) and (a2)): FR-028 c9 gets a second text: "Cancel this subscription? You will not be charged." shown when `accessUntil` is null; add both texts as keys in 3.4.
4. GUESS after 150 s of polling (FR-013 c4): show "This is taking longer than expected." and keep polling every 30 s until the status is terminal or 15 minutes have passed since `createdAt`, with a "Check now" button that polls immediately; after 15 minutes polling stops and the button stays. Put 3 s / 150 s / 30 s / 15 min in 4.7 (frontend limits, `limits.ts`) if not there.
5. GUESS missing UI texts: add to 3.4 keys + English text for: dimension names (6 codes), experience levels with year ranges (from the 3.1 table), bands, severities, importance (`CORE` / `NICE_TO_HAVE`), analysis statuses, the FREE upgrade banner, FR-013's "taking longer" text, FR-023's two texts, FR-024 c8 text, FR-027 c6 text, the FR-030 state texts, CR-035's message and button, the Razorpay script-failure text (4.4). And a PDF label set in the backend template table: product name "SurfingBear Resume Optimizer", section headings (Overall score, Dimension scores, Missing skills, Suggested rewrites), dimension labels (same English names), level labels. Reuse texts already written in the FRs verbatim (single source: 3.4 holds them, FRs may quote).
6. GUESS duplicates in AI output: CMP-011 deduplicates missing skills case-insensitively (keep the first) before the catalogue filter and positions; a suggestion's offset is the first occurrence of `original_line` in the redacted text; ties (same severity, same offset) keep the AI's order.
7. GUESS FakeAiClient: it reads the redacted text and takes the first 5 non-empty lines that contain no `[EMAIL]`, `[PHONE]` or `[URL]` placeholder (fewer lines → fewer suggestions; zero is valid), so its output always passes the FR-011 checks.
8. CONTRADICTION NFR-015 "Input ≤ 12,000 tokens": reword as an expected size, not a cap: "about 7,500–10,000 input tokens for an English resume at the 30,000-character limit; not enforced per call — cost per call is bounded by `TEXT_MAX_CHARS` and `max_tokens`"; `ai_tokens_total` still measures it. Make 7.3 / other mentions consistent.
9. GUESS metric label values: list them: `ai_tokens_total{type="input"|"output"}`; `razorpay_webhook_events_total{type}` = the seven handled event types plus `other`; `billing_activation_seconds{path}` adds `checkout_expiry_fetch` (the FR-024 c3 expiry fetch in API-017) to `verification|webhook|reconciliation`; update NFR-020's definition to include it.
10. GUESS page security headers: the web middleware sets on every HTML response, with a per-request nonce (Next.js nonce support): `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-<nonce>' https://checkout.razorpay.com; connect-src 'self' <RO_API_BASE_URL> https://api.razorpay.com https://lumberjack.razorpay.com; frame-src https://api.razorpay.com https://checkout.razorpay.com; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'`, plus `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`. Put it in CR-037 (or the CR that owns web headers) and 4.8.3; a Playwright check that the Pro page loads Checkout with no CSP violation.
11. GUESS token lifetime: extend the platform assumption (A-18): the platform's access token lives 15 minutes; the platform keeps its own longer session, and its login page, for a user whose platform session is still valid, sets a fresh `sb_access` and redirects to `returnTo` at once without showing a form, so a 401 redirect is a short round trip. The module never refreshes tokens. Add one sentence on what the user loses: unsent form state (a chosen file must be picked again); a payment already completed in Checkout is not lost (webhook/reconciliation grant it, FR-029). Align 4.5's "15-minute token delay" with it.
12. GUESS prompt file selection: add `app.ai.prompt-version` / `ANTHROPIC_PROMPT_VERSION` (default `analysis-v1`) naming `prompts/<version>.txt`; `prompt_version` stored = this value; startup fails if the file is missing.

Then v1.3 change-log row ("Build test 6 of the improvement round: an independent agent logged 12 items (2 contradictions, 10 guesses, none high); all fixed. Last build test of the round."), header v1.3, checkers, commit "improvement: build test 6 fixes, v1.3 (12 items: Pro page states, 5xx alert, cancel dialog, long polling, UI texts, AI duplicates, fake AI lines, NFR-015 input size, metric labels, CSP, token lifetime, prompt version)". Report SHA and checker lines.

Address this before completing your current task.
```

- *tool:* Bash `cd /c/Users/user/spec-homework && git pull --ff-only 2>&1|tail -1; cd sachin && grep -n -o -E '(3\. The Pro page shows one of six states.{0,2600}|9\. The UI ask…`
- *tool:* Bash `cd sachin && sed -n 1221p spec.md | cut -c1-500; grep -n -o -E '(6\. Missing skills are compared only.{0,400}|If the script fails to…`
- *tool:* Write `m1.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Write `m2.py`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
- *tool:* Bash `cd sachin && SP=<scratchpad>…`
## Final report

## Result
All 12 build-test-6 items are fixed, and the spec header now reads v1.3. One local commit on `main` in `spec-homework`, staged with `git add sachin/spec.md` only. The message is plain with no trailers, and it is **not pushed**:

- `b299ae3` improvement: build test 6 fixes, v1.3 (12 items: Pro page states, 5xx alert, cancel dialog, long polling, UI texts, AI duplicates, fake AI lines, NFR-015 input size, metric labels, CSP, token lifetime, prompt version)

## What changed, by item
1. **Pro page states (FR-030 criterion 3):** (b) now requires plan `PRO`, status `ACTIVE` and no cycle-end cancel. (c) is plan `PRO` with a cycle-end cancel or status `CANCELLED`. The real count is seven states, corrected in FR-030 and in the 2.2 page table.
2. **5xx alert:** `Api5xxRateHigh` now counts only `api_errors_total{code="INTERNAL_ERROR"}` against all requests, still with a minimum of 3. A new warning `ServerBusy` fires when `SERVER_BUSY` errors exceed 20 in 5 minutes. NFR-013's pointer says so. The alert table never used the load balancer's own 5xx metric, so nothing else changed.
3. **Cancel dialog:** FR-028 criterion 9 has a second text for when `accessUntil` is null. Both texts are keys in 3.4.
4. **Long polling:** FR-013 criterion 4 now continues after 150 s with the "taking longer" text. It polls every 30 s with a "Check now" button and stops 15 minutes after `createdAt`. The four values (3 s / 150 s / 30 s / 15 min) are `limits.ts` constants in the 4.7 web table, with a new 7.3 row and an extended Playwright test row.
5. **UI texts in 3.4:** keys and English texts for dimensions, levels with year ranges, bands, severities, importance, analysis statuses, the FREE upgrade banner, the polling texts, FR-023, FR-024 criterion 8, FR-027 criterion 6, every FR-030 state and button, both cancel dialogs, CR-035's message and button, and the Razorpay script-failure text. FR texts are copied verbatim, and 3.4 says it is the single source. The PDF table gained the product name, section headings, dimension labels and level labels.
6. **AI duplicates:** CMP-011 deduplicates missing skills ignoring case, keeping the first. A suggestion's offset is the first occurrence of its line. Ties keep the AI's order.
7. **FakeAiClient:** uses the first 5 non-empty lines of the redacted text that contain no placeholder, so zero suggestions is possible and the output always passes the FR-011 checks.
8. **NFR-015 input size:** now an expected 7,500–10,000 tokens, not enforced per call; cost per call is bounded by `TEXT_MAX_CHARS` and `max_tokens`. No other mention of 12,000 remains.
9. **Metric labels:** 4.8.7 now lists the values for `ai_tokens_total{type}`, `razorpay_webhook_events_total{type}` (the seven handled types or `other`) and `billing_activation_seconds{path}`, which gains `checkout_expiry_fetch`. NFR-020 is updated to match.
10. **Page security headers:** CR-037 and 4.8.3 have the CSP with a per-request nonce, `X-Content-Type-Options` and `Referrer-Policy`, set by the middleware. CR-037's check adds a Playwright test that Checkout loads on the Pro page with no CSP violation.
11. **Token lifetime:** A-18 now covers:
    - the 15-minute platform token;
    - the silent re-login through the platform's own session;
    - that the module never refreshes tokens;
    - what a redirect loses (unsent form state, but not a completed payment).
    4.5's "Plan source" row is aligned with it.
12. **Prompt version:** new property `app.ai.prompt-version` / `ANTHROPIC_PROMPT_VERSION` (default `analysis-v1`) selects `prompts/<value>.txt`. It is the stored `prompt_version`, and a worker refuses to start if the file is missing. Added to 4.7, 4.6.2, 4.8.2 and TBL-006.

A v1.3 change-log row lists what changed, why (12 items: 2 contradictions, 10 guesses, none high; last build test of the round) and the affected IDs. No ID was added, renumbered or reused.

## Checker results
- ID checker:
  - CR 40, FR 30, NFR 21, CMP 23, TBL 19, API 22, ERR 26, A 21 defined.
  - `error codes listed: 26 | used with a status but not in 6.3: ['EXCELLENT', 'IGNORED', 'ORPHANED']` — the known false positives, so it still exits 1.
  - `FR/NFR rows: 51 | missing from matrix: []`
  - `API in summary not in body: [] | body not in summary: []`
- Mermaid: `12 diagrams, 0 failed`.
- Stale-wording search: no "six states", no "15-minute token delay" and no old fake-AI line rule left. The only "12,000" left is the v1.3 note on NFR-015 recording the old value.
