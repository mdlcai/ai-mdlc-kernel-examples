# ARCHITECTURE.md — ServiceHub

> Source of truth for **architecture**. Behavior is governed by `SPEC.md`.
> Precedence: `RESEARCH.md` > `ARCHITECTURE.md` > `SPEC.md`.
> Build: comprehensive · review_gates: auto · archetype: saas.

## 1. System Overview

ServiceHub is a centralized, multi-tenant IT Service Management (ITSM) platform:
a React single-page app talking to a versioned REST API (`/v1/`) backed by
PostgreSQL, with realtime updates over WebSocket and asynchronous work
(ticket routing, SLA escalation, notifications, webhook/email delivery) handled
by a Redis-backed job queue. Everything runs in Docker.

```
                         ┌────────────────────────────────────────────┐
                         │                 Browser (SPA)               │
                         │   React + Vite + Tailwind + TanStack Query  │
                         │   socket.io-client (realtime)               │
                         └───────────────┬─────────────▲──────────────┘
                                HTTPS REST │   WebSocket │ events
                         ┌───────────────▼─────────────┴──────────────┐
                         │             API service (Express 5)         │
                         │  auth · RBAC · org-scope · Zod validation   │
                         │  controllers → services → repositories      │
                         │  Socket.IO server · Helmet · rate-limit     │
                         └───┬──────────────┬───────────────┬──────────┘
                             │              │               │ enqueue
                ┌────────────▼───┐   ┌──────▼──────┐   ┌────▼─────────────┐
                │  PostgreSQL    │   │   Redis     │   │  BullMQ queues    │
                │  (TypeORM,RLS) │   │ (queue/RL/  │   │  routing · sla    │
                │  append-only   │   │  socket adp)│   │  notify · outbox  │
                │  audit_logs    │   └─────────────┘   │  webhook-delivery │
                └────────────────┘                     └────┬──────────────┘
                                                            │ consume
                                              ┌─────────────▼──────────────┐
                                              │     Worker service          │
                                              │  (same image, worker entry) │
                                              │  routing/SLA/notify/outbox  │
                                              └───────┬─────────┬───────────┘
                                                      │ SMTP    │ HTTPS
                                              ┌───────▼───┐ ┌───▼──────────┐
                                              │  MailHog  │ │ outbound     │
                                              │ (dev SMTP)│ │ webhooks     │
                                              └───────────┘ └──────────────┘
```

## 2. Data Flow

1. **Request submission** — User POSTs a ticket to `/v1/tickets`. The API
   validates (Zod), persists within the caller's `organization_id`, writes an
   audit record and an `outbox_event` in the **same transaction**, assigns a
   per-org ticket number, computes SLA deadlines, and enqueues `routing` +
   `sla` jobs. A `ticket.created` event is emitted to the org's Socket.IO room.
2. **Routing** — The `routing` worker auto-assigns to the best-fit technician
   (skill match + lowest current workload), writes assignment + audit, emits
   `ticket.assigned`, enqueues a `notify` job.
3. **SLA** — On create and on every status change, the `sla` worker (delayed
   jobs) re-computes response/resolution deadlines against the org's
   business-hours calendar, schedules breach-warning (80%) and breach jobs, and
   escalates on breach (emit `ticket.escalated`, notify manager).
4. **Notifications** — `notify` worker fans out in-app notifications (persisted +
   realtime) and email (via the outbox relay → SMTP), with retries + DLQ.
5. **Outbox relay** — Any domain change that must reach the outside world
   (email, outbound webhook) is written as an `outbox_event` in the same DB
   transaction; the `outbox` relay polls/consumes and delivers at-least-once
   with idempotency keys.
6. **Inbound webhooks** — Monitoring systems POST signed alerts to
   `/v1/webhooks/inbound/:source`. The handler verifies the HMAC signature on
   the **raw body first**, dedupes by `(source, external_id)`, then enqueues an
   ingest job that creates/updates a ticket.
7. **Realtime** — The SPA subscribes to per-org and per-ticket rooms; dashboards
   and queues update live without polling.

## 3. Component / Service Orchestration

| Service | Responsibility | Entry |
|---|---|---|
| **api** | REST API, auth, Socket.IO server, validation, authorization | `apps/api/src/index.ts` |
| **worker** | BullMQ consumers: routing, sla, notify, outbox, webhook-delivery | `apps/api/src/worker.ts` |
| **web** | React SPA (served by nginx in prod, Vite in dev) | `apps/web` |
| **postgres** | Primary datastore (TypeORM migrations, RLS) | container |
| **redis** | Queue backend, rate-limit store, Socket.IO Redis adapter | container |
| **mailhog** | Dev SMTP capture (prod: real SMTP via env) | container |

The API and worker share one codebase/image; the worker is a separate process
(separate container) so queue load never blocks request latency.

### Layering (api)
`routes → controllers → services → repositories → entities`
- **controllers**: HTTP concerns, Zod parse, map domain → DTO, map errors → RFC 9457.
- **services**: business logic, transactions, enqueue jobs, emit events.
- **repositories**: the ONLY layer that touches `AppDataSource.getRepository(...)`;
  all tenant-scoped reads/writes go through `withOrgScope(...)`.
- **entities**: TypeORM models.

## 4. Data Layer

- **PostgreSQL** via **TypeORM** (`synchronize: false` in every environment;
  schema changes only through reviewed migrations in `apps/api/src/db/migrations`).
- **Multi-tenancy**: every tenant-owned table carries `organization_id`.
  Application enforcement via `withOrgScope(orgId, fn)`; **defense-in-depth via
  Postgres Row-Level Security** — the request opens its transaction with
  `SET LOCAL app.current_org = '<org>'` and RLS policies restrict rows to that org.
- **Append-only audit log** (`audit_logs`): insert-only. The runtime DB role is
  granted only `INSERT`/`SELECT` on this table (no `UPDATE`/`DELETE`).
- **Outbox** (`outbox_events`): domain change + outbox row committed atomically;
  relay delivers and marks delivered/failed.
- **Idempotency**: inbound webhooks dedupe via a UNIQUE constraint on
  `inbound_webhook_events (source, external_id)`. Ticket numbering is unique per
  org via UNIQUE `tickets (organization_id, number)`.

### Core entities
`organizations`, `users` (Argon2id hash, optional TOTP secret), `teams`,
`team_members`, `roles`/`permissions` (RBAC), `categories`, `tickets`,
`ticket_comments`, `attachments`, `sla_policies`, `business_hours_calendars`,
`ticket_sla_states`, `escalations`, `approvals` (security access-request chains),
`notifications`, `notification_preferences`, `audit_logs`, `outbox_events`,
`inbound_webhook_events`, `webhook_endpoints` (outbound), `webhook_deliveries`,
`saved_reports`. Full DDL detail lives in `SPEC.md` §Data Model.

## 5. Key Design Decisions (alternatives in DECISIONS.md)

| Decision | Choice | Why |
|---|---|---|
| API framework | Express 5 | Stable, ubiquitous middleware (Helmet, rate-limit, Multer) |
| ORM | TypeORM | Constraint-mandated; migrations + decorators |
| Realtime | Socket.IO + Redis adapter | Rooms/reconnection; horizontal scale |
| Queue | BullMQ + Redis | Delayed jobs (SLA), retries/backoff, DLQ |
| Auth | JWT access (15m) + rotating refresh (httpOnly cookie) + Argon2id | OWASP/NIST aligned |
| Errors | RFC 9457 `application/problem+json`, typed codes | Consistent, field-level detail |
| Delivery | Transactional outbox + queue relay | Guaranteed (at-least-once) email/webhook |
| Validation | Zod at the boundary | TS-first runtime validation |
| Tenancy | org-scope wrapper + Postgres RLS | App + DB defense-in-depth |
| Charts | Recharts | Mature React charting for dashboards |

## 6. Cross-Cutting Concerns

- **Security**: Helmet (CSP, HSTS), HTTPS-only (HTTP→HTTPS redirect + HSTS at the
  reverse proxy), distributed rate limiting (`rate-limit-redis`), CSRF/Origin
  guard on cookie-authenticated state-mutating routes, Argon2id, anti-enumeration
  on auth, SSRF guard on outbound webhooks, EXIF-strip + re-encode on uploads.
- **Observability**: `pino` structured JSON logs with request-id, OpenTelemetry
  traces/metrics, Sentry error capture, `/api/health` (+ dependency checks) and
  `/api/metrics`.
- **Config**: environment-based secrets only; every variable documented in
  `.env.example`; fail-fast validation of required env at boot.
- **Backups**: daily `pg_dump` + WAL archiving for PITR (documented; cron in
  compose/ops).

## 7. Deployment

- `docker-compose.yml` brings up postgres, redis, mailhog, api, worker, web.
- CI (`.github/workflows/ci.yml`): install → typecheck → lint → test → build →
  invariant lint.
- Local-stack deploy is the default reachable target (Docker). Cloud deploy uses
  the same images behind a TLS-terminating reverse proxy.

## 8. Implementation Notes & Risks

- SLA accuracy depends on persisted absolute deadlines + idempotent escalation
  workers, never in-memory timers (see RESEARCH §5).
- Migration drift guarded by `synchronize:false` + CI schema check.
- Realtime scale uses the Redis adapter + sticky sessions at the proxy.

## 9. Architectural Invariants

Each rule below has a machine-checkable (or `manual`) entry in `invariants.json`
with the matching `id`. The Stage 4 invariant-lint pass and the Drift Detection
Gate consume that file. **Never disable an invariant to pass a gate** — amend
ARCHITECTURE §9 and `invariants.json` in the same commit with an ADR.

| ID | Rule | invariants.json check |
|---|---|---|
| **INV-1** | Tenant-scoped DB access goes only through the repositories/db layer (`AppDataSource.getRepository(` must not appear in controllers/services). | `forbidden-pattern` `AppDataSource\.getRepository\(` outside `apps/api/src/repositories/**`, `apps/api/src/db/**` |
| **INV-2** | TypeORM auto-schema is never enabled (`synchronize: true` forbidden everywhere). | `forbidden-pattern` `synchronize:\s*true` outside (none) |
| **INV-3** | Ticket numbering is unique per organization. | `required-unique-constraint` table `tickets` columns `[organization_id, number]` |
| **INV-4** | Inbound webhook events are idempotent (dedupe by source + external id). | `required-unique-constraint` table `inbound_webhook_events` columns `[source, external_id]` |
| **INV-5** | Inbound webhook signature is verified before any DB/queue work. | `boundary-order` first `verifyHmacSignature` second `enqueueInboundAlert` in `apps/api/src/modules/webhooks/inbound.controller.ts` |
| **INV-6** | A distributed rate-limiter middleware exists. | `required-file` `apps/api/src/middleware/rateLimit.ts` |
| **INV-7** | The environment contract is documented. | `required-file` `.env.example` |
| **INV-8** | The invariant-lint runner exists as an executable artifact. | `required-file` `scripts/invariant-lint.mjs` |
| **INV-9** | Web UI colors come from the token system, not hard-coded hex in components. | `forbidden-pattern` `#[0-9a-fA-F]{6}` outside `apps/web/src/theme/**`, `apps/web/src/index.css`, `apps/web/tailwind.config.ts`, `**/*.test.*`, `**/*.spec.*` |
| **INV-10** | Every SPEC §UI Surface screen is routable and every Key Workflow has a UI e2e test. | `ui-coverage` screens `apps/web/src/pages/**` spec `SPEC.md` |
| **INV-11** | Tenant isolation is enforced at the DB via Row-Level Security keyed on `app.current_org`. | `manual` — verify RLS policies + `SET LOCAL app.current_org` in the tenancy middleware/migrations |
| **INV-12** | The audit log is append-only; the runtime DB role has no UPDATE/DELETE on `audit_logs`. | `manual` — verify grants in the init/migration SQL and absence of update/delete on the audit repository |
| **INV-13** | Password hashing is Argon2id and is never silently swapped for a weaker/polyfill primitive. | `manual` — verify `argon2` usage in `apps/api/src/modules/auth` and no `bcryptjs`/browser polyfill |
| **INV-14** | Auth rate-limiting fires before the password hash comparison. | `manual` — verify `authRateLimiter` precedes `verifyPassword` on the login route |

Invariant count: **14** (9 machine-checkable, 1 ui-coverage, 4 manual — see `invariants.json`).
