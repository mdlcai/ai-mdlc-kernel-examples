# ARCHITECTURE.md — ServiceHub

**Source of truth for architecture.** Behavior lives in `SPEC.md`. Derived from `RESEARCH.md` (build_depth: comprehensive). Version 0.1.0.

ServiceHub is a multi-tenant IT Service Management (ITSM) platform: employees submit IT requests/incidents; agents triage, comment, and resolve; managers monitor queues and SLA breaches. Hard isolation between tenants and same-tenant cross-user privacy are the defining non-functional requirements.

---

## 1. System Architecture

### 1.1 Topology (monorepo, ADR-09)
```
servicehub/
├─ apps/
│  ├─ api/            NestJS 11 — REST /v1, WebSocket gateway, guards, services
│  │  └─ (also hosts the BullMQ worker process: `start:worker`)
│  └─ web/            Next.js 16 (App Router) — marketing + authenticated app shell
├─ packages/
│  └─ shared/         Shared DTO/types, error codes, API contract constants
├─ docker-compose.yml postgres · redis · api · worker · web · caddy (TLS)
├─ Caddyfile          TLS termination, HTTP→HTTPS redirect, HSTS, header buffers
└─ invariants.json    machine-checkable §9
```

### 1.2 Layers (backend)
1. **Edge / Proxy** — Caddy terminates TLS (HTTPS only, ADR per `protocol_support`), forces HTTP→HTTPS redirect, sets HSTS, sizes large client header buffers.
2. **HTTP layer (NestJS)** — Controllers under `/v1`. Global `ValidationPipe({ whitelist, transform })`. Global exception filter → RFC 9457 problem+json (ADR-07). Helmet, CORS allowlist, rate-limit guard (`@nestjs/throttler` + Redis store).
3. **Auth & context layer** — `JwtAuthGuard` (access token) → populates `principal {userId, tenantId, role}`. `nestjs-cls` middleware stores tenant context in AsyncLocalStorage (ADR-02). `TenantTransactionInterceptor` opens a transaction and runs `SET LOCAL app.tenant_id` before the handler (ADR-01).
4. **Authorization layer** — `RolesGuard` (RBAC via `@Roles()` + `Reflector`). `assertResourceAccess(row, principal)` row-ownership chokepoint (ADR-03).
5. **Domain services** — `TicketsService`, `CommentsService`, `AttachmentsService`, `SlaService`, `NotificationsService`, `AuditService`, `ReportsService`, `AuthService`, `UsersService`, `WebhooksService`, `OutboxService`.
6. **Data layer (TypeORM 1.0)** — Entities + migrations; RLS policies installed by migration. Repositories used only through tenant-scoped transactions. Connection pool sized for medium scale.
7. **Async layer (BullMQ 5 + Redis)** — Queues: `email-outbox`, `webhook-outbox`, `sla-evaluator` (repeatable), `attachment-scan`, `report-export`. Worker process consumes; idempotent handlers.
8. **Realtime (socket.io)** — `EventsGateway`: JWT-authenticated handshake; rooms `tenant:<id>` and `ticket:<id>`; authz re-checked on room join (ADR / R9).
9. **Observability** — structured JSON logging (pino), `/api/health` healthcheck, metrics endpoint (`/metrics` Prometheus format), centralized error reporting hook (Sentry-compatible, env-gated).

### 1.3 Frontend architecture (Next.js 16 App Router)
- **Marketing surface** `/` — ported from `DESIGN-TEMPLATE.html` (nav, hero+app mockup, trust strip, feature cards, CTA, footer) using copied `:root` tokens.
- **Auth surfaces** `/login`, `/register`, `/forgot-password`, `/reset-password`.
- **App shell** (`(app)` route group) — persistent sidebar nav (224px) + contextual header + content region (saas archetype Layout Doctrine). Theme toggle (light/dark, system-aware, persisted).
- **Data layer** — TanStack Query v5; typed API client in `lib/api`; `useMutation` + `invalidateQueries` for optimistic ticket updates; query keys namespaced by tenant.
- **Realtime** — socket.io client subscribes to `ticket:<id>` / queue rooms; pushes invalidate relevant queries.
- **Design tokens** — single token layer in `app/globals.css` (`:root` + `[data-theme="dark"]`) copied verbatim from template; Tailwind theme maps to the CSS variables; shadcn primitives restyled to tokens.

### 1.4 Data flow — "submit ticket" (W1)
```
Employee → web form (FE validation) → POST /v1/tickets (JWT)
  → ValidationPipe → JwtAuthGuard(principal) → RolesGuard(requester+)
  → TenantTxInterceptor: BEGIN; SET LOCAL app.tenant_id
  → TicketsService.create: INSERT ticket (tenant_id, requester_id, status=NEW)
      + SlaService.computeInitialDueAt
      + AuditService.record('ticket.created')
      + OutboxService.enqueueEmail(confirmation)   ← same transaction
  → COMMIT
  → EventsGateway emits ticket.created to tenant queue room
  → email-outbox worker drains → Resend send → mark sent
  → 201 { ticketNumber, dueAt, status }
```

### 1.5 Data flow — "agent resolves" (W2)
Agent picks ticket → `PATCH /v1/tickets/:id/assign` (self) → `POST /v1/tickets/:id/comments {internal:true}` (role-gated; never visible to requester) → `PATCH /v1/tickets/:id/status {status:RESOLVED}` (state-machine guard, optimistic version check) → audit + SLA stop + outbox email to requester.

### 1.6 Data flow — "manager queue/SLA" (W4)
Manager → `GET /v1/tickets?scope=team&slaBreached=true` (RolesGuard manager) → server-computed SLA status (no client trust) → dashboard metric cards + table.

---

## 2. Agent / Service Orchestration Design
ServiceHub is not an LLM-agent product; "orchestration" = the async job graph.
- **SLA evaluator** (repeatable, every 60s): scans open tickets, recomputes `at_risk`/`breached`, emits `sla.breached` events + outbox emails. Single authority for SLA state (ADR-05).
- **Email outbox drain**: claims `pending_emails` rows (`status=PENDING`, `FOR UPDATE SKIP LOCKED`), sends via Resend, marks `SENT`/`FAILED` with capped exponential backoff → DLQ after N attempts. Idempotent: UNIQUE `(to_address, template_key, business_ref_id)`.
- **Webhook outbox drain**: same pattern for outbound webhooks; HMAC signs at INSERT time; `Idempotency-Key` header.
- **Attachment scan**: validates magic bytes, sets `scan_status` CLEAN/INFECTED; pluggable AV hook (G1).
- **Report export**: generates PDF/CSV/JSON asynchronously; stores artifact; notifies via socket + email.

---

## 3. Design Patterns
- **Transactional Outbox** (ADR-04) — guaranteed delivery.
- **Inbox / idempotent consumer** — `webhook_events` dedup table; `verify → check_idempotency → process`.
- **Optimistic locking** — TypeORM `@VersionColumn` on `tickets`; 409 + problem+json on mismatch (R7).
- **State machine** — explicit allowed ticket status transitions; guarded in `TicketsService.transition`.
- **Repository-through-tenant-transaction** — no raw repo access to tenant tables outside the scoped tx (RLS belt-and-suspenders).
- **Chokepoint authorization** — `assertResourceAccess` (ADR-03).
- **CLS context** — AsyncLocalStorage tenant/principal propagation (ADR-02).

---

## 4. Dependencies (pinned at Stage 3; versions from RESEARCH §3.1)
**Backend:** `@nestjs/core` ^11.1, `@nestjs/throttler`, `@nestjs/passport` ^11, `@nestjs/jwt` ^11, `@nestjs/websockets` ^11, `@nestjs/bullmq` ^11, `typeorm` ^1.0, `pg`, `bullmq` ^5.78, `ioredis`, `nestjs-cls`, `class-validator` ^0.15, `class-transformer`, `helmet`, `argon2` (password hashing), `resend` ^6.12, `nestjs-pino`/`pino`, `socket.io` ^4.8, `@socket.io/redis-adapter`, `file-type` (magic bytes), `pdfkit` (PDF export), `zod` (env validation).
**Frontend:** `next` ^16.2, `react` ^19, `@tanstack/react-query` ^5.101, `tailwindcss` ^4.3, shadcn/ui primitives, `socket.io-client`, `lucide-react`.
**Test/quality:** `jest`/`@nestjs/testing`, `supertest`, `@playwright/test` (e2e/UI), `eslint`, `typescript` ^5.

External APIs: Resend (email + inbound webhooks via Svix/Standard Webhooks).

---

## 5. Key Technical Decisions + Alternatives
See ADR-01..ADR-10 in DECISIONS.md. Summary: RLS+CLS for isolation; outbox for delivery; single SLA evaluator; chokepoint authz with 404-to-non-owner; RFC 9457 errors; JWT+refresh auth with anti-enumeration; storage abstraction; Caddy TLS.

---

## 6. Data Model (overview; full DDL in SPEC §3)
Core tenant-scoped tables (all carry `tenant_id`, RLS-protected): `tenants`, `users`, `tickets`, `ticket_comments`, `ticket_attachments`, `ticket_events` (timeline/audit), `sla_policies`, `categories`, `pending_emails`, `pending_webhooks`, `webhook_events`, `audit_logs`, `notifications`, `refresh_tokens`. Global/non-tenant: `migrations`.

Key constraints: UNIQUE `(tenant_id, ticket_number)`; UNIQUE `(to_address, template_key, business_ref_id)` on `pending_emails`; UNIQUE `(provider_event_id)` on `webhook_events`; UNIQUE `(tenant_id, email)` on `users`; FK + `tenant_id` on every child row; `@VersionColumn` on `tickets`.

---

## 7. Implementation Notes & Risks
- RLS context MUST be set inside the same transaction as the query (R1). The `TenantTransactionInterceptor` is the only place it is set.
- Webhook signature verification MUST run before any DB read (R6).
- Auth rate-limit MUST fire before the argon2 verify (R4).
- Attachments served only via a signed, ownership-checked endpoint — never a static path (R2/R3).
- Header buffers sized on the Caddy proxy and Node (`--max-http-header-size=32768`) to survive accumulated-cookie requests.

---

## 8. Performance & Scale (`scale: medium`, 1k–50k concurrent)
- PG connection pooling (pgbouncer-style via pool config); Redis cache for SLA policy + category lookups + session/refresh denylist.
- Real BullMQ queues (mandatory at medium tier).
- Targets (SPEC §9 acceptance): ticket submit < 2s, dashboard < 1.5s, search < 500ms, SLA compliance 95%.
- Indexes: `tickets(tenant_id, status, priority, due_at)`, `tickets(tenant_id, requester_id)`, `ticket_events(ticket_id, created_at)`, full-text index on ticket subject/body for search.

---

## 9. Architectural Invariants

Every rule below has a machine-checkable or `manual` entry in `invariants.json`. Drift Detection and the Stage 4 invariant-lint consume that file.

| ID | Rule | File / section ref | Check type |
|----|------|--------------------|-----------|
| INV-1 | Every tenant-scoped DB query runs inside the tenant transaction wrapper; no raw `getRepository`/`dataSource.manager` writes to tenant tables outside `apps/api/src/common/tenant`. | §1.2 layer 6, ADR-01 | forbidden-pattern |
| INV-2 | `assertResourceAccess` (object-level authz chokepoint) exists and is the only ownership gate; row-owned ticket routes + sub-resources call it. | §1.2 layer 4, ADR-03 | required-file + manual |
| INV-3 | RLS is enabled on tenant tables: a migration enables row level security and defines tenant policies. | §1.2 layer 6, ADR-01 | required-file (migration) + manual |
| INV-4 | `tickets` has UNIQUE `(tenant_id, ticket_number)`. | §6 | required-unique-constraint |
| INV-5 | `pending_emails` has UNIQUE `(to_address, template_key, business_ref_id)` (email idempotency). | §6, ADR-04 | required-unique-constraint |
| INV-6 | `webhook_events` has UNIQUE `(provider_event_id)` (inbound webhook dedup). | §6, ADR-04 | required-unique-constraint |
| INV-7 | `users` has UNIQUE `(tenant_id, email)`. | §6 | required-unique-constraint |
| INV-8 | Inbound webhook handler verifies signature before any DB read: `verifySignature` precedes any repository/query call in the webhook controller. | §7, R6 | boundary-order |
| INV-9 | Auth rate-limit guard precedes password hash verification in the auth controller/service. | §7, R4 | boundary-order |
| INV-10 | All API errors are RFC 9457 problem+json via a single global exception filter. | §1.2 layer 2, ADR-07 | required-file + manual |
| INV-11 | No hard-coded hex colors / off-scale font-sizes in app UI source — colors/sizes/spacing come from the design-token layer (CSS vars / Tailwind theme). Exempt: the token-definition files (globals.css/marketing.css) and transactional **email** HTML templates (email clients do not support CSS custom properties). | §1.3, DESIGN tokens | forbidden-pattern |
| INV-12 | Design template `:root` tokens are copied verbatim into `apps/web/app/globals.css`. | §1.3, ADR (DESIGN template) | required-file + manual |
| INV-13 | Password hashing uses argon2 (no bcryptjs/argon2-browser substitution); secrets only from env. | §4, R4/R10 | forbidden-pattern |
| INV-14 | Attachments are served only through the ownership-checked download controller, never a static/public path. | §7, R2/R3 | manual |
| INV-15 | Every §UI Surface screen route resolves to a rendered route; every non-internal SPEC endpoint is referenced by UI source; every Key Workflow (W1–W4) has an e2e test driving it through the UI to its terminal step. | SPEC §UI Surface, §5 | ui-coverage |
| INV-16 | The invariant-lint runner exists and is wired as `lint:invariants`. | Stage 4 | required-file |

Invariant count: **16**.
