# ARCHITECTURE.md — SmarCoders

> Medical coding workflow & QA training platform on **synthetic** (Synthea) data.
> Source of truth for **architecture**. Behavior lives in `SPEC.md`. Deviations are logged in `DECISIONS.md`.
> Build depth: **comprehensive**. Archetype: **saas**. Protocol: **HTTPS only**. Security baseline: **HIPAA**.

---

## 1. System Overview

SmarCoders is a containerized monorepo deployed as a local-first Docker stack (cloud-portable). It lets **coders** code synthetic patient charts (ICD-10-CM diagnoses + procedures), the system **validates** codes against format/sequencing/comorbidity rules, **supervisors audit** coded charts and return structured feedback, and **managers** watch a real-time queue/metrics dashboard. All clinical data is Synthea-synthetic; the platform nonetheless enforces HIPAA-grade access control, audit logging, anti-enumeration auth, rate limiting, and encryption-at-rest posture for PHI-equivalent handling.

### Component map

```
┌──────────────────────────────────────────────────────────────────┐
│  Browser (Next.js 15 App Router · shadcn/ui · TanStack Query v5)   │
│   /login /queue /charts/:id /audit /audit/:id /metrics /imports    │
└───────────────┬───────────────────────────────────────────────────┘
                │ HTTPS (REST + SSE)   httpOnly cookie session (JWT)
        ┌───────▼─────────────────────────────────────────┐
        │  Nginx reverse proxy (TLS term, HSTS, HTTP→HTTPS │
        │  redirect, header-buffer sizing)                 │
        └───────┬─────────────────────────┬────────────────┘
                │ /api/*                   │ /  (static/SSR)
        ┌───────▼──────────────┐   ┌───────▼─────────┐
        │  NestJS 11 API        │   │  Next.js server │
        │  Modules:             │   └─────────────────┘
        │  auth, users, charts, │
        │  coding, validation,  │   ┌─────────────────┐
        │  audit, queue, metrics│──▶│  Redis (BullMQ) │
        │  imports, webhooks,    │   └───────┬─────────┘
        │  audit-log, outbox,   │           │ workers
        │  health, notifications│   ┌───────▼─────────┐
        └───────┬───────────────┘   │ Worker process  │
                │ pg (parameterized) │ import, outbox,  │
        ┌───────▼───────────────┐   │ email, metrics   │
        │  PostgreSQL 17         │◀──┘ relay           │
        │  (encrypted volume)    │   └─────────────────┘
        └────────────────────────┘
   External (placeholder-gated): Resend · Datadog · Sentry · AWS Secrets Manager
```

### Repository layout (monorepo, pnpm workspaces)

```
SmartCoders/
├── apps/
│   ├── api/                 # NestJS 11 backend (REST + SSE + worker)
│   │   ├── src/
│   │   │   ├── common/      # guards, interceptors, filters, decorators
│   │   │   ├── server/      # data-access layer (the ONLY place that touches pg)
│   │   │   │   ├── db/      # pool, migrations runner, query helpers
│   │   │   │   └── scope/   # withUserScope / withOrgScope wrappers
│   │   │   ├── modules/     # auth, users, charts, coding, validation,
│   │   │   │                #   audit, queue, metrics, imports, webhooks,
│   │   │   │                #   notifications, health, audit-log
│   │   │   ├── jobs/        # BullMQ processors (import, outbox-relay, email, metrics)
│   │   │   └── main.ts
│   │   └── test/            # jest unit/integration + supertest
│   └── web/                 # Next.js 15 App Router frontend
│       ├── app/            # routes (login, queue, charts, audit, metrics, imports)
│       ├── components/     # shadcn/ui-derived + app components
│       ├── lib/            # api client, query hooks, tokens
│       └── e2e/            # Playwright (one spec per SPEC §5 workflow)
├── packages/
│   ├── shared/             # zod schemas, error codes, DTO types, ICD-10 ruleset
│   └── tokens/             # design tokens (copied verbatim from DESIGN-TEMPLATE.html)
├── db/migrations/          # SQL migrations (numbered, forward-only)
├── scripts/                # invariant-lint.mjs, seed-synthea.mjs, smoke-test.sh
├── infra/                  # nginx.conf, Dockerfiles
├── .claude/                # CLAUDE handoff scaffold (commands, agents, settings)
├── docker-compose.yml
├── invariants.json
└── *.md (governance docs)
```

---

## 2. Layers & Data Flow

1. **Presentation** (`apps/web`): Next.js App Router. Server Components for first paint of queue/metrics; Client Components + TanStack Query for live data. SSE subscription for queue dashboard refresh (`< 1s`). All design values come from `packages/tokens` (verbatim DESIGN-TEMPLATE tokens).
2. **API** (`apps/api/src/modules`): NestJS controllers → services. Controllers are thin; services hold business logic and call the **data-access layer only**. Cross-cutting concerns via Nest interceptors/guards: `AuthGuard` (JWT cookie), `RolesGuard` (RBAC), `RateLimitGuard`, `AuditInterceptor` (writes audit-log rows), `SentryGlobalFilter`.
3. **Data access** (`apps/api/src/server`): the *only* layer permitted to import the `pg` pool. Every query touching tenant/user-scoped tables goes through `withOrgScope(orgId, fn)` / `withUserScope(userId, fn)`. This is the canonical scoped wrapper (INV-1).
4. **Persistence**: PostgreSQL 17. Forward-only SQL migrations in `db/migrations`. JSONB for stored FHIR chart payloads.
5. **Async / jobs**: BullMQ on Redis. Processors: `import` (parse Synthea bundle → charts), `outbox-relay` (publish domain events), `email` (drain `pending_emails`), `metrics` (recompute rollups).
6. **Reliability patterns**:
   - **Transactional outbox** (`has_dual_write`): a domain mutation and its `outbox_events` row are written in one DB transaction; the `outbox-relay` worker publishes at-least-once; consumers are idempotent (INV-5).
   - **Webhook verify-first** (`has_webhooks`): inbound webhooks verify HMAC-SHA256 over the raw body **before** any DB read, then dedupe via `processed_events` UNIQUE(event_id) (INV-4, INV-6).
   - **Email outbox** (`has_email` via guaranteed-delivery): never sync-send in a request handler; insert into `pending_emails` in the triggering txn, drain with capped backoff → DLQ.

---

## 3. Agent & Tool Orchestration

No LLM agents at runtime. "Orchestration" here = the BullMQ worker topology:
- **API process** enqueues jobs; never blocks a request on email/import.
- **Worker process** (separate container, same image) runs all processors + the outbox/email relays on a fixed interval.
- **SSE** streams queue counters from API (backed by a short-TTL Redis cache refreshed by the `metrics` job).

---

## 4. Design Patterns

- **Module-per-bounded-context** (NestJS feature modules).
- **Repository/data-access isolation** — pg access funneled through `server/db` + scope wrappers.
- **Default-deny RBAC** — `RolesGuard` denies unless a role is explicitly allowed; tenant check always applied.
- **Structured error contract** — every error is `{ error: { code, message, fields? } }` (typed codes in `packages/shared/errors`). UI maps `fields` to inputs (SPEC §7).
- **Outbox + idempotent consumer** for dual-write.
- **Append-only hash-chained audit log** — each `audit_log` row stores `prev_hash` + `row_hash` for tamper-evidence.
- **Anti-enumeration auth** — uniform response shape + timing on login/signup; rate-limit fires before password hash compare.

---

## 5. Dependencies (libraries / APIs / versions)

| Dependency | Version (pinned) | Role |
|---|---|---|
| Node.js | 22 LTS | runtime |
| Next.js | 15.x | frontend |
| React | 19.x | UI |
| @tanstack/react-query | 5.x | data fetching |
| Tailwind CSS | 3.4.x | styling (token-bound) |
| NestJS | 11.x | backend framework |
| @nestjs/bullmq + bullmq | 11.x / 5.x | jobs |
| pg | 8.x | Postgres driver (raw, parameterized) |
| ioredis | 5.x | Redis client |
| zod | 3.x | validation (shared schemas) |
| argon2 | 0.41.x | password hashing (native; never substitute) |
| jose | 5.x | JWT sign/verify |
| pino | 9.x | structured JSON logging |
| resend | current | email (placeholder-gated) |
| dd-trace | 5.x | Datadog APM (init-first; placeholder-gated) |
| @sentry/nestjs / @sentry/nextjs | 10.x | error reporting (placeholder-gated) |
| @aws-sdk/client-secrets-manager | 3.x | secrets (falls back to env in local) |
| PostgreSQL | 17.x | database |
| Redis | 7.x | queue/cache |
| Nginx | 1.27.x | TLS termination / reverse proxy |
| Playwright | 1.x | e2e (one spec per §5 workflow) |

External SaaS (Resend, Datadog, Sentry, AWS Secrets Manager) are **placeholder-gated**: absent real keys, the app degrades gracefully (email → outbox stays queued; secrets → env fallback; telemetry → no-op) and exposes contract-correct behavior. Logged in DECISIONS.md.

---

## 6. Key Decisions & Alternatives (see DECISIONS.md for ADRs)

| Decision | Choice | Alternative considered | Why |
|---|---|---|---|
| Code set in v1 | ICD-10-CM only (+ procedure by code) | Full CPT descriptors | CPT is AMA license-restricted (ADR-002) |
| DB access | Raw `pg` + parameterized queries + scope wrappers | Prisma/TypeORM ORM | Tight control over scoping invariant + simpler migration story; ADR-003 |
| Real-time | SSE | WebSocket | One-way job-status push; simpler; ADR-004 |
| Dual-write | Transactional outbox + polling relay | Debezium CDC | No extra infra for a medium-scale app; ADR-005 |
| Password hash | argon2id (native) | bcryptjs | Native primitive, no substitution; ADR-006 |
| Sessions | httpOnly cookie JWT (jose) | Header bearer tokens | CSRF-guarded cookie avoids JS token theft; ADR-007 |
| Deploy | Local Docker stack (cloud-portable) | Direct cloud | Externals unprovisioned at build time; ADR-008 |

---

## 7. Implementation Notes & Risks

- **CPT licensing** gated to v1 ICD-10-CM-only scope (RESEARCH §6).
- **Audit-log integrity** is HIPAA-load-bearing; hash-chain verified by a job + an invariant (manual).
- **Header-buffer sizing**: nginx `client_header_buffer_size 16k` + `large_client_header_buffers 8 32k`; Node `--max-http-header-size=32768` — foreign accumulated cookies must not 400/431 (INV-13).
- **Encryption at rest**: Postgres data volume documented as encrypted (managed disk / LUKS); secrets via Secrets Manager. Honestly gated as delegated where the platform provides it.

---

## 8. Environment & Configuration (summary; full matrix in SPEC §"Environment & Configuration")

Single `.env` (local) → AWS Secrets Manager (prod). Required: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `SESSION_COOKIE_NAME`, `APP_URL`, `WEBHOOK_SIGNING_SECRET`. Optional/placeholder-gated: `RESEND_API_KEY`, `EMAIL_FROM`, `SENTRY_DSN`, `DD_*`, `AWS_*`. Every var enumerated in `.env.example` with owning service.

---

## 9. Architectural Invariants

> The source for the Stage 4 invariant lint pass and the ship-time Drift Detection Gate.
> Each rule has a corresponding entry in `invariants.json`. Machine-checkable types are enforced by `scripts/invariant-lint.mjs`; `manual` rules are audited at the Drift Detection Gate.

| ID | Rule | File / Section | Check type |
|----|------|----------------|------------|
| **INV-1** | No raw pg pool construction/use (`getPool(`/`new Pool(`/`pool.query(`) outside `apps/api/src/server/**`. Modules access data only via server-layer helpers (`query`/`queryOne`/`withOrgScope`) and the transaction client handed to them by server-layer `withTransaction` — that client is the sanctioned unit-of-work API (ADR-013). | §2.3 | forbidden-pattern |
| **INV-2** | A design-token file exists and is the single source of color/type/spacing values (copied verbatim from DESIGN-TEMPLATE). | `packages/tokens/tokens.css` | required-file |
| **INV-3** | `users` table enforces unique email (case-insensitive) to prevent duplicate-account enumeration. | §10 data model | required-unique-constraint |
| **INV-4** | Inbound webhook `processed_events` has a UNIQUE(event_id) so replays are idempotent. | §2.6 | required-unique-constraint |
| **INV-5** | `outbox_events` table exists (transactional outbox for dual-write). | §2.6 | required-file (migration) → modeled as required-unique-constraint on (id) |
| **INV-6** | In every webhook controller, signature verification precedes any DB read (`verifySignature` before `processed_events` lookup / service call). | `apps/api/src/modules/webhooks/**` | boundary-order |
| **INV-7** | Auth rate-limit guard precedes the password-hash compare in the auth service login path. | `apps/api/src/modules/auth/auth.service.ts` | boundary-order |
| **INV-8** | No hard-coded secrets / placeholder credentials committed in source (`sk_live`, real keys). | repo (outside `.env.example`, tests) | forbidden-pattern |
| **INV-9** | `.env.example` exists and enumerates every env var. | `.env.example` | required-file |
| **INV-10** | UI coverage: every SPEC §UI Surface screen route renders, every non-internal endpoint is referenced by UI source, every §5 workflow has an e2e spec driving it through the UI to its terminal step. | `apps/web/**`, `SPEC.md` | ui-coverage |
| **INV-11** | Health endpoint exists and reports dependency status. | `apps/api/src/modules/health/**` | required-file |
| **INV-12** | Audit log is append-only & hash-chained; no `UPDATE`/`DELETE` on `audit_log` anywhere in source. | `apps/api/src/**` | forbidden-pattern |
| **INV-13** | Reverse proxy + Node header-buffer sizing present (nginx large_client_header_buffers / Node max-http-header-size). | `infra/nginx.conf` | manual |
| **INV-14** | argon2 (not bcryptjs/argon2-browser) is the password hash primitive; no substitution. | `apps/api/src/modules/auth/**` | forbidden-pattern |

Invariant count: **14** (12 machine-checkable, 2 effectively manual/boundary-audited).

---

## 10. Data Model (overview; authoritative DDL in `db/migrations`, behavior in SPEC §6)

`organizations`, `users` (role: manager|supervisor|coder), `charts` (synthetic encounter, JSONB payload, specialty, difficulty, status, assignee), `chart_codes` (assigned ICD-10 dx/proc codes + rationale + sequence + is_principal), `validation_results`, `audits` (supervisor review, score, decision), `audit_findings`, `feedback`, `audit_log` (append-only, hash-chained), `outbox_events`, `processed_events`, `pending_emails`, `import_jobs`, `code_reference` (ICD-10-CM seed). FKs scoped by `org_id` throughout.

---

## 11. Threat Model (comprehensive — see RESEARCH §5 STRIDE list)

Boundaries: Auth (spoof/repudiation → anti-enum + rate-limit-before-hash + audit), Chart access (EoP/disclosure → default-deny tenant+role guards + isolation tests), Audit log (tampering → append-only hash-chain), Webhook ingress (spoof/replay → verify-first + idempotency), Outbox (consistency → single-txn + idempotent consumers), Secrets/at-rest (disclosure → Secrets Manager/KMS + TLS-only + encrypted volume).
