# ARCHITECTURE.md — Pulse

Infrastructure monitoring SaaS. Source-of-truth for architecture (precedence: RESEARCH > ARCHITECTURE > SPEC).
Build depth: **comprehensive**. Archetype: **saas**.

---

## 1. System Architecture

Pulse is a multi-tenant monorepo with four runtime surfaces over shared Postgres + Redis:

```
                         ┌───────────────────────────────────────────────┐
   Browser (React SPA) ──┤ HTTPS / WSS (HSTS, SameSite cookie)            │
                         └───────────────┬───────────────────────────────┘
                                         │
                 ┌───────────────────────▼─────────────────────────┐
                 │  apps/api  — Express 5 REST (/v1) + Socket.IO    │
                 │  • auth (sessions)   • monitors/incidents CRUD   │
                 │  • channels/rules    • reports (JSON/CSV)        │
                 │  • inbound webhooks  • /v1/ingest (API-key)      │
                 │  • rate-limit, audit, error-contract middleware  │
                 └───────┬───────────────────────┬─────────────────┘
                         │ enqueue               │ pub/sub (Redis adapter)
                         ▼                       ▼
              ┌────────────────────┐    ┌──────────────────────┐
              │ Redis / Valkey     │◄───┤ Socket.IO rooms       │
              │ BullMQ queues:     │    │  org:<id>             │
              │  • schedule        │    └──────────────────────┘
              │  • checks          │
              │  • notifications   │
              └─────────┬──────────┘
                        │ consume
              ┌─────────▼───────────────────────────────────────┐
              │ apps/worker — BullMQ workers                     │
              │  • scheduler  (repeatable→enqueue checks+jitter) │
              │  • checker    (http/tcp/icmp/dns/push probes)    │
              │  • evaluator  (alert state machine, incidents)   │
              │  • notifier   (fan-out: email/sms/slack/teams/wh)│
              └─────────┬───────────────────────────────────────┘
                        │
              ┌─────────▼──────────┐
              │ PostgreSQL 16      │  org-scoped tables (+ RLS), partition-ready metrics
              │ (Drizzle ORM)      │
              └────────────────────┘
```

**Layers**
1. **Presentation** — `apps/web` (React 19 + Vite + Tailwind + TanStack Query + React Router + Recharts). Marketing landing (ported from DESIGN-TEMPLATE.html) + authenticated app shell.
2. **API / Edge** — `apps/api`. Express 5 mounted at `/v1`; Socket.IO on the same HTTP server. Middleware chain: requestId → structured logger → HTTPS/HSTS → rate-limit → cookie/session auth → CSRF/Origin guard → Zod validation → handler → error-contract (RFC 9457).
3. **Domain / workers** — `apps/worker`. Stateless BullMQ consumers. Scheduler enqueues per-monitor checks via repeatable jobs with jitter; checker runs probes; evaluator advances the alert state machine and opens/updates incidents; notifier fans out with retries/DLQ + idempotency.
4. **Data** — `packages/db` (Drizzle schema, migrations, `scopedDb` wrapper, `unsafeDb` raw client confined to this package). Postgres for relational + time-series (partition-ready); Redis for queue, rate-limit buckets, and Socket.IO adapter.
5. **Shared contract** — `packages/shared` (Zod schemas reused API↔web, typed error codes, the pure alert state-machine function, channel-payload builders).

**Data flow (a check cycle).** Scheduler tick → enqueue `checks:run{monitorId}` (jittered) → checker probes target, writes `check_results` row + latest `monitors.status`, emits `monitor.update` to `org:<id>` room → evaluator consumes result, runs flap-detecting state machine; on confirmed transition opens/updates an `incident`, writes `incident_events`, and enqueues one `notifications` job **per matching alert rule × channel** (deterministic idempotency key) → notifier delivers, retries with backoff, dead-letters on exhaustion, audit-logs every attempt.

## 2. Data Model

All tenant tables carry an indexed `org_id` and are reached only through `scopedDb(orgId)`. Postgres **Row-Level Security** is enabled as defense-in-depth (policy `org_id = current_setting('app.org_id')`).

| Table | Key columns | Notes |
|-------|-------------|-------|
| `organizations` | id, name, slug(unique), created_at | tenant root |
| `users` | id, **email(unique)**, org_id, password_hash, name, role(owner/admin/operator/viewer), failed_login_count, locked_until, last_login_at | email globally unique (ADR-0005) |
| `sessions` | id, user_id, org_id, token_hash(unique), expires_at, ip, user_agent | opaque cookie token, hashed |
| `api_keys` | id, org_id, name, key_hash, prefix, last_used_at | for `/v1/ingest` |
| `monitors` | id, org_id, name, type, target, interval_seconds, timeout_ms, config(jsonb), enabled, status, last_checked_at, region, created_by | status ∈ up/degraded/down/paused/pending |
| `check_results` | id, org_id, monitor_id, checked_at, status, latency_ms, status_code, error, region | time-series, partition-ready on checked_at |
| `metrics` | id, org_id, monitor_id, ts, cpu, mem, disk, custom(jsonb) | OS metrics via ingest |
| `incidents` | id, org_id, monitor_id, status(open/acknowledged/resolved), severity, title, root_cause, opened_at, acknowledged_at, acknowledged_by, resolved_at | one open incident per monitor |
| `incident_events` | id, org_id, incident_id, ts, kind, message, actor | timeline |
| `notification_channels` | id, org_id, type(email/sms/slack/teams/webhook), name, config(jsonb, **AES-GCM secrets**), enabled | ADR-0009 |
| `alert_rules` | id, org_id, monitor_id(nullable=all), channel_id, on_states(text[]), cooldown_seconds, enabled | routes incidents→channels |
| `notifications` | id, org_id, incident_id, channel_id, status(pending/sent/failed/dead), attempts, **idempotency_key(unique)**, last_error, created_at, sent_at | guaranteed delivery (ADR-0008) |
| `audit_logs` | id, org_id, ts, actor_user_id, action, resource_type, resource_id, ip, metadata(jsonb) | SOC2 |

Migrations are Drizzle SQL files in `packages/db/drizzle/` (auditable artifacts). UNIQUE constraints on `users.email` (INV-3) and `notifications.idempotency_key` (INV-2) are migration-defined.

## 3. Services & Orchestration

- **API service** (`apps/api`): REST resource modules (auth, monitors, incidents, channels, rules, reports, ingest, webhooks), Socket.IO gateway (handshake authenticated by session cookie; `socket.join('org:'+orgId)`; emits only to that room), Redis adapter for multi-instance scale.
- **Queues** (BullMQ): `schedule` (repeatable per-monitor jobs), `checks` (one job per due check), `notifications` (one job per channel delivery). Concurrency-bounded workers; exponential backoff; dead-letter via `status='dead'` + `failed` queue retention.
- **Worker processors**: `scheduler.ts` (reconciles monitors ↔ repeatable jobs, adds jitter), `checker.ts` (protocol probes), `evaluator.ts` (alert state machine + incident lifecycle), `notifier.ts` (channel adapters).
- **Channel adapters**: `email` (nodemailer), `sms` (Twilio), `slack` (incoming webhook JSON/Block Kit), `teams` (Power Automate Workflows MessageCard), `webhook` (HMAC-SHA256 signed, SSRF-guarded). One interface `deliver(channel, incident): Promise<DeliveryResult>`.

## 4. Design Patterns
- **Scoped repository**: `scopedDb(orgId)` injects `org_id` on every read/write; raw `unsafeDb` confined to `packages/db` (INV-1).
- **State machine** (pure, in `packages/shared`): `nextStatus(prev, window, thresholds)` — flap detection via N-consecutive / failure-ratio hysteresis.
- **Outbox/idempotency**: notification rows keyed by deterministic idempotency key; UNIQUE constraint closes the TOCTOU window at the DB layer.
- **Adapter**: uniform notification channel interface.
- **Circuit breaker**: per-channel breaker isolates a failing provider.
- **Problem+JSON error contract** (RFC 9457): every error response is `{type,title,status,detail,code,errors?}`.

## 5. Dependencies (pinned at install; lockfile committed)
Backend: `express@^5`, `drizzle-orm`, `drizzle-kit`, `postgres` (pg driver), `bullmq`, `ioredis`, `socket.io`, `@socket.io/redis-adapter`, `argon2` (or `@node-rs/argon2` fallback), `zod`, `pino`, `pino-http`, `cookie`, `twilio`, `nodemailer`, `helmet`, `cors`.
Frontend: `react@^19`, `react-dom`, `vite`, `@vitejs/plugin-react`, `tailwindcss`, `@tanstack/react-query`, `react-router`, `recharts`, `socket.io-client`.
Tooling: `typescript`, `tsx`, `vitest`, `@playwright/test`, `eslint`, `prettier`. Container: Docker + Compose V2. CI: GitHub Actions.

## 6. Cross-cutting Concerns
- **Security/HTTPS**: Helmet; HTTPS-only with HSTS in production; HTTP→HTTPS redirect at the reverse proxy (nginx). `Secure;HttpOnly;SameSite=Lax` session cookie; Origin/CSRF guard on state-mutating routes. Argon2id hashing. Secrets via env vars + `.env.example`; channel secrets AES-256-GCM at rest; Pino redaction of `authorization`, `cookie`, `*.token`, `*.secret`, `password`.
- **Rate limiting**: Redis token-bucket keyed by `(org, ip)` with auth/read/write tiers; `429 + Retry-After`. Auth limiter fires before password compare (INV-5).
- **Audit logging**: middleware records every mutation to `audit_logs` (actor, action, resource, ip).
- **Observability**: Pino structured JSON logs with requestId correlation; `/api/health` (db + redis + queue checks); counters for checks run, notifications sent/failed, queue depth.
- **Header-buffer sizing**: nginx `client_header_buffer_size 16k; large_client_header_buffers 8 32k;`; Node `--max-http-header-size=32768`.
- **CI**: `.github/workflows/ci.yml` runs install → lint → typecheck → unit+integration tests → build → dependency/secret scan → invariant-lint.

## 7. Implementation Notes & Risks
- Redis is the critical-path dependency (queue + rate-limit + WS adapter) → HA in production; `/api/health` reports it.
- ICMP often needs raw sockets/root; checker uses a TCP-connect fallback when ICMP is unavailable in-container, logged per check (degraded probe, not failure).
- Metrics volume at 50k devices: table partition-ready on `checked_at`/`ts`; rollup job downsamples; TimescaleDB optional, not required to run.
- Email/SMS/Slack/Teams require real provider credentials → unprovisioned in local stack; notifier degrades to recorded-but-pending with a clear contract (Contract Smoke Test path).

## 8. Trust Boundaries (summary; full STRIDE in RESEARCH §5)
B1 anon→API, B2 tenant→other tenants, B3 inbound webhooks→handler, B4 outbound targets, B5 worker→DB. Controls: §9 invariants + §6 concerns. OWASP A01 (scopedDb+RLS), A02 (env secrets+redaction), A03 (Zod+parameterized), A05 (rate-limit), A07 (Argon2id+lockout), A08 (webhook signature verify), A10 (SSRF egress guard).

## 9. Architectural Invariants

Every rule below has a corresponding `invariants.json` entry (id cross-reference). Machine-checkable rules are enforced by `scripts/invariant-lint.mjs` in the Verification Gate and the Drift Detection Gate; `manual` rules are prose-audited.

| ID | Rule | File / section | Check type |
|----|------|----------------|-----------|
| INV-1 | All tenant-scoped DB access goes through `scopedDb(orgId)`; the raw `unsafeDb` client is confined to `packages/db`. | packages/db/src/scoped.ts | forbidden-pattern |
| INV-2 | `notifications.idempotency_key` has a UNIQUE constraint (guaranteed-delivery, no double-send). | packages/db/drizzle/* | required-unique-constraint |
| INV-3 | `users.email` has a UNIQUE constraint (anti-duplicate-account, single-lookup login). | packages/db/drizzle/* | required-unique-constraint |
| INV-4 | Inbound webhook handlers verify the signature before any DB write (`verifyInboundSignature` precedes `recordInbound`). | apps/api/src/modules/webhooks/** | boundary-order |
| INV-5 | Auth login applies the rate limiter before the password compare (`rateLimit` precedes `verifyPassword`). | apps/api/src/modules/auth/** | boundary-order |
| INV-6 | `invariants.json` exists at project root. | invariants.json | required-file |
| INV-7 | `.env.example` exists (every env var documented). | .env.example | required-file |
| INV-8 | `docker-compose.yml` exists (full local stack). | docker-compose.yml | required-file |
| INV-9 | `.github/workflows/ci.yml` exists (ci_cd_required). | .github/workflows/ci.yml | required-file |
| INV-10 | Backend services use the structured logger, not `console.log`. | apps/api, apps/worker, packages | forbidden-pattern |
| INV-11 | Every SPEC §UI Surface screen is routable and every §5 workflow has a UI e2e test to its terminal step. | apps/web/src/screens/**, SPEC.md | ui-coverage |
| INV-12 | Colors/type/spacing/radius/shadow come from the design-token layer (template `:root`), not ad-hoc values. | apps/web/src/styles/tokens.css | manual |
| INV-13 | Production enforces HTTPS-only + HSTS with an HTTP→HTTPS redirect. | deploy/nginx.conf, apps/api security middleware | manual |
| INV-14 | Notification-channel secrets are encrypted at rest (AES-256-GCM) and never logged. | packages/shared/src/crypto.ts, apps/api channels module | manual |
| INV-15 | Outbound custom-webhook delivery passes an SSRF egress guard (`assertPublicHttpsUrl`) before fetch. | apps/worker/src/channels/webhook.ts | manual |

## Alternatives Considered (comprehensive)
- **ORM**: Drizzle (chosen — SQL-transparent migrations, no engine binary, easy scoped wrapper) vs Prisma (heavier, engine binary) vs Kysely (lower-level). 
- **Queue**: BullMQ/Redis (chosen — repeatable jobs + retries/DLQ in one) vs pg-boss (no extra infra, but weaker at 50k throughput) vs RabbitMQ (more ops).
- **Realtime**: Socket.IO (chosen — rooms, reconnection, Redis adapter) vs raw `ws` (more glue) vs SSE (one-way only).
- **Auth**: server-side sessions (chosen — revocable, XSS-safe) vs JWT (stateless but hard to revoke).
- **Metrics store**: partition-ready Postgres (chosen — satisfies constraint, runs anywhere) vs TimescaleDB (optional add-on) vs ClickHouse (separate store, more ops).
- **Hashing**: Argon2id (chosen, OWASP-preferred) vs bcrypt (fallback if native build fails).
