# REPORT.md — Pulse

Build summary, metrics, reconciliation, and gate results.

---

## Build Input Reconciliation

One row per non-blank input field (Domain Signals + Build Constraints). Disposition ∈ {Applied, Deferred, Conflict}.
Non-blank field count: **27** · Reconciliation rows: **27** · Unresolved Conflicts: **0**.

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| domain_signals.has_webhooks | true | Applied | ARCH §3 (inbound webhooks module), §8; outbound notification fan-out (Slack/Teams/custom) SPEC §API /v1/channels, §5 W3 |
| domain_signals.has_websocket | true | Applied | ARCH §3 realtime service (Socket.IO, rooms-per-tenant); SPEC §UI live dashboard |
| domain_signals.has_geo | true | Applied | ARCH §2 `check_results.region`; SPEC §UI Surface "Map / Topology" screen |
| protocol_support | HTTPS only | Applied | ARCH §6 (HTTPS-only, HSTS, HTTP→HTTPS redirect); QUICKSTART Protocol & TLS |
| ci_cd_required | true | Applied | ARCH §7 GitHub Actions pipeline (.github/workflows/ci.yml: lint, typecheck, test, build, scan, invariant-lint) |
| monitoring | comprehensive | Applied | ARCH §6 observability (Pino structured logs, /api/health, metrics counters, queue depth) |
| container_strategy | Docker | Applied | docker-compose.yml (db, redis, api, worker, web); per-service Dockerfiles |
| database_preference | PostgreSQL | Applied | ARCH §2 PostgreSQL 16 + Drizzle; partition-ready metrics |
| notification_urgency | guaranteed delivery | Applied | ADR-0008; ARCH §3 notifier (BullMQ attempts+backoff+DLQ, idempotency INV-2) |
| webhook_targets | Slack, Teams, custom webhooks | Applied | ARCH §3 notification channels; SPEC §API /v1/channels; Teams via Power Automate (ADR-0007) |
| sms_service | Twilio | Applied | ARCH §3 SMS channel (Twilio SDK); inbound signature verify (INV-4) |
| security_baseline | SOC2, OWASP Top 10 | Applied | ARCH §6 + §9 invariants; RESEARCH §5 threat model; COMPLIANCE.md |
| rate_limiting | true | Applied | ARCH §6 Redis token-bucket per tenant+IP; INV-7; SPEC §4 |
| audit_logging | true | Applied | ARCH §2 `audit_logs` table; §6 audit middleware on mutations |
| secrets_management | environment variables | Applied | ARCH §6 env-var secrets + .env.example; Pino redaction; channel secrets AES-GCM (ADR-0009) |
| api_style | REST | Applied | ARCH §3 Express REST; SPEC §API Surface |
| api_versioning | URL path (/v1/) | Applied | ARCH §3 `/v1` router mount; SPEC §API Surface |
| realtime_needed | true | Applied | ARCH §3 Socket.IO realtime push |
| background_jobs | job queue | Applied | ARCH §3 BullMQ (scheduler, checker, notifier queues) |
| performance_requirements | sub-minute detection / MTTD<5m / API<200ms / 99.9% | Applied | SPEC §9 perf targets as per-feature acceptance criteria; ARCH §6 |
| logging_format | structured JSON | Applied | ARCH §6 Pino JSON; INV (manual) |
| scale | large — 50k+ | Applied | ARCH §2 (indexed org_id, partition-ready metrics), §3 (stateless autoscaled workers, jitter) |
| multi_tenant | true | Applied | ARCH §2 org_id everywhere; scopedDb wrapper + RLS (INV-1); §9 |
| target_platforms | web | Applied | apps/web React SPA |
| alert_channels | email, SMS, Slack, Teams, webhooks | Applied | ARCH §3 five channel adapters; SPEC §API /v1/channels |
| report_formats | JSON, CSV | Applied | SPEC §API /v1/reports (?format=json|csv) |
| mdlc_attribution | structural | Applied | NOTICE file + README "Built with" + package keywords (Stage 4 reconciliation) |

Key-decision fields (`deployment`, `backend_language`, `auth_model`) were blank in this dashboard-wizard RESEARCH.md → resolved as assumptions (logged: backend_language = TypeScript/Node; deployment = Docker Compose local-stack with cloud-ready compose; auth_model = server-side sessions, ADR-0005).

## Pre-build assumptions summary
(to be finalized at Stage 3 pre-build checkpoint) — TypeScript/Node backend; Docker Compose deploy; server-side session auth; Postgres 16; Redis/Valkey queue; agentless-first metric ingest.

## Deploy Target Reachability
(pending `ship`)

## Verification Gate
(pending Stage 4)

## Reviewer Gate
(pending Stage 4)

## Smoke Test Results
(pending)

## Security baseline summary
(pending Security Audit Gate)

## Verification Gate
All quality scripts exit 0: build ✓ · typecheck ✓ · lint ✓ · unit tests (shared 20 / worker 15) ✓ · api integration 5/5 (RUN_DB_TESTS, live PG+Redis) ✓ · production build ✓ · invariant-lint 11/11 + 4 manual ✓. Verdict: PASS.

## Reviewer Gate
Fresh-context general-purpose reviewer (Claude Code Task tool) fed RESEARCH/ARCHITECTURE/SPEC + working tree. Per-item: (1) feature completeness PASS, (2) tenant isolation PASS, (3) security controls PASS, (4) data model & migrations PASS, (5) error contract PASS, (6) alert/worker logic PASS, (7) runnable/documented ADVISORY (env-var naming drift CHECK_CONCURRENCY→CHECKER_CONCURRENCY — FIXED in .env.example). OVERALL: PASS.

## Interface Contract Validation (final)
API endpoint coverage ✓ (35 endpoints smoke-verified) · frontend↔backend contract ✓ · UI coverage ✓ (16 routes routable, 6 workflows e2e, INV-11) · auth middleware wiring ✓ · DB schema alignment ✓ · env completeness ✓.

## Pre-Delivery Completeness Gate
Route & handler completeness ✓ · data layer ✓ · validation ✓ · CRUD ✓ · auth flow ✓ · UI surface completeness ✓ · environment & startup completeness ✓.

## Startup verification
api + worker boot clean (structured logs), /api/health 200 {postgres:up, redis:up}, end-to-end check pipeline verified (monitor → probe → result → status). 21/21 functional smoke at API level.

## Security baseline summary
SOC2 / OWASP Top 10 baseline implemented; Security Audit PASS (0 CRITICAL/HIGH residual); COMPLIANCE A9/B2/C0.

## Deploy Target Reachability — Outcome C (local-stack)
No cloud target configured → local-stack via `docker compose up -d --build`. Externals (Twilio/SMTP) optional/unprovisioned (notifier degrades; Slack/Teams/webhook channels post directly; Twilio token wired for inbound-webhook signature verification on the local stack). Verification: `docker compose ps` → db/redis/api healthy, web/worker up; exit 0.

## Drift Detection Gate
Invariant lint 11/11 machine-checkable pass + 4 manual verified → 0 drifts. Auto-proceeded to deploy.

## Smoke Test Results
- **Functional Smoke (BUILD.md): 21/21 PASS** against the deployed HTTPS stack (https://localhost:8443): health, W1 auth bootstrap, multi-tenant isolation (404), W2 CRUD, background-job execution, W3 alerting config, W4 incident lifecycle (open→ack→resolve), W5 reports+CSV export, W6 API-key + metric ingest, inbound webhook signature-verify + idempotency, ≥16KB large-header resilience. Saved to `smoke-test.log`.
- **UI e2e (Playwright): 6/6 PASS** — every Key Workflow W1–W6 driven end-to-end through the rendered browser to its terminal step (CSV download, metric gauges, incident resolution).
- VERSION marked `0.1.2` functional-verified.

## Final reconciliation
MDLC attribution: NOTICE + README "Built with" + package keywords ✓. All ADRs reconciled against the codebase. Full suite green; production images ship without devDependencies (`npm prune --omit=dev`).
