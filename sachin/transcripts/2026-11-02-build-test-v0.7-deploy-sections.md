# Build test of the v0.7 deployment and operations sections

Exported transcript, 2026-11-02. Claude Code agent run on its own, reading only `sachin/spec.md`. Tool results (the spec text it read) are left out; each tool call is shown as one line.

## Prompt

```text
You are a senior developer doing a build test of one spec. Read ONLY this file: sachin/spec.md. Do NOT open, list or grep any other file or folder under spec-homework (other people's work is there and must not be read). Do not edit anything.

Focus on the parts changed in version v0.7 (marked *(v0.7)*): CR-034 to CR-040 (sections 1.1, 1.5, 1.7, 1.8, 1.9), assumptions A-17 to A-20 (3.3), CMP-023 and the component diagram (4.1, 4.2), 4.3.4, 4.3.7, 4.4, 4.5, 4.7, the whole of section 4.8 (4.8.1 to 4.8.7), 6.1, NFR-005, NFR-013, NFR-014, NFR-015, and sections 7.2, 7.3 and the v0.7 change-log row. Read the rest of the spec as needed for context.

Imagine you must configure and deploy every environment (local, dev, staging, prod) for both the Spring Boot backend and the Next.js frontend, write the GitHub Actions pipelines, the Terraform, the alerting, and the frontend 401 handling, using only this spec. List every place where you would have to GUESS something that changes what gets built, and every CONTRADICTION between two parts of the spec (including between v0.7 text and older text that it made stale). Ignore wording and style. Also flag any factual error you are confident about (for example a wrong AWS, Next.js, Terraform or Prometheus fact).

For each item: section/ID, what is missing or contradictory, severity (high/medium/low), and a one-line suggested fix. End with the total count. Keep the report under 900 words.
```

- *tool:* Read `spec.md`
- *tool:* Grep `^#{1,5} `
- *tool:* Read `spec.md` lines 1-195
- *tool:* Read `spec.md` lines 388-587
- *tool:* Read `spec.md` lines 726-825
- *tool:* Read `spec.md` lines 1013-1057
- *tool:* Read `spec.md` lines 1198-1657
- *tool:* Read `spec.md` lines 1198-1417
- *tool:* Read `spec.md` lines 1418-1657
- *tool:* Read `spec.md` lines 2185-2266
- *tool:* Read `spec.md` lines 3092-3171
- *tool:* Grep `liveness|NEXT_PUBLIC|Local JWKS|dev-token|ro-api.surfingbear.com|\.surfingbear\.com|Sentry|uptime|8081|ECS_CONTAINER|cloudwatch|ADOT|prometheus`
- *tool:* Read `spec.md` lines 216-315
- *tool:* Grep `GRANT|DEFAULT PRIVILEGES|ro_app`
- *tool:* Read `spec.md` lines 1958-1973
- *tool:* Read `spec.md` lines 2060-2074
## Final report

I read only sachin/spec.md and edited nothing. I found 25 problems: 4 high, 11 medium, 10 low. G = I would have to guess, C = two parts of the spec contradict, F = a factual error.

**High**

1. **4.8.4 OIDC, dev deploy (C/F).** `ro-ci-deploy-dev` trusts only `ref:refs/heads/main`, but a GitHub Environment `dev` exists in every repo. If the deploy job uses `environment: dev`, the token's `sub` is `repo:…:environment:dev` and assuming the role fails. Fix: say the dev deploy job uses no Environment, or trust `environment:dev`.
2. **4.8.4 step 0 (G).** Staging and prod deploys must check that the image exists in ECR, which lives in `sb-tools`. No `sb-tools` role trusts the `environment:staging` or `environment:prod` subjects: the push role trusts `main` only, and the pull role trusts the web repo only. Fix: add `ro-ci-ecr-read` (`ecr:DescribeImages`) that trusts those subjects.
3. **4.8.4/4.8.5 plan role (G, security).** `ro-ci-tf-plan-prod` trusts `pull_request`. The prod state holds the RDS master password, the Redis AUTH token and the origin secret. Any PR branch can therefore read prod secrets without a reviewer. Fix: plan staging and prod only on `main` or behind the Environment, or keep secrets out of state.
4. **4.8.7 counter alerts (F).** `increase()` gives 0 for a counter series' first sample. `RetentionViolation`, `BillingReconciliationFix`, `ProHeavyUse`, `JwksFetchFailing`, `WebhookSignatureFailures` and `RateLimiterUnavailable` count rare events. Counters created on first increment, plus a new `task_id` per task, mean the first event per task never alerts. A daily retention violation can stay silent forever. Fix: register every counter and label value at 0 at startup.

**Medium**

5. **NFR-014 / 4.8.7 (F).** In the `awsxray` exporter, span attributes become X-Ray metadata, which cannot be searched. "Find the trace by `requestId`" then fails. Fix: `indexed_attributes: [requestId]`.
6. **4.8.5 step 3 (F).** "Random ≥32 chars" for `db-owner` breaks RDS: a master password may not contain `/ ' " @` or a space. Fix: give the allowed character set.
7. **4.8.5 step 5 (F).** Amazon Managed Grafana service-account tokens live at most 30 days, so the stored `ro/tools/grafana-token` expires and `envs/tools` applies start failing. Fix: add a rotation step or mint the token in the pipeline.
8. **4.8.3/4.8.5 certificates (G).** An HTTPS listener cannot use a certificate that is not issued yet, and the validation records come from the platform only after Terraform outputs exist. The first full apply fails. Fix: document a third pass (certificates → platform adds records → full apply).
9. **4.8.6 vs 4.8.5 (C).** The connection budget assumes `ro-api` and `ro-worker` roll one after the other. A Terraform task-definition change rolls both at once: 12×20 + 20×10 = 440, above the ~400 connections of prod. Fix: apply services one at a time, or resize the pools or the instance.
10. **4.8.7 routing (G).** The prod critical topic sends Chatbot custom JSON to PagerDuty, whose SNS/CloudWatch integration expects alarm JSON. Whether incidents open and resolve is unknown. Fix: give PagerDuty its own topic and payload.
11. **CR-016 (G).** Nothing is named as the setter of HSTS. The ALB terminates TLS, so Spring sees HTTP and does not emit it. Fix: name the ALB response-header setting or a Spring setting, and the platform for `www`.
12. **4.8.6/CR-029 worker shutdown (G).** Rolling deploys and scale-in stop workers in the middle of AI calls of up to 2×45 s. ECS waits 30 s by default and at most 120 s, so every deploy fails analyses. Fix: `stopTimeout` 120 and a graceful drain of `XREADGROUP`.
13. **4.8.5 inputs (G).** The inputs list leaves out values the spec uses elsewhere:
    - `pagerduty_endpoint`
    - the `sb-tools` account id and ECR URI
    - Chatbot Slack workspace and channel ids
    - the Grafana role ARN that `ro-grafana-read` must trust
    - ADOT and canary runtime versions
    - ECS health-check grace period

    Fix: list every one as an input.
14. **NFR-013 / dashboard (G).** "Error rate by code" and "5xx by `error.code`" have no metric behind them: `http.server.requests` carries no `error.code`. Fix: define `api_errors_total{code}` or drop the panel.
15. **CR-036 build check (G).** It does not say whether the `surfingbear` grep is case-sensitive. The client bundle contains the `X-Requested-With: SurfingBear` header value (CR-018), so a case-insensitive grep fails every build. Fix: make it case-sensitive and grep for the domains.

**Low**

16. **CR-036 (G).** `register()` calls `process.exit(1)`. If Next runs instrumentation during `next build`, the build breaks. Fix: skip the check when `NEXT_PHASE` is `phase-production-build`.
17. **CR-034 / 6.1 / CR-001 (C).** CR-001 says "every request is authenticated", which suggests a non-local `/local/login` without a token returns 401. `LocalLoginAbsentIT` expects 404. Fix: state that the security chain covers only the API prefixes and other paths return 404.
18. **4.7 vs 4.8.2 (C, stale).** `app.env` says the bucket name comes from `<env>`, while 4.8.2 requires `S3_BUCKET`. Fix: keep one.
19. **4.8.7 (C).** It says "both rate alerts" need a minimum event count. There are three, and `Api5xxRateHigh` has no minimum.
20. **CR-011 vs 4.8.7 (C).** The web server logs at FATAL, a level CR-011 does not define.
21. **CR-040 / 4.8.7 (G).**
    - The canary's Lambda logs go to `/aws/lambda/cwsyn-*`, which is not a `/ro/<env>/` group and has no retention set.
    - "The load balancer already sees every request" is untrue: ALB access logs are never enabled.
22. **A-20 / 4.8.5 (F).** Slack must be authorised in AWS Chatbot once per AWS account (dev, staging, prod), not once overall.
23. **Dashboard worker task count (G).** It needs ECS Container Insights, which is never enabled.
24. **CR-038 (G).** ECR tag immutability makes a re-run of a merge workflow fail at the push. Fix: skip the push when the tag exists.
25. **4.8.2 (G).** `SPRING_PROFILES_ACTIVE` is missing from the "every variable" table, though `@Profile("local")` depends on it. Fix: say how the profile is set.

These facts checked out correct: Next.js 15.5 Node.js middleware, standalone output, `basePath` handling and default 307 redirect; the CloudFront managed policies; the ALB TLS policy name and header conditions; the RDS `max_connections` formula; the ElastiCache AUTH character rule; the Fargate size combinations; AMP 150-day and X-Ray 30-day defaults; the ADOT and Alertmanager SNS syntax; Terraform 1.9 locking with DynamoDB.

**Total: 25 (4 high, 11 medium, 10 low).**
