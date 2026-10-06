# ARCHITECTURE.md — ThreatWatch

> Stage 1 blueprint · build_depth **standard** · scale tier **small** (+ `has_websocket` connection-scaling bump) · kernel `feat/govern`
> Inputs: RESEARCH.md (Product Vision, 12 Key Workflows, Success Metrics, Build Constraints, Design Language, §3–§7), DESIGN-TEMPLATE.html (ADR-002/003), DECISIONS.md ADR-001…010.

## 1. Problem → Solution Summary

Enterprise SOC teams get telemetry from many disconnected tools, so the analyst ends up being the correlation engine. ThreatWatch is a **multi-tenant security intelligence layer** that runs this pipeline:

**INGEST → NORMALIZE → ENRICH → CORRELATE → DETECT → ALERT → INVESTIGATE → RESPOND**

Analysts see the result on a live operating picture: dashboard, threat map, alert feed, source and asset profiles, and search. The core analyst path is **Dashboard → Alert → Source → Target → Related Events → Threat Intel → Timeline → Resolution**, and it never leaves ThreatWatch.

**Personas → roles:**

| Persona | Role | Can do |
|---|---|---|
| Security Manager / org creator | `owner` | Everything, including org settings, team and the job trigger |
| Security Engineer | `admin` | Integrations, rules, notification policies, reports, team (except owners) |
| SOC Analyst / Incident Responder | `analyst` | Triage alerts, investigate, search, edit asset metadata |
| Executive / Compliance / IT | `viewer` | Read-only dashboards, map, alerts, reports |

**Threat model (summary):**

| Asset | Threat | Control |
|---|---|---|
| Tenant data | Cross-tenant read/write (IDOR) | `scoped(orgId)` services, 404 on foreign ids, isolation tests, org-keyed WS channels |
| Ingest endpoint | Forged events, replay, flood, oversized payloads | Standard-Webhooks HMAC verified first, 300 s window, UNIQUE message id, 256 KB cap, per-integration rate limit |
| Accounts | Credential stuffing, session theft, CSRF | argon2id, login rate limit, `__Host-` HttpOnly SameSite=Lax cookie, Origin check on mutations, session rotation |
| Outbound deliveries | SSRF via customer URLs | `assertSafeUrl` (https, DNS-resolved private/link-local/metadata ranges blocked, no redirects, 5 s timeout) before every dispatch; the request connects to the validated IP (pinned lookup, ADR-028) |
| WebSocket | Cross-site WebSocket hijacking, message floods | Origin allow-list + session check in the `upgrade` handler, server-push only, 20 msg/10 s inbound budget |
| Secrets | Leakage in DB or logs | AES-256-GCM at rest, pino redaction, shown once at creation |
| UI | XSS | Nonce CSP (Profile N), React escaping, no `dangerouslySetInnerHTML` beyond the nonce'd theme bootstrap |

## 2. System Architecture

```
                        ┌────────────────────────── Docker Compose (single host) ─────────────────────────┐
 Browser ──HTTPS:9443──▶│ caddy (tls internal, :80→:443 redirect)                                          │
 Integrations ─HTTPS──▶ │   ├─ /api/*, /ws ──▶ api (Express 5 + ws, :4000)  ──┐                            │
                        │   └─ /*          ──▶ web (Next.js 16, :3000)        │  Temporal client            │
                        │                                                     ▼                            │
                        │  postgres:17 ◀── Prisma 7 (adapter-pg) ── api   temporal (CLI dev server :7233)  │
                        │      ▲   ▲  LISTEN tw_events ──▶ api WS hub        ▲                              │
                        │      │   └── NOTIFY ◀── worker (Temporal worker: workflows + activities) ─┘       │
                        │      └────────────────────────── worker ──SMTP──▶ mailpit (local) / SMTP          │
                        └──────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Processes / containers

| Service | Image / build | Role | Health |
|---|---|---|---|
| `caddy` | `caddy:2.10-alpine` | TLS termination (`tls internal`), :80 → :443, reverse proxy, WebSocket passthrough | `caddy validate` + TCP |
| `web` | `web/Dockerfile` (multi-stage, non-root, tini) | Next.js 16 standalone server; `proxy.ts` sets the per-request CSP nonce and the auth redirect; the `(app)` layout reads the session from `API_INTERNAL_URL` (cookie + Caddy's X-Forwarded-For only) so the shell is server-rendered (ADR-031) | `GET /healthz` (web liveness) |
| `api` | `server/Dockerfile` target `api` | Express 5 REST `/api/v1`, `/api/health`, `/api/openapi.json`, `ws` hub at `/ws`, Temporal client, LISTEN `tw_events` | `GET /api/health` |
| `worker` | `server/Dockerfile` target `worker` | Temporal worker (task queue `threatwatch`), registers schedules at boot | `/healthz` on the internal port 4001 (worker running + DB) |
| `temporal` | `temporalio/temporal` (pinned tag) `server start-dev --ip 0.0.0.0 --db-filename /data/temporal.db` | Workflow engine; UI on :8233 bound to 127.0.0.1 | `temporal operator cluster health` |
| `postgres` | `postgres:17-alpine` | System of record; LISTEN/NOTIFY fan-out | `pg_isready` |
| `mailpit` | `axllent/mailpit` | Local SMTP sink; UI on 127.0.0.1:8026 | HTTP |
| `migrate` | `server` image, one-shot | `prisma migrate deploy` + seed; `api` and `worker` depend on it with `service_completed_successfully` | exit 0 |

Published host ports (probed 2026-10-04; 80, 443, 3000, 3100, 4000, 5432, 8025, 8080 and 8443 are taken): `HTTPS_PORT=9443`, `HTTP_PORT=8088`, `MAIL_UI_PORT=8026`, `TEMPORAL_UI_PORT=8233`. Postgres and the api are **not** published. All are env-overridable.

### 2.2 Code layout (npm workspaces)

```
package.json                 # workspaces: server, web; root scripts (lint:invariants, web:baseline, test, build)
server/
  prisma/schema.prisma       # single schema; migrations/ (SQL, forward + down.sql per change)
  prisma.config.ts
  src/config/env.ts          # zod env schema — fail fast naming the variable
  src/lib/                   # db.ts (Prisma + adapter-pg), logger.ts (pino), problem.ts (RFC 9457), crypto.ts (AES-GCM, tokens),
                             # signing.ts (Standard Webhooks HMAC), ssrf.ts (assertSafeUrl), clock.ts, pagination.ts, notify.ts (pg_notify)
  src/domain/                # PURE logic, no I/O: normalize.ts, enrich.ts, rules.ts (DSL eval), correlate.ts, risk.ts, states.ts
  src/services/              # ONLY place that touches Prisma; every function takes a TenantScope {orgId, userId, role}
    auth/ orgs/ integrations/ ingest/ rules/ alerts/ assets/ sources/ search/ dashboard/ notify/ email/ deliveries/ reports/ jobs/ audit/
  src/api/                   # app.ts (express factory), main.ts (listen + shutdown), middleware/, routes/*.ts, ws/hub.ts, openapi.ts
  src/worker/                # main.ts (Worker.create + shutdown), workflows.ts (deterministic), activities.ts (thin → services), schedules.ts
  test/                      # vitest: unit (domain), integration (services + routes against a real Postgres), jobs (effect tests)
web/
  src/proxy.ts               # CSP nonce + unauthenticated redirect (presence of the session cookie only; authz is in the api)
  src/app/                   # (marketing)/page.tsx, (auth)/login|register, (app)/dashboard|map|alerts|rules|integrations|sources|assets|events|search|notifications|reports|settings/*
  src/components/            # design-system primitives (Button, Field, Table, Badge, Card, EmptyState, Skeleton, Toast, Dialog, Tabs…)
  src/lib/api.ts             # fetch wrapper: same-origin, credentials, Problem parsing, Idempotency-Key helper
  src/state/                 # React Context: SessionContext, FiltersContext, LiveContext (WebSocket)
  src/styles/tokens.css      # the ADR-002/003 token set — the only file with raw colour literals (plus globals.css)
  e2e/                       # Playwright: W1…W12 workflow specs, desktop + mobile projects, axe
Caddyfile · compose.yaml · .env.example · .github/workflows/ci.yml · scripts/ (pinned runners, smoke-test.sh)
```

### 2.3 Data flow: the core pipeline

1. **Ingest (`POST /api/v1/ingest/:integrationId`)**:
   - Raw body is capped at 256 KB.
   - The only pre-verification read loads the integration's encrypted secret by primary key.
   - `verifyIngestSignature` checks Standard Webhooks HMAC-SHA256 over `id.timestamp.body` within ±300 s, using a constant-time compare.
   - A bad or missing signature, an unknown integration and a disabled integration all return the same 401 problem.
   - Only after verification does it parse with zod (≤500 events per batch).
2. **In one transaction**, ingest:
   - inserts `webhook_receipts(integration_id, message_id)`; a UNIQUE conflict means a replay, which returns `200 {duplicate:true}` with no other effect
   - normalizes each event (`domain/normalize.ts` → OCSF-inspired `NormalizedEvent`)
   - enriches each event (geo from `geo_ip_ranges`, intel from `indicators`, asset match by IP/hostname, auto-creating the asset)
   - inserts `events`, deduped by UNIQUE `(integration_id, source_event_id)`
   - writes malformed events to `ingest_errors`
   - creates an `ingest_batches` row with `detection_status='pending'` (the detection outbox, `has_dual_write`)
   - updates the integration's `last_event_at` and counters
   - sends `pg_notify('tw_events', {orgId, kind:'events', count})`
   - Response: `202 {batchId, accepted, duplicates, rejected, errors:[{index, code, message}]}`. This is the "no silent loss" contract (R13).
3. **Detection kick-off:**
   - After commit, the api calls `temporal.workflow.start(detectBatch, {workflowId:'detect-'+batchId, args:[{orgId,batchId}]})`, so the workflow id itself is idempotent.
   - If Temporal is unreachable, the batch stays `pending` and the `detection-sweep` schedule claims it within 60 s.
4. **`detectBatch` workflow** runs activities `claimBatch` → `evaluateBatch` → `finalizeBatch`:
   - `claimBatch`: `UPDATE ingest_batches SET claimed_at=now, detection_status='processing' WHERE id=$1 AND claimed_at IS NULL`. Zero rows means it was already claimed, so it exits.
   - `evaluateBatch`:
     - loads the enabled rules' current versions
     - evaluates each event against each rule (`domain/rules.ts`): match clauses, plus thresholds that count events over a window grouped by a field, measured in event time
     - creates or extends a correlation group (`domain/correlate.ts`): same org + shared entity (src IP / asset / indicator) within 30 min of event time
     - creates an `alert` with `alert_events` evidence rows, a risk score (severity × confidence, `domain/risk.ts`), and the MITRE technique from the rule
     - enqueues notifications per policy (`pending_emails`, `pending_deliveries`) **in the same transaction** as the alert
     - sends `pg_notify(alert.created)`
   - `finalizeBatch`: sets `detection_status='done'` with counts, or `failed` with an error (anomaly flag).
5. **Delivery:** the `email-drain` and `delivery-drain` schedules send queued notifications (capped backoff, DLQ). `escalation-check` re-notifies unacknowledged alerts past the policy's `escalate_after_min`.
6. **Realtime:**
   - The api holds one dedicated `pg` client running `LISTEN tw_events` and fans each notification out to the WebSocket connections of that `orgId` only.
   - Payloads carry ids and kinds, never row data. The client refetches through scoped REST, so authorization stays in one place.
7. **Investigate:** alert detail joins the evidence events, rule version, source (indicator profile), target asset, correlation-group siblings, intel and the `alert_activity` timeline. From there analysts assign, escalate, resolve or mark a false positive. Transitions are validated by `domain/states.ts`.

### 2.4 Scheduled work (Temporal Schedules: `has_scheduled_work`)

Every job is a pair of service functions, `runXForOrg(scope, now)` and `runX(now)` (the sweep iterates orgs and calls the per-org function). Each run writes a `job_runs` row with `processed`, `skipped`, `failed` and `error`. Workflows pass `now` from deterministic workflow time to the activities, and services never read `Date.now()` themselves. Schedules evaluate in **UTC**. Every schedule uses overlap policy `SKIP` and `catchupWindow: 10m`: a job that came due while the worker was down runs once on restart, and older due runs are skipped (startup catch-up). On SIGTERM the worker stops polling and lets in-flight activities finish within 10 s; anything unfinished is retried by Temporal (shutdown behavior).

| # | Trigger (schedule id) | Cadence | Business effect (asserted by its effect test) |
|---|---|---|---|
| T1 | `detection-sweep` | every 1 min | `pending` ingest batches older than 30 s → claimed and evaluated; alerts exist for matching events |
| T2 | `email-drain` | every 1 min | `pending_emails` due → `sent` (SMTP accepted) or `failed`/backoff → `dlq` after 3 |
| T3 | `delivery-drain` | every 1 min | `pending_deliveries` due → `delivered` (2xx) or backoff 1m/5m/30m/2h/6h → `dlq` on the 6th failed attempt (ADR-027) |
| T4 | `escalation-check` | every 1 min | unacknowledged alerts past the policy threshold → `escalated_at` set, escalation notification queued exactly once |
| T5 | `ingestion-health` | every 5 min | integrations with no events for longer than `silence_after_min` → health `silent`; recent errors > 20% → `degraded`; recovered → `healthy` |
| T6 | `retention-sweep` | daily 03:00 UTC | events older than the org's `retention_days` deleted (alert evidence rows retained via snapshot); `webhook_receipts` older than 7 days deleted |
| T7 | `report-generate` | hourly at :05 | report schedules due (daily/weekly) → `reports` row with the aggregate snapshot; an email to each recipient queued |

Manual trigger, which the smoke test uses: `POST /api/v1/admin/jobs/:job/run` (owner only, enabled only when `JOB_TRIGGER_ENABLED=true`, 404 otherwise). It starts `runJobForOrg(job, orgId)` as a Temporal workflow, so the real worker path is exercised, and returns the `job_runs` row.

### 2.5 Tenancy & authorization

- Every tenant table has `org_id NOT NULL` with an index led by `org_id`. Services receive a `TenantScope`; queries always include `where: { orgId: scope.orgId, … }`. Lookups by id use `findFirst({where:{id, orgId}})`, and a miss returns 404 (§7).
- RBAC lives in `services/auth/rbac.ts` as a single permission matrix (SPEC §4). Routes declare `requirePermission('alerts:write')`.
- Global read-only reference data (`geo_ip_ranges`, `indicators` with `org_id IS NULL` from the bundled feed, `mitre_techniques`) is not tenant data.
- `scale: small`: single api and worker instance. Statelessness is preserved (sessions in Postgres, realtime via LISTEN/NOTIFY), so horizontal scaling needs no redesign: add api replicas behind Caddy, then move the rate-limit store to a shared store (R12, Deferred measurement).

## 3. Agent and Tool Orchestration Design

ThreatWatch has no AI/LLM component at runtime. The "agents" are deterministic Temporal workflows:
- `detectBatch`: per batch, started on ingest.
- `runJobForOrg`: manual trigger.
- seven scheduled sweep workflows.

Workflow code is restricted to orchestration (`proxyActivities`, `sleep`, workflow time). Activities are thin adapters to services. Activity retry policy: initial 1 s, backoff ×2, max 30 s interval, 5 attempts for DB activities. Delivery activities don't retry at the Temporal level, because the outbox owns retries (avoids double backoff).

**Build orchestration (MDLC):**
- Stage 3 uses a single builder agent per feature, in dependency waves (BUILD-PLAN.md).
- An `invariant-check` subagent and `/smoke` command are emitted in Stage 4 for continuation.
- Governance runs through the `govern` hook on every commit after the plan.

## 4. Data Layer

PostgreSQL 17 with Prisma 7.10 (`@prisma/adapter-pg`, explicit `output`). IDs are UUID v7, generated in the app for time-sortable cursors. All timestamps are `TIMESTAMPTZ` in UTC. Coordinates are `NUMERIC(9,6)` with CHECK bounds. There is no FLOAT anywhere (ADR-006). Key tables (full columns in SPEC §2):

| Table | Purpose | Key constraints |
|---|---|---|
| `organizations` | tenant | `slug` UNIQUE, `retention_days` CHECK 7..365 |
| `users` | account (one org per user, v1) | `email` UNIQUE (lower-cased), `org_id`, `role` CHECK in (owner, admin, analyst, viewer), `disabled_at` |
| `sessions` | server sessions | `token_hash` UNIQUE, `expires_at`, `user_id` |
| `audit_logs` | append-only audit trail | `org_id, created_at` index |
| `idempotency_keys` | side-effecting POST replay | UNIQUE `(org_id, key)`, stored response, 24 h TTL |
| `integrations` | event sources | `secret_enc`, `kind`, `status`, `health`, `silence_after_min` |
| `webhook_receipts` | inbound replay guard | UNIQUE `(integration_id, message_id)`, retention 7 d (≥72 h) |
| `ingest_batches` | detection outbox | `claimed_at`, `detection_status` CHECK, `anomaly` |
| `ingest_errors` | malformed events | `integration_id, created_at` index |
| `events` | normalized events | UNIQUE `(integration_id, source_event_id)`; `(org_id, event_time)`, `(org_id, src_ip)`, `(org_id, asset_id)` indexes |
| `assets` | targeted assets | UNIQUE `(org_id, identifier)` |
| `indicators` | threat intel (global feed rows have NULL org) | UNIQUE `(org_id, type, value)` NULLS NOT DISTINCT |
| `geo_ip_ranges` | offline geo | `start_ip, end_ip` (inet), lat/lng NUMERIC(9,6) CHECK |
| `rules` / `rule_versions` | detection rules + immutable versions | UNIQUE `(rule_id, version)` |
| `correlation_groups` | connected activity | `(org_id, entity_type, entity_value, last_seen)` |
| `alerts` / `alert_events` / `alert_activity` | alerts, evidence, timeline | `status` CHECK; UNIQUE `(alert_id, event_id)` |
| `notification_policies` | who and how per severity | channels: email, delivery endpoint |
| `delivery_endpoints` | customer webhook targets | `url` (https), `secret_enc` |
| `pending_emails` | email outbox (`has_email`) | `(status, next_attempt_at)` index; UNIQUE `(to_address, template_key, business_ref)` |
| `pending_deliveries` | outbound webhook outbox (`has_webhook_send`, kernel name `pending_webhooks`, ADR-011) | `(status, next_retry_at)` index; `signature` NOT NULL at insert |
| `report_schedules` / `reports` | reporting | `cadence` CHECK |
| `job_runs` | scheduled-work observability | `(job, org_id, started_at)` |

Migrations are SQL from `prisma migrate dev`, each with a hand-written `down.sql` beside it. The deploy runs `prisma migrate deploy` in the `migrate` one-shot. CI runs migrations against an empty DB plus the seed.

## 5. Design Patterns

- **Hexagonal-lite:** domain (pure) → services (I/O, tenancy) → adapters (Express routes, Temporal activities, WS hub). Routes never touch Prisma (INV-13).
- **Transactional outbox** for every second-system write: Temporal kick-off (`ingest_batches`), email (`pending_emails`), outbound webhooks (`pending_deliveries`).
- **Idempotent claim** `UPDATE … WHERE claimed_at IS NULL` (batches) and `UPDATE … WHERE status='pending' AND next_attempt_at <= now` with `FOR UPDATE SKIP LOCKED` (outboxes).
- **State machines as data** (`domain/states.ts`): alert, email, delivery, batch and integration health. Every admitted status has a writer (INV-20…).
- **Problem Details** everywhere (§7 envelope); `requestId` propagates from the Caddy `X-Request-Id` or is minted.
- **Server-push invalidation** over WS (ids only) plus a scoped REST refetch.
- **Design system as tokens:** `tokens.css` CSS variables mapped into Tailwind v4 `@theme`. Components consume semantic tokens only (INV-19).

## 6. Dependencies (pinned exact)

| Package | Version | Where | Why |
|---|---|---|---|
| next | 16.3.8 | web | constraint (App Router, `proxy.ts`) |
| react / react-dom | 19.3.0 | web | Next 16 peer |
| tailwindcss / @tailwindcss/postcss | 4.3.3 | web | token mapping |
| express | 5.2.1 | server | constraint |
| ws | 8.22.0 | server | realtime |
| prisma / @prisma/client / @prisma/adapter-pg | 7.10.0 | server | ORM (ADR-004) |
| pg | 8.x (pinned at install) | server | adapter + LISTEN client |
| @temporalio/client, worker, workflow, activity | 1.24.0 | server | constraint |
| zod | 4.6.5 | server, web | validation, env schema, OpenAPI source |
| pino / pino-http | 10.4.0 / 11.0.0 | server | structured logs |
| helmet | 8.3.0 | server | headers on API responses |
| express-rate-limit | 8.7.0 | server | rate limiting (memory store, small tier) |
| argon2 | 0.45.1 | server | password hashing |
| nodemailer | 7.x (pinned at install) | server | SMTP transport (drain only) |
| typescript | 5.9.3 | all | ADR-004 |
| vitest | 5.0.3 | server, web | unit/integration |
| @playwright/test / @axe-core/playwright | 1.63.0 / 4.13.0 | web | e2e, a11y, web-baseline runner |
| lighthouse | latest at install, pinned | root dev | web-baseline at standard |
| eslint | 10.12.0 + typescript-eslint | all | lint |

No paid APIs. External network at runtime is only customer delivery URLs (SSRF-guarded) and SMTP.

## 7. Security Architecture (OWASP Top 10 floor)

- **Transport:**
  - Caddy `tls internal` on `:443` (published as 9443), with `:80` → `:443` redirect.
  - An HSTS policy (`max-age=31536000; includeSubDomains`) is emitted only when `HSTS_ENABLED=true`. The local self-signed deploy sends `max-age=0`, the RFC 6797 "no policy" value, so browsers never pin localhost (ADR-025).
  - Caddy's header buffers are generous by default, so no tuning is needed (DECISIONS ADR-012). Node services start with `--max-http-header-size=32768`.
- **Headers:**
  - `web/src/proxy.ts` sets `Content-Security-Policy` with a per-request nonce (`script-src 'self' 'nonce-…' 'strict-dynamic'`, `style-src 'self' 'unsafe-inline'`, `connect-src 'self' wss:`, `frame-ancestors 'none'`, `object-src 'none'`, `base-uri 'self'`, `form-action 'self'`).
  - helmet on the API sets `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin` and `Cross-Origin-Opener-Policy`.
  - Caddy adds `Permissions-Policy`.
- **Sessions & CSRF:**
  - Cookie `__Host-tw_session`: 32 random bytes, stored as a SHA-256 hash, HttpOnly, Secure, SameSite=Lax, Path=/, 12 h sliding idle with a 7 d absolute expiry, rotated at login and on password change.
  - State-changing requests require `Origin` to be in `APP_ORIGINS`, otherwise 403 `CSRF_ORIGIN`.
- **Passwords:** argon2id (m=19456, t=2, p=1). Length 12–128, screened against a bundled top-10k breached list. Login failures return a generic message. Rate limits: 10/min/IP plus 5/min per email on login, 5/h/IP on register.
- **Rate limits:** global 300/min/IP on `/api`, ingest 120 req/min per integration plus 600/min/IP (ADR-028), search 60/min/user.
- **Secrets:** AES-256-GCM using `APP_ENCRYPTION_KEY` (32-byte base64, env-validated). Integration and delivery secrets are shown once.
- **SSRF:** `server/src/lib/ssrf.ts` `assertSafeUrl(url)`:
  - https only (http allowed only when `ALLOW_HTTP_DELIVERY=true`, for tests)
  - resolves DNS and rejects private, loopback, link-local, CGNAT, multicast and metadata (169.254.169.254, fd00::/8) ranges
  - pins the resolved IP for the request
  - pinned-IP `http(s).request` (redirects never followed, ADR-029), 5 s timeout
- **Logging:** pino redaction of `req.headers.cookie`, `authorization`, `webhook-signature`, `*.password`, `*.secret*` and `email` in auth logs. Audit log entries cover auth events, role changes, integration and rule changes, alert transitions and policy changes.
- **Dependencies:** `npm audit --omit=dev --audit-level=high` in CI. Exact pins, committed lockfile.

## 8. Key Technical Decisions & Alternatives

| Decision | Chosen | Alternatives | Why |
|---|---|---|---|
| Background jobs | Temporal schedules + workflows (CLI dev server) | node-cron, BullMQ | Constraint is Temporal. Dev server is the supported single-container option (ADR-004) |
| Realtime transport | `ws` + Postgres LISTEN/NOTIFY | Socket.IO + Redis | No Redis at the small tier. Checklist-friendly raw WS |
| Rule language | JSON DSL (match clauses + threshold) | Sigma YAML, KQL | Testable, UI-editable, Sigma-inspired vocabulary |
| Geo | Seeded offline IP-range table | MaxMind/DB-IP full MMDB, map API | Image size, licensing, no billing vector (ADR-010) |
| Map rendering | Inline SVG equirectangular world (Natural Earth 110m simplified, public domain) | Mapbox/Leaflet tiles | CSP-clean, offline, no third-party requests |
| Auth | DB sessions + argon2id | JWT, OAuth/SSO | Revocable. SSO is a Non-Goal (ADR-005) |
| One org per user | `users.org_id` | memberships M:N | Smaller surface for v1. Logged as a Deferred extension |
| API ↔ web topology | Same origin via Caddy path routing | Separate subdomains | Cookie scoping, CSRF simplicity, one TLS endpoint |

## 9. Architectural Invariants

Each rule maps 1:1 to an `invariants.json` entry by id.

| ID | Rule | Check | Reference |
|---|---|---|---|
| INV-1 | The governance ledger is well-formed, append-only, and every post-plan commit is decided | governance | GOVERN.md; BUILD.md §Governed Change |
| INV-2 | Web Delivery Baseline report passes and is fresh | web-baseline | WEB.md; FRONTEND-AUDIT.json |
| INV-3 | The pinned web-baseline runner is present | required-file `scripts/web-baseline.mjs` | WEB.md |
| INV-4 | Every SPEC §6 screen, non-internal endpoint and W1–W12 workflow is reachable through the UI, with a substantive e2e | ui-coverage `web/src/app/**` | SPEC §6, §5 |
| INV-5 | Health endpoint exists | required-file `server/src/api/routes/health.ts` | Service Floor |
| INV-6 | `.env.example` exists | required-file | Service Floor |
| INV-7 | Env schema module exists | required-file `server/src/config/env.ts` | Service Floor |
| INV-8 | No floating-point column types in migrations (money/coordinates) | forbidden-pattern `\b(FLOAT|REAL|DOUBLE PRECISION)\b` (only `.sql`/schema files scanned) | has_payments (deferred, ADR-006), has_geo |
| INV-9 | Idempotency keys unique per org | required-unique-constraint `idempotency_keys(org_id,key)` | Service Floor; §4 |
| INV-10 | Outbound delivery calls `assertSafeUrl` before `fetch(` | boundary-order in `server/src/services/deliveries/dispatch.ts` | has_webhook_send; §7 SSRF |
| INV-11 | Each scheduled trigger T1–T7 has an effect test asserting its business effect | manual (verify `server/test/jobs/*.test.ts`) | has_scheduled_work; §2.4 |
| INV-12 | Inbound replay guard unique on `(integration_id, message_id)` | required-unique-constraint `webhook_receipts` | has_webhooks |
| INV-13 | Prisma is used only in services, `lib/db.ts`, seeds and tests | forbidden-pattern `prisma\.\w+\.(findMany|findFirst|findUnique|create|createMany|update|updateMany|upsert|delete|deleteMany|count|aggregate|groupBy)\(` outside services/db/seed/test | §5 hexagonal-lite; tenancy |
| INV-14 | Email is sent only by the drain transport (never synchronously) | forbidden-pattern `sendMail\(` outside `server/src/services/email/transport.ts` | has_email |
| INV-15 | Server-side `fetch(` only in the SSRF-guarded dispatcher | forbidden-pattern `\bfetch\(` outside web/, the dispatcher, tests and scripts | §7 SSRF |
| INV-16 | Ingest verifies the signature before ingesting | boundary-order `verifyIngestSignature\(` → `ingestBatch\(` in `server/src/api/routes/ingest.ts` | has_webhooks |
| INV-17 | WS upgrade checks Origin before accepting | boundary-order `isAllowedOrigin\(` → `handleUpgrade\(` in `server/src/api/ws/hub.ts` | has_websocket |
| INV-18 | No `dangerouslySetInnerHTML` outside the nonce'd root layout | forbidden-pattern | §7 XSS |
| INV-19 | No raw hex colours in UI components (tokens only) | forbidden-pattern `#[0-9A-Fa-f]{6}\b` outside tokens/globals and non-UI files | ADR-002/003 |
| INV-20 | Events dedupe on `(integration_id, source_event_id)` | required-unique-constraint `events` | R13 |
| INV-21 | Assets unique per org identifier | required-unique-constraint `assets(org_id, identifier)` | §4 |
| INV-22 | Rule versions are immutable and unique | required-unique-constraint `rule_versions(rule_id, version)` | W8 versioning |
| INV-23 | Email dedupe on `(to_address, template_key, business_ref)` | required-unique-constraint `pending_emails` | has_email hint |
| INV-24…28 | Alert statuses `acknowledged`, `investigating`, `resolved`, `false_positive`, `escalated` (escalated_at) are written by a writer surface | required-pattern in `server/src/services/**` | alert state machine |
| INV-29…31 | Email statuses `sent`, `failed`, `dlq` are written | required-pattern in `server/src/services/email/**` | has_email |
| INV-32…34 | Delivery statuses `delivered`, `failed`, `dlq` are written | required-pattern in `server/src/services/deliveries/**` | has_webhook_send |
| INV-35…37 | Batch statuses `processing`, `done`, `failed` are written | required-pattern in `server/src/services/ingest/**` | has_dual_write |
| INV-38…40 | Integration health `degraded`, `silent`, `healthy` are written | required-pattern in `server/src/services/integrations/**` | W9 health |
| INV-41 | Session cookie uses the `__Host-` prefix | required-pattern `__Host-tw_session` in `server/src/services/auth/**` | §7 |
| INV-42 | Mutations pass the CSRF Origin check | required-pattern `requireSameOrigin` in `server/src/api/app.ts` | §7 |
| INV-43 | Health route is reachable and reports every dependency | required-pattern `checks` in `server/src/api/routes/health.ts` | Service Floor |

Performance targets (SPEC §9; verified by smoke and integration timing, not by lint):
- detection < 30 s from ingest (p95 measured in the smoke test)
- typical dashboard query < 5 s (target < 500 ms at seed volume)
- investigation load < 10 s (target < 1 s)
- health ≤ 500 ms

Required CI gates: typecheck, lint, unit+integration tests, `lint:invariants`, build, Playwright e2e.

## 10. Implementation Notes & Risks

- **R1 scope:** the plan has 10 features. Every one of the 12 workflows is completable in the UI. Explicit deferrals are SSO, PDF export, live ThreatFox sync (ADR-013) and multi-org membership.
- **R8 latency:** the ingest → workflow start is immediate. The smoke test measures ingest → alert ≤ 30 s.
- **R12:** availability and horizontal-scaling metrics are reported as **Deferred (not measured)**.
- **Temporal image tag:** verify on the host at compose time (Gap 1). The worker waits for Temporal with a bounded retry (60 s) before exiting non-zero.
- **Next 16 proxy:** it sets the nonce and redirects only. Every authorization decision is re-made in the api (Next proxy-bypass advisories).
- **Windows host:** use LF line endings in shell scripts and the Caddyfile (`.gitattributes`).
