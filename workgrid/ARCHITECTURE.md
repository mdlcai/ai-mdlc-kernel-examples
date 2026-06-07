# ARCHITECTURE.md — WorkGrid

> Source of truth for architecture. Behavior lives in `SPEC.md`; precedence is
> `RESEARCH.md` > `ARCHITECTURE.md` > `SPEC.md`. Build depth: **comprehensive**
> (threat model + alternatives-considered included per major decision).

## 1. Overview

WorkGrid is a **multi-tenant SaaS work-management platform**: projects, tasks,
tickets/incidents (with SLA + auto-routing + escalation), configurable workflows
with approval gates, real-time updates, executive portfolio dashboards, audit
logging, and inbound/outbound integrations with GitHub and Slack plus email via
Resend.

**Core problem:** tool fragmentation → poor visibility, missed SLAs, manual
roll-ups. **Solution:** one workspace where work routes itself, dependencies stay
visible, and reporting is live.

**Tenancy:** every record belongs to an `organization` (the tenant). Users belong
to one or more organizations via `memberships` carrying a role. Isolation is
enforced at the database (RLS) and the service layer (ADR-0005).

## 2. System Architecture (layers, modules, data flow)

```
                         ┌────────────────────────────────────────┐
        HTTPS (nginx TLS)│  apps/web  — Next.js 16 (App Router)     │
  Browser ───────────────▶  RSC pages + Client components          │
        WSS               │  TanStack Query (server state)         │
                          │  Zustand (UI state) · shadcn/ui · TW    │
                          └───────────────┬────────────────────────┘
                                          │ REST /v1/*  +  Socket.IO
                          ┌───────────────▼────────────────────────┐
                          │  apps/api — NestJS 11                    │
                          │  ┌─────────── HTTP layer ─────────────┐ │
                          │  │ Guards: JwtAuth → Tenant → Rbac    │ │
                          │  │ Interceptors: TenantScope, Audit,  │ │
                          │  │   Logging, Transform(problem+json) │ │
                          │  │ Throttler (Redis store)            │ │
                          │  └────────────────────────────────────┘ │
                          │  Modules: auth, orgs, members, projects, │
                          │   tasks, tickets, workflows, dashboard,  │
                          │   notifications, reports, webhooks,      │
                          │   integrations, audit, health           │
                          │  Real-time: EventsGateway (socket.io)    │
                          │  Outbox: writes outbox_event in-tx       │
                          └───────┬──────────────┬─────────────┬─────┘
                                  │              │             │
                       TypeORM(pg)│         ioredis│       Socket.IO
                                  ▼              ▼  Redis adapter
                          ┌──────────────┐  ┌──────────────────────┐
                          │ PostgreSQL   │  │ Redis (BullMQ queues, │
                          │ RLS + GUC    │  │  throttler, ws pub/sub)│
                          └──────────────┘  └──────────┬────────────┘
                                                       │ consumes
                          ┌────────────────────────────▼───────────┐
                          │ apps/api worker entrypoint (BullMQ)      │
                          │  queues: outbox-relay, sla-escalation,   │
                          │   notifications, reports                 │
                          │  adapters: Resend, GitHub, Slack (no-op  │
                          │   when secrets are placeholders)         │
                          └──────────────────────────────────────────┘
```

**Request data flow (authenticated mutation):**
1. nginx terminates TLS, forwards to Next.js / API; HSTS + security headers (helmet).
2. `JwtAuthGuard` validates the access token → attaches `principal {userId}`.
3. `TenantGuard` resolves the active `organizationId` from the JWT/membership header and verifies membership; rejects with `E.NOT_FOUND` for non-member orgs (no existence leak).
4. `RbacGuard` checks the required permission for the route's role.
5. `TenantScopeInterceptor` opens a transaction-bound connection and `SET LOCAL app.current_tenant_id = <org>`; all repository work in the request runs inside it.
6. Service executes domain logic; any external side effect is appended to `outbox_event` **in the same transaction**.
7. `AuditInterceptor` writes an append-only `audit_log` row for state-mutating routes.
8. On commit, the worker relay later delivers outbox events; `EventsGateway` emits a real-time event to room `tenant:<org>`.
9. `TransformInterceptor` / `AllExceptionsFilter` shapes success and error (`application/problem+json`, RFC 9457) responses.

## 3. Module catalog

| Module | Responsibility |
|--------|----------------|
| `auth` | register, login, refresh, logout, me; Argon2id; anti-enumeration; rate-limited |
| `orgs` | organizations (tenants), create/switch, settings |
| `members` | memberships, roles, invitations |
| `projects` | projects, budget, health rollup |
| `tasks` | tasks, dependencies (DAG), assignment, status, board/list |
| `tickets` | incidents/tickets, severity, SLA, auto-routing, escalation, comments |
| `workflows` | workflow definitions (states + transitions + approval gates), runs |
| `dashboard` | portfolio aggregation (project health, budget burn, velocity, SLA) |
| `notifications` | in-app notification center + email fan-out (via outbox) |
| `reports` | export portfolio/tickets/tasks as PDF / CSV / JSON |
| `webhooks` | inbound GitHub/Slack receivers (HMAC verified before DB) |
| `integrations` | outbound GitHub/Slack/Resend adapters consumed by the outbox relay |
| `audit` | append-only audit log + query |
| `events` | Socket.IO gateway, tenant rooms, auth on handshake |
| `health` | `/v1/health` + `/api/health` liveness/readiness, dependency checks |

## 4. Data Layer (PostgreSQL + TypeORM)

- **ORM:** TypeORM (0.3.x line, ADR-0003). Entities under `apps/api/src/**/entities`, migrations under `apps/api/src/database/migrations`. `synchronize: false` always (migrations only).
- **Tenancy column:** every tenant-owned table has `organization_id uuid NOT NULL` (the `tenant_id` GUC maps to this).
- **RLS:** a migration runs `ALTER TABLE … ENABLE ROW LEVEL SECURITY; … FORCE ROW LEVEL SECURITY;` and creates policy `USING (organization_id = current_setting('app.current_tenant_id')::uuid)` for every tenant table. The app role is non-superuser so FORCE applies.
- **Optimistic concurrency:** mutable domain entities carry `@VersionColumn version`; updates require the expected version → conflict ⇒ `409 E.CONFLICT`.
- **Money:** budgets stored as integer minor units (cents) — never floats.
- **Custom fields:** `JSONB` columns where workflows/tickets need extensibility.
- **Key tables:** `organizations`, `users`, `memberships`, `invitations`,
  `projects`, `tasks`, `task_dependencies`, `tickets`, `ticket_comments`,
  `sla_policies`, `workflows`, `workflow_states`, `workflow_transitions`,
  `workflow_runs`, `approvals`, `notifications`, `audit_log`, `outbox_event`,
  `webhook_delivery`, `refresh_tokens`, `report_jobs`.
- **Uniqueness/idempotency:** `users(organization_id, email)` unique; `outbox_event(idempotency_key)` unique; `webhook_delivery(source, external_id)` unique (inbound dedup); `task_dependencies(task_id, depends_on_id)` unique.
- **Backups:** WAL archiving + daily `pg_dump` (documented; cron container in compose) → PITR. ARCHITECTURE-level backup strategy honored.

**Alternatives considered:** schema-per-tenant (rejected: 50k tenants × schema = migration/connection blow-up); database-per-tenant (rejected at this scale); Prisma (rejected: weaker raw RLS/GUC control than TypeORM `query`/transaction primitives).

## 5. Cross-cutting subsystems

**Real-time (has_websocket).** `EventsGateway` (socket.io) authenticates the
handshake via the access token, joins `tenant:<org>` (and `user:<id>`) rooms, and
re-checks RBAC on subscribe. Domain services emit typed events
(`ticket.updated`, `task.moved`, `notification.created`, …) scoped to the tenant
room. Horizontal fan-out via the Socket.IO **Redis adapter** so all pods deliver.
Never a global broadcast (INV-15 manual).

**Background jobs (Bull with Redis).** BullMQ (the maintained Bull successor —
logged) queues: `outbox-relay` (deliver pending outbox events), `sla-escalation`
(delayed jobs per ticket; recompute on state change), `notifications`,
`reports`. ioredis with `maxRetriesPerRequest: null`. The worker runs as a
separate process entrypoint (`apps/api/src/worker.ts`) but shares the codebase.

**Dual-write / outbox (has_dual_write).** ADR-0007. `OutboxService.enqueue()` is
only ever called inside a domain transaction. The relay consumer delivers with
the row's `idempotency_key`, retries with backoff, and moves exhausted events to
`status = 'dead'`. External adapters are idempotent and degrade to a logging
no-op when their secret env vars are placeholders (ADR-0008).

**Inbound webhooks (has_webhooks).** Raw-body middleware preserves the exact
bytes. Each receiver verifies the provider HMAC (`verifySignature`) **before**
any DB read, then dedups via `webhook_delivery(source, external_id)`, then
processes (`processEvent`). Order `verifySignature → checkIdempotency →
processEvent` (INV-4 boundary-order). Slack timestamp replay window ±5 min.

**SLA + escalation.** On ticket create/route, an `sla_policy` sets `due_at`; a
delayed BullMQ job fires at the breach time. State changes cancel/replace the
job. An idempotent reconciliation sweep recomputes breaches from `due_at` to
survive scheduler drift.

## 6. Integrations & notifications

- **Resend** (email): outbound only, via outbox relay; Bearer key from env; no-op adapter without `RESEND_API_KEY`.
- **GitHub**: inbound webhooks (`X-Hub-Signature-256`, HMAC-SHA256 over raw body); outbound notifications via signed delivery.
- **Slack**: inbound (`X-Slack-Signature` `v0=`, signing base `v0:ts:body`, ±5-min window, `url_verification` challenge); outbound via incoming webhook / `chat.postMessage`.
- **Alert channels:** email (Resend) + in-app notification center.

## 7. Observability, errors, config

- **Logging:** `nestjs-pino` structured JSON; request-id correlation; PII redaction (email/password/token fields). No `console.log` in app code (INV-12).
- **APM/monitoring:** OpenTelemetry-ready hooks; `/v1/health` (+ `/api/health` alias) returns dependency status (db, redis); a `/v1/metrics`-style readiness probe.
- **Error reporting:** Sentry adapter initialized from `SENTRY_DSN` (no-op when absent).
- **Config:** centralized `@nestjs/config` typed config module; `process.env` is read **only** inside `apps/api/src/config/**`, `main.ts`, `worker.ts` (INV-5). Every referenced var appears in `.env.example` (INV-3 + interface validation).
- **Errors:** RFC 9457 `application/problem+json`; typed `E.*` error codes; validation errors carry field-level `errors[]` so the UI can attach messages to controls.

## 8. Deployment & scaling

- **Containers:** multi-stage Dockerfiles for `web`, `api`, `worker`; `docker-compose.yml` brings up nginx (TLS), web, api, worker, postgres, redis, and a backup sidecar. Kubernetes manifests under `deploy/k8s` for the K8s constraint.
- **TLS / protocol:** HTTPS only (ADR-0009) — nginx terminates TLS, 301 HTTP→HTTPS, HSTS; local dev uses mkcert/self-signed.
- **Header buffers:** nginx `client_header_buffer_size 16k; large_client_header_buffers 8 32k;` and Node `--max-http-header-size=32768` (large-cookie resilience).
- **Scale (50k+):** API and worker are stateless and horizontally scalable; sticky sessions for WS or Redis-adapter fan-out; Postgres connection pooling; Redis for throttling/queues/ws.
- **CI/CD:** `.github/workflows/ci.yml` — install → typecheck → lint → test → build → invariant lint.

## 9. Architectural Invariants

Each rule has a machine-checkable (or `manual`) entry in `invariants.json`; the
`id` cross-references prose ⇄ JSON. The Stage 4 lint pass and the ship-time Drift
Detection Gate evaluate these.

| ID | Rule | File/section | Check |
|----|------|--------------|-------|
| INV-1 | Tenant-owned tables are accessed only through the service/repository layer — controllers must not inject repositories directly. | §2, §3; `apps/api/src/**` | forbidden-pattern `@InjectRepository(` in controllers |
| INV-2 | API versioning is global (`VersioningType.URI`, `/v1/`); controllers must not hardcode a `v1` path prefix. | §2; `enableVersioning` in `main.ts` | forbidden-pattern `@Controller('v1` |
| INV-3 | `.env.example` exists and enumerates every referenced env var; real `.env` is gitignored. | §7 | required-file `.env.example` |
| INV-4 | Inbound webhooks verify the HMAC signature before processing the event. | §5 Inbound webhooks; `modules/webhooks` | boundary-order `verifySignature` → `processEvent` |
| INV-5 | `process.env` is read only in the config layer / process entrypoints. | §7 Config | forbidden-pattern `process.env.` outside config/main/worker |
| INV-6 | Distributed rate limiting is configured (throttler module present). | §2, §5 | required-file `apps/api/src/common/security/throttler.config.ts` |
| INV-7 | Outbox events are uniquely keyed for idempotent delivery. | §5 outbox | required-unique-constraint `outbox_event(idempotency_key)` |
| INV-8 | Inbound webhook deliveries are deduplicated by `(source, external_id)`. | §5 webhooks | required-unique-constraint `webhook_delivery(source,external_id)` |
| INV-9 | UI uses design tokens — no raw hex colors in web components/pages outside the token layer. | DESIGN tokens; `apps/web` | forbidden-pattern `#[0-9a-fA-F]{6}` outside token files |
| INV-10 | Users are unique per tenant by email. | §4 | required-unique-constraint `users(organization_id,email)` |
| INV-11 | Audit log is append-only and written for state-mutating actions. | §2 step 7, §3 audit | manual |
| INV-12 | Application code emits structured logs, not `console.log`. | §7 logging | forbidden-pattern `console.log(` in `apps/api/src` |
| INV-13 | PII is never written to logs (redaction configured). | §7; pii_handling | manual |
| INV-14 | Every tenant table enables + forces RLS scoped to the tenant GUC. | §4 RLS | manual |
| INV-15 | Real-time events are emitted only to tenant-scoped rooms, never globally. | §5 real-time | manual |
| INV-16 | Every SPEC §UI Surface screen is routed, every non-internal endpoint is UI-reachable, and every SPEC §5 workflow has a UI e2e test. | SPEC §UI Surface, §5 | ui-coverage |
| INV-17 | A required design-token contract file exists (canonical tokens copied from template). | ADR-0002; `apps/web/src/app/globals.css` | required-file `apps/web/src/app/globals.css` |

## 10. Implementation notes & risks

- RLS + pooling: set the GUC with `SET LOCAL` inside the request transaction so it never leaks across pooled connections (research gap #1). Verified by a cross-tenant integration test.
- BullMQ wrapper version lag (research gap #3): pin `@nestjs/bullmq` + `bullmq` together; lockfile committed.
- WS scaling (research gap #2): start with the standard Redis adapter; document the streams-adapter upgrade.
- The no-op external adapters mean the local smoke test exercises the full outbox/workflow path without live Resend/GitHub/Slack; cloud upgrade documented in QUICKSTART.
