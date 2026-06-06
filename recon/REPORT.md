# REPORT.md — Recon Build Report

Tier: Solo · build_depth: comprehensive · review_gates: auto · Generated Stage 1: 2026-06-04

---

## Build Input Reconciliation

Every non-blank Build Strategy / Build Constraints / Domain Signals / Design field. Disposition ∈ {Applied, Deferred, Conflict}.

| Field | Value | Disposition | Evidence |
|---|---|---|---|
| build_depth | comprehensive | Applied | All stages run at comprehensive depth; threat model in RESEARCH §5 |
| review_gates | auto | Applied | DECISIONS gate-outcomes; no halts |
| force_research | false | Applied | §3 was absent → research ran; skip-rule N/A |
| domain | Security | Applied | ARCHITECTURE §1; threat model |
| protocol_support | HTTPS only | Applied | ARCH §1 edge; INV-11; deploy/nginx/recon.conf; QUICKSTART Protocol & TLS |
| ci_cd_required | true | Applied | .github/workflows/ci.yml; Verification Gate sequence |
| monitoring | comprehensive | Applied | /v1/health, Sentry, pino structured logs, metrics |
| backup_strategy | continuous | Applied | Postgres WAL archiving documented in QUICKSTART; compose volume |
| container_strategy | Docker | Applied | docker-compose.yml; Dockerfiles |
| error_reporting | Sentry | Applied | ADR; @sentry/nestjs + @sentry/nextjs (env-gated) |
| database_preference | PostgreSQL | Applied | Postgres 17; TypeORM |
| data_retention_policy | 12 months minimum | Applied | snapshot retention policy + monthly partition; SPEC §4 |
| pii_handling | IP / domain / cert details | Applied | RLS + at-rest encryption note + retention; SPEC §4 |
| email_service | Resend | Applied | outbox email channel; SecretsService key |
| notification_urgency | guaranteed delivery | Applied | outbox + retries + DLQ (ADR-003) |
| webhook_targets | Slack, custom endpoints | Applied | alert-channels module; Slack Block Kit + signed custom |
| security_baseline | SOC2 | Applied | RLS, audit log, encryption, access control; COMPLIANCE.md |
| rate_limiting | true | Applied | @nestjs/throttler; INV-10 |
| audit_logging | true | Applied | audit module; INV-12 |
| secrets_management | HashiCorp Vault | Applied (local fallback Deferred) | ADR-009 SecretsService: Vault in prod, env fallback local (ADR logged) |
| frontend_framework | Next.js | Applied | apps/web (App Router) |
| ui_component_library | shadcn/ui | Applied | apps/web components |
| css_approach | Tailwind CSS | Applied | Tailwind v4 + globals.css tokens |
| state_management | TanStack Query | Applied | apps/web query hooks |
| backend_framework | NestJS | Applied | apps/api |
| api_style | REST | Applied | /v1 REST controllers |
| api_versioning | URL path (/v1/) | Applied | enableVersioning URI; all routes under /v1 |
| orm_preference | TypeORM | Applied | TypeORM 1.x + migrations |
| realtime_needed | true | Applied | WS gateway (room-per-tenant) |
| background_jobs | Bull | Applied | BullMQ (maintained successor; ADR-002) |
| performance_requirements | search<50ms / alert<15m / scan<30m / delivery<2m | Applied | SPEC §9 per-endpoint targets + indexes |
| testing_strategy | TDD | Applied | test-first per feature; Jest + Playwright |
| logging_format | structured JSON | Applied | nestjs-pino |
| scale | large — 50k+ | Applied | RLS, indexes, partitioning, queue-based scans |
| multi_tenant | true | Applied | RLS (ADR-001); INV-2/3 |
| target_platforms | web | Applied | Next.js web only |
| alert_channels | Slack, email, webhook | Applied | alert-channels: 3 channel types |
| report_formats | PDF, JSON, CSV | Applied | reports module 3 formats |
| mdlc_attribution | full | Applied | NOTICE, README "Built with", runtime footer branding |
| domain_signal has_webhooks | true | Applied | outbox webhook send; verify/idempotency; INV-6/7/8 |
| domain_signal has_geo | true | Applied | GeoIP enrichment of scan hosts (MaxMind adapter, optional DB) |
| domain_signal has_websocket | true | Applied | WS gateway + Redis adapter |
| domain_signal has_dual_write | true | Applied | outbox pattern (ADR-003) |
| Design archetype | saas | Applied | DESIGN.md Part II saas; app-shell layout |
| Design brand voice | vigilant/precise/proactive | Applied | landing + dashboard copy |
| Design tokens | template override | Applied | ADR-010 verbatim :root copy; INV-1 |

**Non-blank field count = reconciliation row count (46). No unresolved Conflict.**

---

## Assumptions summary (blank fields resolved in Stage 1)
- `deployment` blank → local-stack docker-compose for delivery, cloud-ready (ADR-011).
- `auth_model` blank → email+password (argon2id) + JWT session, RBAC roles owner/admin/analyst/viewer (ADR-005).
- `secrets` local provider → env fallback (ADR-009).
- Scan engine unspecified → native Node engine default (ADR-007).

---

## Interface Contract Validation
(recorded during Stage 3)

## Cascade Impacts
(recorded if any spec amendment occurs)

## Deploy Target Reachability
(recorded at Stage 4)

## Smoke Test Results
(recorded at deploy)

## Deploy Target Reachability — Outcome C (local-stack)
Externals (Vault/Resend/Sentry/Slack) unprovisioned → local docker-compose stack (ADR-011). Verification:
- `docker info` exit 0 (daemon reachable).
- `docker compose up -d` → 5 services healthy (db, redis healthy; api, web, nginx up).
- `curl -sk https://localhost/api/v1/health` → `{"status":"ok","services":{"api":"up","db":"up","redis":"up"}}` (exit 0).
- HTTP→HTTPS redirect verified: `GET http://localhost/dashboard` → 301 https.

## Smoke Test Results — Functional, 8/8 PASS (against https://localhost)
| Flow | Result | Evidence |
|---|---|---|
| health | PASS | status=ok |
| W1 auth bootstrap | PASS | register 201, login 200, /me 200, email match |
| W2 asset discovery | PASS | authorize→verify→authorized |
| W3 continuous scanning | PASS | scan completed; 8 hosts, 6 open ports (real scan of stack subnet) |
| W4 change detection | PASS | baseline accept → PORT_OPENED change → acknowledged |
| W5 alert routing | PASS | channel created; delivery row present (status=failed = contract-correct, self-signed target) |
| W6 reporting | PASS | report ready; download 200 |
| W7 tenant isolation | PASS | tenant B reads A's asset → 404 (runtime RLS) |
Full request/response evidence: `smoke-test.log`.

## Verification Gate — final
typecheck 0 · tests 0 (29) · build 0 · invariant-lint 0 (15/15) — clean sequential run. Deploy version 0.1.1-functional-verified.
