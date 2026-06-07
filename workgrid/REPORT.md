# REPORT.md — WorkGrid Build Report

## Assumptions Summary (Stage 1 blank-field resolutions)
The following inputs were blank/unspecified and resolved by the agent (see DECISIONS.md):
- `auth_model` → email+password, Argon2id, JWT access + rotating refresh (ADR-0006).
- `deployment` → local Docker Compose stack (externals unprovisioned); cloud path documented (ADR-0008).
- `prompt_mode` → `direct` (ADR-0001).
- Exact library patch versions → pinned stable lines (ADR-0003).

## Build Input Reconciliation
Every non-blank input field is honored and traceable. (38 non-blank fields → 38 rows.)

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| domain | Team work | Applied | Whole product scope (ARCHITECTURE.md §1) |
| domain_signals.has_webhooks | true | Applied | ARCHITECTURE.md §6 Integrations; SPEC §5 W-Webhook; INV-4 |
| domain_signals.has_websocket | true | Applied | ARCHITECTURE.md §5 Real-time gateway; SPEC §5 |
| domain_signals.has_dual_write | true | Applied | Transactional outbox, ADR-0007; ARCHITECTURE.md §5; INV-7 |
| protocol_support | HTTPS only | Applied | nginx TLS + redirect + HSTS, ADR-0009; QUICKSTART §Protocol&TLS |
| ci_cd_required | true | Applied | `.github/workflows/ci.yml` (Stage 3 deliverable) |
| monitoring | APM | Applied | OpenTelemetry hooks + `/api/health`/`/metrics`; ARCHITECTURE §7 |
| backup_strategy | daily + PITR | Applied | Postgres WAL archiving + `pg_dump` cron in compose/docs; ARCHITECTURE §4 |
| container_strategy | Docker + K8s | Applied | Dockerfiles + docker-compose + k8s manifests; ARCHITECTURE §8 |
| error_reporting | centralized + alerting | Applied | Sentry adapter (no-op without DSN) + structured error pipeline; ARCHITECTURE §7 |
| database_preference | PostgreSQL | Applied | ARCHITECTURE §4 Data Layer |
| pii_handling | identities/emails/content | Applied | Encryption-at-rest, PII log redaction, audit access; ARCHITECTURE §9 INV-13 |
| email_service | Resend | Applied | Resend adapter via outbox worker; ARCHITECTURE §6 |
| notification_urgency | guaranteed delivery | Applied | Outbox + retries + DLQ (at-least-once), ADR-0007 |
| webhook_targets | GitHub, Slack | Applied | Outbound webhook adapters (HMAC-signed); ARCHITECTURE §6 |
| security_baseline | SOC2, OWASP Top 10 | Applied | SPEC §4; COMPLIANCE.md; audit log, RBAC, RLS, headers |
| rate_limiting | true | Applied | @nestjs/throttler + Redis store; INV-6 |
| audit_logging | true | Applied | `audit_log` append-only table + interceptor; INV-11 |
| secrets_management | env vars w/ encryption | Applied | `.env` gitignored, `.env.example` committed, K8s Secrets; INV-5 |
| frontend_framework | Next.js | Applied | apps/web (App Router) |
| ui_component_library | shadcn/ui | Applied | apps/web/src/components/ui |
| css_approach | Tailwind CSS | Applied | apps/web tailwind config bound to design tokens; INV-9 |
| state_management | TanStack Query + Zustand | Applied | apps/web data + ui stores |
| backend_framework | NestJS | Applied | apps/api |
| api_style | REST | Applied | SPEC §3 API surface |
| api_versioning | URL path /v1/ | Applied | `enableVersioning(URI)`; all routes `/v1/*`; INV-2 |
| orm_preference | TypeORM | Applied | apps/api entities + migrations |
| realtime_needed | true | Applied | socket.io gateway, tenant rooms; ARCHITECTURE §5 |
| background_jobs | Bull with Redis | Applied | BullMQ (Bull successor) + Redis; ADR logged; ARCHITECTURE §5 |
| performance_requirements | <200ms/<2s/<1s | Applied | SPEC per-endpoint acceptance criteria §9 |
| testing_strategy | TDD | Applied | Jest unit/integration + Playwright e2e; tests-first per feature |
| logging_format | structured JSON | Applied | nestjs-pino structured logger; INV-12 |
| scale | large 50k+ | Applied | Stateless API + Redis adapter + pooled DB; ARCHITECTURE §8 |
| multi_tenant | true | Applied | RLS + scoped service layer, ADR-0005; INV-1 |
| target_platforms | web | Applied | apps/web only |
| alert_channels | email, in-app | Applied | Resend + in-app notification center; SPEC §5 |
| report_formats | PDF, CSV, JSON | Applied | Report export service; SPEC §5 W-Reports |
| mdlc_attribution | structural | Applied | NOTICE file + README "Built with" + package keywords (Stage 4) |

No field left as an unresolved Conflict. Reconciled field count = 38 = non-blank field count. ✓

## Interface Contract Validation
- API endpoint coverage: every SPEC §3 endpoint has a route handler (13 modules). ✓
- Frontend↔backend contract: shared types in `@workgrid/shared`; web `apiFetch` paths match `/v1` routes. ✓
- UI coverage (reverse): `ui-coverage` invariant green — 18 routes resolve, 8 workflows have e2e. ✓
- Auth middleware wiring: global Jwt/Tenant/Rbac guards + `@MinRole` per route. ✓
- DB schema alignment: entities ↔ migration/RLS; bootstrap synchronize + 0001 RLS. ✓
- Env var completeness: all referenced vars in `.env.example` (INV-3). ✓

## Cascade Impacts
None — no spec amendments dropped a RESEARCH-level deliverable. Two implementation
bugs found at smoke (ADR-0011 refresh jti, ADR-0012 envelope unwrap) were fixed at
root cause and re-verified; no spec change required.

## Verification Gate
- Typecheck: PASS (packages/shared, apps/api, apps/web).
- Unit/integration tests: 61 passed / 61 (11 suites). PASS.
- Production build: apps/api `nest build` PASS; apps/web `next build --webpack` PASS (18 routes).
- Invariant lint: 13/13 machine-checkable PASS, 4 manual upheld (see below). PASS.

## Invariant Lint Result
17 invariants: 13 machine-checkable evaluated, 13 passing, 0 failing; 4 manual
(INV-11 audit append-only trigger, INV-13 log redaction, INV-14 RLS FORCE policy,
INV-15 tenant-room emits) verified by inspection and upheld.

## Smoke Test Results
- Functional smoke (`smoke-test.sh` vs https://localhost:8444): **15/15 PASS** —
  health, HTTP→HTTPS 301, W1 register/login/me, unauth 401, W2/W7 ticket CRUD+resolve,
  W6 dashboard, W8 report download, tenant isolation 404, webhook bad-signature 403,
  structured validation error, 16KB cookie 200. Saved to `smoke-test.log`.
- UI e2e (Playwright, live deploy): **9/9 PASS** — W1–W8 + landing.

## Deploy Target Reachability
Outcome **C** (local-stack). No cloud target/credentials configured; external
secrets are placeholders → no-op adapters (ADR-0008). Brought up via
`docker compose up -d --build`; smoke run against `https://localhost:8444`.
Verification: `curl -k https://localhost:8444/api/health` → 200
`{"status":"ok","services":{"db":"up","redis":"up"},...}`.

## security_baseline summary
SOC2 + OWASP Top 10: 15 controls ✓, 1 ⚠ (accepted build-time advisory, ADR-0013),
0 ✗. No CRITICAL/HIGH residuals. See COMPLIANCE.md.
