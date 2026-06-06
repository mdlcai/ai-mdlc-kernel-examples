# ARCHITECTURE.md — Recon

> Continuous external attack-surface monitoring SaaS.
> Source-of-truth precedence: RESEARCH.md > ARCHITECTURE.md > SPEC.md.
> Stage 1 artifact. Generated from RESEARCH.md (verified 2026-06-04). build_depth: **comprehensive**.

---

## 1. System Architecture

Recon is a multi-tenant SaaS composed of a **NestJS API**, a **Next.js web app**, a **Postgres** primary store, a **Redis/BullMQ** job + cache + websocket-fanout layer, and an **isolated scan-worker** tier. The scan worker is the crown-jewel security boundary — it is, by design, an authorized outbound-network engine and is the most locked-down component.

```
                         ┌──────────────────────────────────────────────┐
   Browser (SOC user) ───┤ Next.js Web (App Router, shadcn/ui, TanStack) │
                         └───────────────┬──────────────────────────────┘
                                         │ HTTPS only (/v1 REST + WS)
                         ┌───────────────▼──────────────────────────────┐
                         │ NestJS API  (Auth, RBAC, RLS tenant context)  │
                         │  Modules: auth, tenants, assets, scans,       │
                         │  changes, alerts, reports, audit, health, ws  │
                         └───┬───────────────┬───────────────┬───────────┘
            TypeORM (RLS)    │               │ BullMQ enqueue│ outbox relay
                         ┌───▼────┐     ┌─────▼─────┐   ┌─────▼──────────┐
                         │Postgres│     │Redis/Bull │   │ Outbox relay   │
                         │ 17 RLS │     │ Valkey    │   │ (webhooks/email│
                         └────────┘     └─────┬─────┘   │  Slack)        │
                                              │ jobs     └─────┬──────────┘
                                      ┌───────▼────────┐       │ signed HTTP / Resend
                                      │ Scan Worker    │   ┌───▼───────────────┐
                                      │ (egress-isolated│   │ Slack / Webhook / │
                                      │  subnet)        │   │ Resend / Sentry   │
                                      │ naabu + nmap +  │   └───────────────────┘
                                      │ TLS/CT probes   │
                                      └───────┬─────────┘
                                              │ guarded egress (deny RFC1918/metadata)
                                       Authorized external targets
```

### Layers
1. **Presentation** — Next.js 16 (App Router), shadcn/ui + Tailwind v4, TanStack Query v5 for server-state; WebSocket client for live scan/alert events. Token system copied verbatim from the uploaded design template (see §9 INV-TOKENS).
2. **API / Application** — NestJS 11 modular monolith. Guards: `JwtAuthGuard`, `RolesGuard`, `ThrottlerGuard`. Per-request **tenant context** sets the Postgres RLS GUC inside a transaction. REST under `/v1`, OpenAPI generated.
3. **Domain services** — asset registry, authorization-to-scan, scan orchestration, snapshot/diff change-detection, alert routing, reporting, audit.
4. **Async / workers** — BullMQ queues: `scan` (run a scan job), `diff` (compute change events), `outbox` (deliver webhooks/email/Slack), `report` (render exports). Job Schedulers drive recurring scans.
5. **Data** — Postgres 17 with Row-Level Security; JSONB scan snapshots; append-only `audit_log`; transactional `outbox` table; 12-month retention with monthly partitioning on snapshots.
6. **Platform** — Docker / docker-compose; Vault KV v2 for secrets; Sentry for errors; structured JSON logs (pino); HTTPS-only edge (nginx reverse proxy, HSTS, HTTP→HTTPS redirect).

## 2. Data Flow (Asset → Scan → Change → Alert)
1. A user registers an **asset** (domain or CIDR) under their org. The asset starts `pending_authorization`.
2. The org proves **authorization to scan** (DNS TXT token / HTTP file token). Until verified, no scan runs against that asset.
3. A **scan** is triggered (manual or by a BullMQ **Job Scheduler** on the asset's cadence). The scan job is enqueued; a per-target Redis lock prevents overlapping runs.
4. The **scan worker** (egress-isolated) resolves targets, re-checks the SSRF deny-list at scan time (anti-DNS-rebinding), runs port discovery (naabu) + service/version + TLS cert inspection + CT enrichment, and writes a normalized **snapshot** (sorted ports/services/cert fingerprints).
5. The **diff** job compares the new snapshot to the asset's last accepted **baseline**, emitting typed **change events** (`PORT_OPENED`, `PORT_CLOSED`, `SERVICE_CHANGED`, `CERT_ROTATED`, `CERT_EXPIRING`, `HOST_NEW`, `HOST_GONE`).
6. For each change event, an **outbox** row is written in the **same transaction**. The outbox relay delivers to configured **alert channels** (Slack Block Kit / signed webhook / Resend email) with retries + DLQ. WebSocket gateway broadcasts to the tenant's room for live dashboards.
7. All steps write to the append-only **audit_log**. Snapshots + change events feed **reports** (PDF/JSON/CSV).

## 3. Modules / Services (NestJS)
| Module | Responsibility | Key endpoints (prefix `/v1`) |
|---|---|---|
| `auth` | Register, login, session/JWT, password (NIST 800-63B), MFA-ready, anti-enumeration | `/auth/register`, `/auth/login`, `/auth/logout`, `/auth/me` |
| `tenants` | Org + membership, RBAC roles (owner/admin/analyst/viewer), invitations | `/orgs`, `/orgs/:id/members` |
| `assets` | Register/list/update/delete domains & CIDR ranges; authorization-to-scan lifecycle | `/assets`, `/assets/:id`, `/assets/:id/authorize`, `/assets/:id/verify` |
| `scans` | Trigger scans, schedule cadence, list runs, scan detail/snapshot | `/assets/:id/scans`, `/scans/:id`, `/assets/:id/schedule` |
| `changes` | Change events feed, acknowledge/triage, baseline accept | `/changes`, `/changes/:id/ack`, `/assets/:id/baseline` |
| `alerts` | Alert channels (Slack/webhook/email), routing rules, delivery log | `/alert-channels`, `/alert-channels/:id`, `/alert-deliveries` |
| `reports` | Generate + download audit/trend reports (PDF/JSON/CSV) | `/reports`, `/reports/:id`, `/reports/:id/download` |
| `audit` | Append-only audit log query | `/audit-log` |
| `health` | Liveness/readiness + dependency status | `/health` |
| `ws` (gateway) | Realtime scan/change/alert events, room-per-tenant | WS `/ws` |

## 4. Design Patterns
- **Tenant isolation** — Postgres RLS with `current_setting('app.tenant_id')`; a TypeORM transaction wrapper (`withTenantScope`) issues `SET LOCAL app.tenant_id` before any scoped query. App-layer `WHERE` alone is never trusted.
- **Outbox** — every change-event → outbox row in one ACID transaction; relay publishes at-least-once; receivers idempotent (dedupe on event id / `Idempotency-Key`).
- **Snapshot-diff** — normalized, hashed snapshots; cheap equality via hash, full JSONB retained for trend reporting.
- **Job Scheduler + lock** — BullMQ `upsertJobScheduler` per asset cadence; Redis `SETNX` per-target lock prevents stacked runs.
- **SSRF guard** — central `TargetGuard` resolves + validates every scan target and every user-supplied webhook URL against an internal-range deny-list; re-resolves at execution time.
- **Structured errors** — RFC 9457 `application/problem+json` with typed `code` + field-level `errors[]` (SPEC §7).

## 5. Dependencies (verified 2026-06-04)
| Dependency | Version | Purpose |
|---|---|---|
| NestJS | 11.x (pinned; v12 ESM deferred) | API framework |
| TypeORM | 1.x | ORM + migrations (RLS-compatible raw SQL) |
| pg | 8.x | Postgres driver |
| BullMQ | 5.x | Job queues + Job Schedulers |
| ioredis | 5.x | Redis/Valkey client |
| @nestjs/throttler | current | Rate limiting (Redis storage) |
| @nestjs/websockets + socket.io + @socket.io/redis-adapter | current | Realtime fanout |
| Next.js | 16.x LTS | Web app |
| shadcn/ui + Tailwind | v4 | UI + tokens |
| @tanstack/react-query | v5 | Server state |
| resend | current | Email delivery |
| @sentry/nestjs + @sentry/nextjs | 10.x | Error reporting |
| node-vault | current | Vault KV v2 (with env fallback for local) |
| pino / nestjs-pino | current | Structured JSON logs |
| argon2 | current | Password hashing (NIST 800-63B) |
| zod / class-validator | current | Input validation |
| Postgres | 17.x | Primary store |
| Valkey (Redis-compatible) | 8.x | Queue/cache/ws-adapter |

> **Scanner engines** (naabu MIT in-app; nmap/masscan as optional external binaries behind the worker boundary) are invoked as separate processes. For a self-contained, deterministic, license-clean, deploy-anywhere build, Recon ships a **native Node TCP/TLS scan engine** (`@recon/scanner`) as the default discovery+service+cert probe, with an adapter interface (`ScanEngine`) so naabu/nmap can be plugged in where those binaries are present and authorized. This keeps the default deploy free of AGPL/NPSL binaries and runnable in CI. See ADR-007.

## 6. Implementation Notes & Risks
- TypeORM 1.0 + RLS: verify pooled connections don't leak the tenant GUC — always `SET LOCAL` inside a transaction, never session-level (Research gap #1).
- Scan worker must run in an egress-filtered network; the metadata IP `169.254.169.254` and RFC1918/loopback/link-local ranges are denied at the application layer regardless (defense in depth) — INV-SSRF.
- At-least-once outbox → consumers idempotent; dedupe window on event id — INV-IDEMP.
- Authorization-to-scan is a hard gate: a scan against an unverified asset must be impossible by construction — INV-AUTHZSCAN.

## 7. Key Decisions (summary; full ADRs in DECISIONS.md)
- ADR-001 Multi-tenant via Postgres RLS (not schema-per-tenant).
- ADR-002 BullMQ Job Schedulers for recurring scans (not legacy bull repeatable).
- ADR-003 Outbox pattern for all alert dual-writes.
- ADR-004 RFC 9457 problem+json error contract.
- ADR-005 argon2id password hashing + breached-password screening (NIST 800-63B).
- ADR-006 Valkey over Redis 8 (license cleanliness).
- ADR-007 Native Node scan engine as default; external binaries pluggable.
- ADR-008 NestJS v11 pinned (defer v12 ESM migration).
- ADR-009 Vault KV v2 for secrets with env-var fallback for local/dev.

## 8. Trade-offs
- **Modular monolith over microservices** — simpler ops at this stage; scan workers are a separate process/queue consumer, giving the isolation that matters without full service sprawl.
- **Native scan engine default** — trades raw nmap depth for deploy portability, license cleanliness, and CI determinism; pluggable adapter preserves the upgrade path.
- **RLS over app-only scoping** — centralizes the hardest-to-get-right control (tenant isolation) in the database.

---

## 9. Architectural Invariants

These are binding rules. Each has a machine-checkable (or `manual`) entry in `invariants.json`. The Stage 4 invariant-lint pass and the ship-time Drift Detection Gate consume them. Never disable an invariant to pass a gate; amend §9 + invariants.json in the same commit with an ADR.

| ID | Rule | File/section ref | Check type |
|---|---|---|---|
| **INV-1 (TOKENS)** | Design tokens (colors, type, spacing, radius, shadow) come only from the design-token layer; no ad-hoc hex colors in web component source. | `apps/web/app/globals.css` `:root` (copied verbatim from DESIGN-TEMPLATE.html); DECISIONS ADR-010 | forbidden-pattern |
| **INV-2 (TENANT-RLS)** | Scoped DB access goes through `withTenantScope`; no direct repository calls to tenant tables outside the scope wrapper / data layer. | `apps/api/src/common/tenant` | forbidden-pattern |
| **INV-3 (RLS-MIGRATION)** | RLS is enabled on every tenant-scoped table via migration. | `apps/api/src/migrations` | required-file |
| **INV-4 (SSRF)** | All scan targets and user webhook URLs pass the central SSRF/internal-range guard before any outbound request. | `apps/api/src/common/security/target-guard.ts` | required-file |
| **INV-5 (AUTHZSCAN)** | A scan cannot run against an asset that is not in `authorized` state (enforced in scan orchestration). | `apps/api/src/scans` ; SPEC §4 | manual |
| **INV-6 (OUTBOX)** | Change-event alert delivery is written through the transactional outbox, not sent inline in the request path. | `apps/api/src/alerts/outbox` | required-file |
| **INV-7 (IDEMP-WEBHOOK)** | Webhook/outbox delivery has a unique idempotency constraint. | migration `outbox_delivery` unique on `(event_id, channel_id)` | required-unique-constraint |
| **INV-8 (WEBHOOK-SIGN-ORDER)** | Outbound webhook delivery signs the payload (`signPayload`) before performing the HTTP request (`fetch`). | `apps/api/src/alerts/webhook-sender.ts` | boundary-order |
| **INV-9 (PASSWORD-HASH)** | Passwords hashed with argon2id; no plaintext/weak hashing primitives in source. | `apps/api/src/auth` | forbidden-pattern |
| **INV-10 (RATE-LIMIT-AUTH)** | Auth + scan-trigger endpoints carry a throttle guard/decorator. | `apps/api/src/auth`, `apps/api/src/scans` | manual |
| **INV-11 (HTTPS-ONLY)** | Edge enforces HTTPS with HTTP→HTTPS redirect + HSTS. | `deploy/nginx/recon.conf` | required-file |
| **INV-12 (AUDIT-LOG)** | Audit log table exists and is append-only (no UPDATE/DELETE in app source). | `apps/api/src/audit`; migration `audit_log` | required-file |
| **INV-13 (UI-COVERAGE)** | Every SPEC §UI Surface screen route resolves; every §5 workflow has an e2e test to its terminal step. | `apps/web/app`, `apps/web/e2e`, SPEC.md | ui-coverage |
| **INV-14 (ENV-EXAMPLE)** | An `.env.example` documenting every env var exists. | `.env.example` | required-file |
| **INV-15 (NO-PLACEHOLDER)** | No `not implemented` / TODO-as-control-flow stubs in shipped source. | `apps/**/src` | forbidden-pattern |

> The invariant-lint runner (`scripts/invariant-lint.mjs`) implements every check type present in `invariants.json` and is wired into the Verification Gate (`npm run lint:invariants`). A check that inspects zero files (other than `required-file`) is a lint FAILURE.
