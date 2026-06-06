# REPORT.md — ServiceHub Build Report

## Build Input Reconciliation

One row per non-blank input field from RESEARCH.md (Build Constraints + domain
signals). Disposition ∈ {Applied, Deferred (ADR), Conflict}.

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| `protocol_support` | HTTPS only | Applied | ARCHITECTURE §6/§7; QUICKSTART §Protocol & TLS; ADR-10 |
| `ci_cd_required` | true | Applied | `.github/workflows/ci.yml` |
| `monitoring` | application performance monitoring | Applied | OpenTelemetry SDK; ARCHITECTURE §6 |
| `backup_strategy` | daily automated backups + PITR | Applied | ops cron + ARCHITECTURE §6; ADR-09 retention |
| `container_strategy` | Docker w/ orchestration | Applied | `docker-compose.yml`; per-service images |
| `error_reporting` | centralized error tracking | Applied | `@sentry/node` integration |
| `database_preference` | PostgreSQL | Applied | TypeORM + Postgres; ARCHITECTURE §4 |
| `pii_handling` | names, emails, dept, device ids | Applied | RLS + field-level access; COMPLIANCE.md |
| `notification_urgency` | guaranteed delivery | Applied | Transactional outbox + queue; ADR-04 |
| `webhook_targets` | monitoring, external ticketing | Applied | inbound + outbound webhook modules |
| `security_baseline` | [SOC2, OWASP Top 10] | Applied | COMPLIANCE.md; threat model RESEARCH §5 |
| `rate_limiting` | true | Applied | `apps/api/src/middleware/rateLimit.ts` (INV-6) |
| `audit_logging` | true | Applied | append-only `audit_logs` (INV-12) |
| `secrets_management` | environment-based secrets | Applied | `.env.example`; boot-time env validation (INV-7) |
| `frontend_framework` | React | Applied | `apps/web` (React + Vite) |
| `api_style` | REST | Applied | Express REST controllers |
| `api_versioning` | URL path (/v1/) | Applied | all routes mounted under `/v1` |
| `orm_preference` | TypeORM | Applied | ADR-02; INV-2 |
| `realtime_needed` | true | Applied | Socket.IO + Redis adapter; ARCHITECTURE §3 |
| `background_jobs` | job queue (routing + notifications) | Applied | BullMQ queues; worker service |
| `performance_requirements` | API<200ms, dash<2s, submit<1s | Applied | SPEC perf acceptance; comprehensive load test (Stage 4) |
| `testing_strategy` | test-after | Applied | tests written per feature after impl (kernel loop) |
| `logging_format` | structured JSON | Applied | pino JSON logger |
| `scale` | medium — 1k-50k | Applied | shared-schema + RLS tier; ADR-05 |
| `target_platforms` | [web] | Applied | web SPA only |
| `alert_channels` | [email, in-app] | Applied | notify worker: email (SMTP) + in-app |
| `report_formats` | [PDF, CSV] | Applied | PDFKit + fast-csv export module |
| `mdlc_attribution` | structural | Applied | NOTICE file, package keywords, README "Built with" |
| `domain_signals` | has_webhooks, has_image_uploads | Applied | webhook modules; upload (Multer+sharp, EXIF strip) |

**Non-blank field count: 29 · Reconciliation rows: 29 · Unresolved Conflicts: 0.**

## Assumptions (blank fields resolved at Stage 1)
SLA calendar (ADR-06), email provider (ADR-07), monitoring webhook format
(ADR-08), data retention (ADR-09). See DECISIONS.md.

## Cascade Impacts
_None recorded yet._

## Interface Contract Validation
- API endpoint coverage: every SPEC §2 endpoint has a handler (auth, tickets,
  approvals, dashboards, reports, admin, webhooks, health). ✓
- Frontend-backend contract: web `src/api/*` hooks call the documented `/v1` paths. ✓
- UI coverage (reverse): 19 routes routable; 7 Key Workflows have UI e2e tests
  (INV-10 ui-coverage PASS). ✓
- Auth middleware wiring: every protected route uses requireAuth + requirePermission. ✓
- DB schema alignment: entities ↔ migration columns (snake_case strategy). ✓
- Env completeness: every env reference present in `.env.example` (INV-7). ✓

## Verification Gate
typecheck=0 · lint=0 · test=0 (25 tests: api 14 + web 11) · build=0 ·
invariant-lint=0 (14/14: 9 machine-checkable + 1 ui-coverage + 4 manual).

## Security Audit Gate
- Pass 1: 1 HIGH (nodemailer ≤8.0.4 SMTP-injection/DoS), 0 secrets in source,
  8 per-feature categories reviewed (tenant isolation, authn/z, rate-limit,
  idempotency, concurrency, operational completeness, substitution discipline,
  no-placeholders).
- Auto-remediate: nodemailer → 8.x (VERSION 0.1.1).
- Pass 2/3 re-scan: **0 vulnerabilities**; full smoke 22/22 (no regression).

## Deploy Target Reachability
- Outcome C — local-stack deploy (equivalent to `docker compose up`): Postgres,
  Redis, MailHog in Docker (ports 5433/6380/8026); API (:4000) + worker (host
  node processes); web served from the production build. Externals (real SMTP,
  Sentry DSN) unprovisioned → MailHog used; error tracking via structured logs.
- Verification: `GET /api/health` → 200 `{status:ok, services:{db,redis,queue:up}}`.

## Test Results
- Unit/integration (vitest): 25 passing.
- Functional smoke (`scripts/smoke-test.mjs`): 22/22 (auth bootstrap, anti-enum,
  W1–W7, tenant isolation, webhook reception + idempotency + bad-sig reject,
  dashboards, report export, large-header resilience). See `smoke-test.log`.
- UI workflow e2e (Playwright W1–W7): run at deploy against the served web origin.

## Cascade Impacts
_None._
