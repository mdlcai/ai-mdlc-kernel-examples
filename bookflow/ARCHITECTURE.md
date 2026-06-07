# ARCHITECTURE.md — BookFlow

> Source of truth for **architecture**. Behavior lives in `SPEC.md`. Architectural rules are enumerated in §9 and machine-encoded in `invariants.json`.
> Build version: see `VERSION.md`. Depth: `comprehensive`. Archetype: `admin`. Tier: `small (<1k concurrent)`.

## 1. System Overview

BookFlow is a **multi-tenant, configurable booking platform**. A monorepo with two deployables plus a worker, backed by one PostgreSQL primary:

```
┌──────────────────────────────────────────────────────────────────────┐
│ Browser (Next.js App Router · shadcn/ui · Tailwind · TanStack Query)   │
│   ─ marketing landing (ported from DESIGN-TEMPLATE.html)               │
│   ─ authenticated admin console (sidebar + content, dense data tables) │
└───────────────┬──────────────────────────────┬───────────────────────┘
                │ REST /v1 (Bearer JWT)         │ SSE /v1/stream (live)
                ▼                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ NestJS API (apps/api)                                                  │
│  Guards: JwtAuthGuard → RolesGuard → TenantGuard                       │
│  Modules: auth · orgs · users · resources · bookings · approvals ·     │
│           workflows · notifications · audit · reports · webhooks · sse │
│  Cross-cutting: TenantContext (CLS) · ScopedRepository · Throttler ·   │
│                 AuditInterceptor · ProblemJson filter · structured log │
└───────────────┬───────────────────────────────┬──────────────────────┘
                │ TypeORM                         │ enqueue (same tx)
                ▼                                  ▼
┌───────────────────────────┐        ┌───────────────────────────────────┐
│ PostgreSQL 16              │        │ Outbox Worker (apps/api worker     │
│  btree_gist + tstzrange    │        │  process)                          │
│  EXCLUDE no-double-booking │        │  polls outbox FOR UPDATE SKIP      │
│  append-only audit_log     │        │  LOCKED → Resend (email) +         │
│  outbox / inbox tables     │        │  outbound webhooks; retry→DLQ      │
└───────────────────────────┘        └─────────────────┬─────────────────┘
                                                        ▼ Svix-signed
                                          Resend  ◄──►  inbound webhooks
                                          (email)       (verify→idempotency→process)
```

**Why this shape:** at <1k concurrent we need no Redis/replicas, but the *outbox worker* (dual-write correctness) and *SSE channel* (live availability/approvals) are correctness/availability requirements, not scaling ones, so they exist from v1.

## 2. Layers & Modules

| Layer | Responsibility | Key files |
|---|---|---|
| **Presentation (web)** | RSC pages + client components; TanStack Query for server-state; SSE subscription | `apps/web/app/**`, `apps/web/lib/api.ts`, `apps/web/lib/query.ts` |
| **API controllers** | REST `/v1/*` endpoints, DTO validation, problem+json errors | `apps/api/src/<module>/*.controller.ts` |
| **Domain services** | Business rules, state machines, transactions | `apps/api/src/<module>/*.service.ts` |
| **Persistence** | TypeORM entities + migrations; scoped queries | `apps/api/src/**/entities/*.entity.ts`, `apps/api/src/migrations/*` |
| **Cross-cutting** | Tenant context, RBAC, throttling, audit, logging, outbox | `apps/api/src/common/**` |
| **Worker** | Outbox drain → email/webhook delivery | `apps/api/src/outbox/outbox.worker.ts` |

### Module responsibilities
- **auth** — register/login/refresh/logout, JWT issue/verify, password hashing (bcrypt), anti-enumeration, rate-limit before hash compare.
- **orgs** — tenant (organization) lifecycle; first-user becomes org admin on registration.
- **users** — membership, roles within an org.
- **resources** — reservable items (room/equipment/service), capacity, attributes, maintenance windows, dependencies.
- **bookings** — create/list/detail/modify/cancel; availability check; DB-enforced conflict prevention; status machine.
- **approvals** — approval requests, decisions (approve/reject/request-changes), SLA, audit.
- **workflows** — per-tenant configurable approval rules (advance-notice, budget threshold, resource-type routing, multi-step chains).
- **notifications** — enqueue templated emails to outbox (never sync-send).
- **webhooks** — inbound Resend (delivery/bounce/complaint) verify→idempotency→process; outbound webhook delivery via outbox.
- **audit** — append-only audit log; query/export.
- **reports** — utilization, approval cycle time, CSV/PDF export.
- **sse** — `/v1/stream` per-tenant live channel (booking + approval events).
- **health** — `/api/health` liveness/readiness with dependency checks.

## 3. Data Flow (representative)

**Booking submission (write path):**
1. `POST /v1/bookings` → `JwtAuthGuard` (valid token) → `TenantGuard` (binds `org_id` to CLS context) → `RolesGuard` (member+).
2. `BookingsService.create` opens a transaction: insert booking row scoped to `org_id`. The `EXCLUDE USING GIST` constraint atomically rejects overlaps (→ mapped to `409 booking_conflict`).
3. `WorkflowsService` evaluates the tenant's active workflow rules → determines required approval steps; booking status set `pending_approval` or auto-`confirmed`.
4. In the **same transaction**, a notification row is written to `outbox` (email to approver/requester) — dual-write solved by outbox.
5. Commit. `SseService` publishes a `booking.created` event to the tenant channel. `AuditInterceptor` writes an audit entry.
6. Outbox worker (separate process) later picks up the row (`FOR UPDATE SKIP LOCKED`), sends via Resend, marks sent; failure → backoff retry → DLQ.

**Approval decision:** `PATCH /v1/approvals/:id` → version-locked transition (optimistic `@VersionColumn`) → booking status advances → outbox notification → SSE `approval.decided` → audit.

**Inbound webhook (Resend):** `POST /v1/webhooks/resend` (raw body) → verify Svix HMAC on raw bytes → idempotency insert on `svix-id` (skip if seen) → update notification delivery status in tx → 2xx.

## 4. Key Technical Decisions (summary; ADRs in DECISIONS.md)

| Decision | Choice | Alt considered | Rationale |
|---|---|---|---|
| Tenant isolation | Row-level `org_id` + scoped repo (CLS) | schema-per-tenant, RLS | Right fit at <1k concurrent; simplest correct; RLS deferred as defense-in-depth. |
| No-double-booking | Postgres `EXCLUDE USING GIST` (tstzrange) | app-level check, advisory locks | Structural guarantee across all paths; race-proof. |
| Dual-write | Transactional outbox + worker | direct send, 2PC | Atomic with business tx; idempotent delivery; matches `has_dual_write`. |
| Realtime | SSE | WebSocket | One-way push suffices ≤1k concurrent; simpler, auto-reconnect. |
| Auth | JWT access + refresh, bcrypt | sessions | Stateless API; RFC 6750 bearer; rotation via refresh. |
| Errors | RFC 7807 `problem+json` | ad-hoc JSON | Field-level detail the UI consumes onto controls. |
| Config | env vars + `.env.example` (vault in prod) | hardcode | `secrets_management` constraint. |

## 5. Dependencies (resolved versions — see package manifests for pins)

**API:** `@nestjs/*` 11.x, `typeorm` 0.3.x, `pg`, `@nestjs/throttler` 6.x, `bcrypt`, `@nestjs/jwt`, `class-validator`/`class-transformer`, `nestjs-cls`, `resend`, `pino`/`nestjs-pino` (structured JSON), `helmet`.
**Web:** `next` (App Router), `react`, `@tanstack/react-query` 5.x, `tailwindcss`, shadcn/ui components, `zod`.
**Infra:** PostgreSQL 16 (`postgres:16-alpine`), Docker Compose.

## 6. Implementation Notes & Risks

- **Header-buffer sizing**: Node serving paths start with `--max-http-header-size=32768` (accumulated cookie jar). nginx reverse proxy (prod) sets `large_client_header_buffers 8 32k`.
- **Caching**: availability/list endpoints are `no-store` on the web side; live state via SSE.
- **Outbox poller** uses `FOR UPDATE SKIP LOCKED` so it is safe under future multi-instance scaling.
- **Time**: all timestamps `timestamptz` (UTC); ranges via `tstzrange(starts_at, ends_at, '[)')` (half-open).
- **Risks**: see RESEARCH.md §5 risk register + threat model. Top: cross-tenant leak (mitigated by scoped repo + tests), double-booking (DB constraint), dual-write (outbox).

## 7. Trust Boundaries (threat model → controls)

Mirrors RESEARCH.md §5 threat model. Enforced by: `TenantGuard` + `ScopedRepository` (tenant↔tenant), `RolesGuard` (privilege), `ThrottlerGuard` before hash compare (anon→auth), Svix verify + idempotency (webhook ingress), outbox (email egress), `EXCLUDE` constraint (booking write), `AuditInterceptor` (repudiation).

## 8. Design Token Contract

The design-token system is canonical from `DESIGN-TEMPLATE.html` `:root` (copied verbatim into `apps/web/app/globals.css` and Tailwind theme). Light tokens are verbatim; dark tokens are derived alongside (never replacing light values). No hard-coded hex/px outside the token layer. This is INV-10/INV-11 below.

---

## 9. Architectural Invariants

> Every rule here has a matching entry in `invariants.json` (`id` cross-references). Machine-checkable rules use a typed check; non-checkable rules are `manual`. Amendments update §9 **and** `invariants.json` in the same commit with an ADR. Never disable an invariant to make a gate pass.

| ID | Rule | File / section ref | Check type |
|---|---|---|---|
| **INV-1** | No direct TypeORM repository access to tenant-scoped tables outside the scoped-repository layer — every tenant query goes through `ScopedRepository`/`withOrgScope`. | `apps/api/src/common/tenant/**` | `forbidden-pattern` |
| **INV-2** | The bookings table must carry a Postgres exclusion constraint preventing overlapping non-cancelled bookings per resource. | booking migration | `required-unique-constraint` (proxy: unique/exclude assertion) + `manual` |
| **INV-3** | Outbox row and the business row it describes are written in the same DB transaction (no sync email/webhook send in request path). | `apps/api/src/notifications/**`, `outbox/**` | `forbidden-pattern` (no `resend.emails.send` outside worker) |
| **INV-4** | Inbound webhook handler verifies signature **before** any DB read/parse of event content. | `apps/api/src/webhooks/resend.controller.ts` | `boundary-order` |
| **INV-5** | Every authenticated endpoint is protected by `JwtAuthGuard` (globally applied; `@Public()` opt-out only). | `apps/api/src/common/guards/**`, `main.ts` | `manual` |
| **INV-6** | Auth rate limiting fires before the password hash compare. | `apps/api/src/auth/auth.service.ts` | `boundary-order` |
| **INV-7** | All API errors are emitted as RFC 7807 `application/problem+json` via the global exception filter. | `apps/api/src/common/filters/problem-json.filter.ts` | `required-file` |
| **INV-8** | Audit log is append-only; every state-mutating action records who/what/when/org. | `apps/api/src/audit/**` | `manual` |
| **INV-9** | A CI/lint config for invariants exists and runs in the verification gate. | `scripts/invariant-lint.mjs` | `required-file` |
| **INV-10** | Design tokens are defined once (`:root` in `globals.css`) and consumed everywhere; no raw hex colors in component files. | `apps/web/app/globals.css`, `apps/web/**/*.tsx` | `forbidden-pattern` |
| **INV-11** | The web `:root` light tokens match `DESIGN-TEMPLATE.html` verbatim (template conformance). | `apps/web/app/globals.css` | `manual` |
| **INV-12** | `.env.example` exists and documents every required env var (no real secrets committed). | `.env.example` | `required-file` |
| **INV-13** | Every SPEC §UI Surface screen route is rendered under `apps/web/app`, and every Key Workflow has an e2e test through the UI. | `apps/web/app/**`, `SPEC.md` | `ui-coverage` |
| **INV-14** | Cross-tenant resource access returns NOT_FOUND (not FORBIDDEN) to avoid existence leak. | `apps/api/src/common/tenant/**` | `manual` |
| **INV-15** | Node serving paths set `--max-http-header-size=32768` (accumulated-cookie resilience). | `package.json` start scripts, `docker-compose.yml` | `forbidden-pattern` (proxy: require flag present via manual) / `manual` |

Total architectural invariants: **15**.
