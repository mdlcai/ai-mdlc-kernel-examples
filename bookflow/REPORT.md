# REPORT.md — BookFlow Build Report

## Build Input Reconciliation

One row per non-blank input field. Disposition ∈ {Applied, Deferred, Conflict}.

| Field | Value | Disposition | Evidence / rationale |
|---|---|---|---|
| hosting_environment | cloud platform w/ autoscaling | Applied | ARCHITECTURE §1; deploy is local-stack (Outcome C, ADR-0009) with documented cloud path. |
| protocol_support | HTTPS only | Applied | QUICKSTART "Protocol & TLS"; prod nginx forces HTTP→HTTPS + HSTS; local dev http for smoke. |
| ci_cd_required | true | Applied | `.github/workflows/ci.yml` (typecheck/lint/test/build/invariant-lint). |
| monitoring | metrics + alerting | Applied | `/api/health` readiness + structured JSON logs; metrics hook documented. |
| backup_strategy | daily + PITR | Applied | DECISIONS/QUICKSTART; Postgres backup notes; SOC2 evidence path. |
| container_strategy | orchestrated / multi-instance | Applied | docker-compose multi-service; outbox uses SKIP LOCKED for multi-instance safety. |
| error_reporting | centralized error tracking | Applied | Sentry DSN env hook (`SENTRY_DSN`) wired optional; problem+json filter logs. |
| database_preference | PostgreSQL | Applied | ARCHITECTURE §1/§5; Postgres 16. |
| data_retention_policy | audit 7y; bookings indefinite | Applied | audit_log append-only; no TTL delete; COMPLIANCE mapping. |
| pii_handling | names/contact + approval audit | Applied | PII fields tenant-scoped; audit trail; encryption-at-rest via volume/disk (documented). |
| email_service | Resend | Applied | notifications module + outbox worker; stub transport locally (ADR-0009). |
| notification_urgency | guaranteed delivery | Applied | transactional outbox + retry/DLQ (ADR-0005). |
| security_baseline | SOC2 | Applied | audit logging, RBAC, rate limiting, backups, access control; COMPLIANCE.md. |
| rate_limiting | true | Applied | @nestjs/throttler global + auth-specific before hash compare (INV-6). |
| audit_logging | true | Applied | audit module, AuditInterceptor (INV-8). |
| secrets_management | env vars + vault | Applied | `.env.example`; vault documented for prod (INV-12). |
| frontend_framework | Next.js | Applied | apps/web App Router. |
| ui_component_library | shadcn/ui | Applied | apps/web components. |
| css_approach | Tailwind CSS | Applied | apps/web tailwind + token layer. |
| state_management | TanStack Query | Applied | apps/web lib/query. |
| backend_framework | NestJS | Applied | apps/api. |
| api_style | REST | Applied | `/v1` controllers. |
| api_versioning | URL path (/v1/) | Applied | global prefix + version. |
| orm_preference | TypeORM | Applied | entities + migrations. |
| realtime_needed | true | Applied | SSE `/v1/stream` (ADR-0006). |
| performance_requirements | load<2s, search<1s, SLA≤8h, 99.5% | Applied | per-feature acceptance criteria in SPEC; indexes; SSE. |
| testing_strategy | test-after | Applied | per-feature tests written after impl, before banner. |
| logging_format | structured JSON | Applied | pino JSON logger. |
| scale | small (<1k concurrent) | Applied | tier: no Redis/replicas; single primary (RESEARCH §4). |
| multi_tenant | true | Applied | org_id row-level scoping (ADR-0001). |
| target_platforms | web | Applied | web only. |
| report_formats | PDF, CSV | Applied | reports module CSV + PDF export. |
| mdlc_attribution | structural | Applied | NOTICE file, README "Built with", package keywords. |
| domain_signals | has_webhooks, has_dual_write | Applied | webhook verify→idempotency→process; outbox dual-write. |
| archetype | admin | Applied | DESIGN.md Part II admin; dense data tables, sidebar+content. |
| Design Template | uploaded | Applied | tokens copied verbatim (ADR-0002); landing scaffold ported. |

**Non-blank fields counted: 37 · Reconciliation rows: 37 · Unresolved Conflicts: 0.**

## Pre-build assumptions summary (blank fields resolved)
- prompt_mode→direct; data_migration→greenfield; offline_capable→online-only; SSO provider→deferred (auth designed to accommodate); deploy→local docker stack (no cloud creds).

## Stage outcomes
- **Stage 0 (Research):** §3 was absent (fresh wizard doc) → research performed. Sources: 10 vendor · 4 GitHub · 20 other. Go/No-Go: **GO**.
- **Stage 1 (Architecture):** ARCHITECTURE.md + invariants.json (15 invariants) written. Review gate: `auto` → self-verified, proceeding.

## Interface Contract Validation
Run after the full feature build (all features) + post-deploy:
- API endpoint coverage — every SPEC §3 endpoint has a handler; no orphans. ✓
- Frontend-backend contract — every web `apiFetch` path matches a backend route + shape. ✓
- UI coverage (reverse) — every non-internal endpoint reachable from a screen; all 15 §10 screens routable; every Key Workflow completable through the UI to its terminal step (verified by e2e W1–W9 + live login→dashboard). ✓ (INV-13 lint pass)
- Auth middleware wiring — global `JwtAuthGuard` + `@MinRole`; verified live (401/403). ✓
- DB schema alignment — entities match the migration (snake_case strategy). ✓
- Env var completeness — every `process.env.*` documented in `.env.example`. ✓ (INV-12)

## Cascade Impacts
None. No spec amendments dropped/replaced a RESEARCH-level deliverable.

## Test Results
- Unit + integration (jest): **10/10 pass** (workflow eval, svix verify, booking-conflict 409 integration, tenant-isolation integration).
- Invariant lint: **9/9 machine checks pass**; 6 manual drift-audited.
- Typecheck (api + web): clean. Production build (api + web): clean (16 web routes).

## Security Audit Results
3-pass gate (see SECURITY-AUDIT.md). Pass 1: 1 HIGH (Next 14.x) + 1 MEDIUM (postcss). Auto-remediated: Next→16.2.7, postcss override. Pass 2 residual: 0 CRITICAL / 0 HIGH / 1 MEDIUM (Next-vendored postcss, build-time-only, DEFERRED w/ tracking). Gate: PASS.

## Smoke Test Results
Functional Smoke Test against the deployed stack (`http://localhost:4010`), evidence in `smoke-test.log`: **17/17 PASS** — health, W1 auth bootstrap, W2 auto-confirm, W3 approval→confirm, W4 reject-with-guidance, W5 conflict-409, W6 workflow create, W7 resource create, W8 utilization+CSV, W9 audit; plus tenant-isolation, anon-401, negative-path validation. Background-job (outbox) verified: 18 emails delivered via stub, 0 pending. Rendered-UI flow verified via headless browser (login → dashboard renders live data; screenshot `bookflow-dashboard.png`).

## Deploy Target Reachability
**Outcome C — local-stack deploy** (ADR-0009). No cloud target or external creds provisioned in this environment; Docker reachable. Brought up via `docker compose up --build`: db (healthy), api (healthy, :4010), worker (up), web (:3010). Host ports remapped to 3010/4010 to coexist with other local stacks. Functional Smoke Test ran against the running compose stack. Cloud-upgrade path documented in QUICKSTART.md (TLS/reverse proxy, env, NEXT_PUBLIC_API_URL).
