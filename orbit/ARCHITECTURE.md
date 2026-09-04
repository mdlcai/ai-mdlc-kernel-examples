# ARCHITECTURE.md — Orbit IT Command Center

> Stage 1 output of the MDLC™ pipeline. Source of truth for architecture. Behavior is specified in `SPEC.md`; decisions and assumptions in `DECISIONS.md`. Build depth: **comprehensive** (threat model and alternatives-considered per major decision are included).

---

## 1. Context & problem framing

**Problem.** IT organizations run tickets, projects, changes, assets, approvals and documentation in disconnected tools. Nothing links the laptop request to the approval to the asset to the closing ticket, so ownership, SLAs and audit evidence are reconstructed by hand.

**Solution.** Orbit is a multi-tenant web platform where every IT work object — ticket, project/task, change, asset, KB article, approval — lives in one Postgres schema under one org, links to its neighbours, emits an immutable audit event on every mutation, and is pushed live to open browsers.

**Users (personas → roles).**

| Persona | Role | Primary surfaces |
|---|---|---|
| Employee / end user | `requester` | Portal: new request, my tickets, KB search, approvals I must give (if manager) |
| IT technician | `agent` | Queue, ticket detail, changes, assets, projects, KB authoring |
| IT manager / team lead | `manager` | Everything an agent sees + approvals inbox, analytics, SLA policies |
| IT admin / security | `admin` | Everything + users, teams, roles, automation rules, audit trail |

**Key workflows** `W1..W9` and **success metrics** are enumerated in `RESEARCH.md` §"Users & Outcomes" (rewritten at Stage 3b) and traced 1:1 in `SPEC.md` §5.

**Non-goals (v1, ADR-013):** SSO/OAuth/SAML, file attachments and image uploads, outbound Slack/Teams/PagerDuty webhooks, SMS, mobile apps, billing, i18n, offline, discovery agents, inbound email-to-ticket, plugins.

**Scale tier.** `scale: "small — under 1k concurrent"` → leading token `small`. Applied exactly per BUILD.md table: single Postgres with the framework default pool, no Redis, in-process scheduler, single instance per service, per-route rate limiting, smoke + representative load test (comprehensive depth). No domain signal bumps a row.

---

## 2. Technology stack (versions verified against the npm registry 2026-09-02)

| Layer | Choice | Version | Why (alternatives in §12) |
|---|---|---|---|
| Language | TypeScript (strict) | 6.0.3 | ADR-005: TS 7.0.2 unsupported by typescript-eslint 8.69 (`<6.1.0`) |
| Runtime | Node.js | 22 LTS (`node:22-alpine` image; host 22.19.0) | Next 16 requires ≥ 20.9; 22 is active LTS |
| Frontend | Next.js App Router, React | 16.3.4, 19.2.8 | Hard constraint `frontend_framework: Next.js` |
| Styling | Tailwind CSS v4 (`@theme` tokens) + CSS custom properties | 4.3.3 | Token layer in one `:root`; no component library (DESIGN.md §4/§8.3) |
| Fonts | Manrope, Inter, JetBrains Mono as local `.woff2` via `next/font/local` | - | Self-hosted from the repo, no third-party request, no FOUT, ≤ 3 families |
| Backend | Express | 5.2.1 | Hard constraint `backend_framework: Express` |
| Validation | zod | 4.5.4 | Shared schemas in `packages/shared`; env schema; OpenAPI generation |
| ORM / migrations | drizzle-orm / drizzle-kit | 0.45.2 / 0.31.10 | ADR-006 |
| Driver | pg (node-postgres) | 8.23.0 | Pool with `SET LOCAL` per transaction for RLS |
| Database | PostgreSQL | 17 (`postgres:17-alpine`) | Hard constraint; RLS, advisory locks, `SKIP LOCKED`, tsvector search |
| Password hashing | argon2 (Argon2id) | 0.45.1 | OWASP recommendation; prebuilt binaries for win32/linux-musl |
| Logging | pino, pino-http | 10.3.1, 11.0.0 | JSON lines, redaction, `requestId` |
| Rate limiting | express-rate-limit | 8.7.0 | In-memory store (small tier), IPv6-safe key generator |
| Headers | helmet (API) + Next `proxy.ts` (pages) | 8.3.0 | CSP Profile N, HSTS at Caddy |
| Email | nodemailer (SMTP) | 9.1.1 | ADR-010 outbox drain |
| Realtime | Server-Sent Events (native `http`) | — | ADR-004 |
| Edge / TLS | Caddy | 2 (`caddy:2-alpine`) | `tls internal` locally, ACME in production; HSTS; SSE flush |
| Containers | Docker Compose | v2 | Hard constraint |
| Unit / integration tests | vitest, supertest, @vitest/coverage-v8 | 4.1.11, 7.2.2 | 80 % statements threshold on `apps/api/src/{modules,lib}` |
| E2E / audit | @playwright/test, @axe-core/playwright, lighthouse | 1.62.1, 4.13.0, 13.4.1 | `W<n>` specs on Desktop Chrome + Pixel 7; pinned baseline runner |
| Lint / format | eslint 9.39.5 flat config (ADR-024) + typescript-eslint 8.69 + jsx-a11y + react-hooks; prettier 3.9 | — | Verification Gate requires a real linter |
| Load test | autocannon | 8.0.0 | Comprehensive-depth load evidence |

---

## 3. System topology

```
Browser ──HTTPS :8443──▶ Caddy (TLS termination, HSTS, redirect 8080→https)
                          ├── /api/*  ──▶ api  (Express 5, :4000)  ──▶ Postgres 17 (:5432 in-network, :5433 on host)
                          └── /*      ──▶ web  (Next.js 16 standalone, :3000)
                                          (web never proxies API traffic in production; the browser calls /api on the same origin)
migrate (one-shot, drizzle migrations + seed)  ──▶ Postgres   (api depends_on migrate: service_completed_successfully)
```

Compose services: `caddy`, `web`, `api`, `migrate`, `postgres`. Named volumes: `pgdata`, `caddy_data`, `caddy_config`. Every long-running service has a `healthcheck`; `api` waits on `postgres: service_healthy` and `migrate: service_completed_successfully`; `web` and `caddy` wait on `api: service_healthy`.

**Single origin is the load-bearing decision.** Pages and API share `https://localhost:8443`, so the session cookie is first-party (`SameSite=Strict`, `__Host-` prefix), there is no CORS, no preflight, and the CSP `connect-src 'self'` covers the SSE stream. In local development without Caddy, `next.config.ts` `rewrites()` forwards `/api/:path*` to `http://localhost:4000` (JSON only; the SSE stream is disabled in that mode and the UI falls back to 20 s polling — dev convenience, never the deploy).

**Ports (ADR-008).** 8443 https, 8080 http→https redirect only, 5433 Postgres on host (tests). Internal: web 3000, api 4000, postgres 5432.

**Process model.** One Node process per service. `api` hosts the HTTP server, the SSE hub, and the scheduler loop (ADR-009). Both Node services start with `--max-http-header-size=32768` (ADR-017) under `tini`, non-root, multi-stage images.

---

## 4. Data layer

### 4.1 Tenancy model
- **Shared schema, `org_id` on every tenant table, Postgres row-level security enabled and forced** (`ALTER TABLE … ENABLE ROW LEVEL SECURITY; FORCE ROW LEVEL SECURITY`) with policy `USING (org_id = NULLIF(current_setting('app.org_id', true), '')::uuid)` — the `NULLIF` makes a pooled connection whose `SET LOCAL` has expired (reads back as `''`) fail **closed** (research §3.7, pgsql-general).
- The API connects as `orbit_app` (created by migration 0001, `NOBYPASSRLS`, no `SUPERUSER`). Migrations run as the owner role (`POSTGRES_USER`).
- **Canonical scoped wrapper:** `apps/api/src/db/scope.ts` exports `withOrgScope(orgId, fn)` which opens a transaction, runs `SET LOCAL app.org_id = $1` and `SET LOCAL app.user_id = $2`, and passes the transaction handle `tx` to `fn`. All module services receive `tx`, never the root `db`. **INV-1** forbids raw `db.` calls outside `apps/api/src/db/**` and the auth/system modules (which legitimately read across orgs: login by email, health probe).
- **Defense in depth:** service queries *also* filter by `org_id` explicitly (the RLS policy is the backstop, not the only filter), so a mis-set GUC yields zero rows rather than another tenant's rows.
- Users belong to exactly one org in v1 (`users.org_id`). Cross-org lookups of another org's resource return **404** (never 403).

### 4.2 Schema overview (authoritative column-level detail in SPEC.md §2)

| Group | Tables | Notes |
|---|---|---|
| System (no RLS; auth module only) | `orgs`, `users`, `sessions`, `password_reset_tokens` | `users.email` globally unique (citext); `sessions.token_hash` unique (SHA-256 of the opaque cookie token) |
| People | `teams`, `team_members` | |
| Ticketing | `tickets`, `ticket_comments`, `sla_policies`, `ticket_categories` | `UNIQUE(org_id, number)`; per-org sequence via `org_counters` row lock; `version` int for optimistic concurrency; SLA columns (`response_due_at`, `resolution_due_at`, `sla_paused_at`, `sla_paused_ms`, `first_responded_at`, `resolved_at`, `closed_at`, `sla_breached_at`) |
| Approvals | `approvals` | polymorphic `subject_type` ∈ {ticket, change, project, purchase} + `subject_id`; `UNIQUE(org_id, subject_type, subject_id, approver_id)` prevents duplicate pending approvals |
| Change | `changes` | `UNIQUE(org_id, number)`; risk, plan, rollback plan, schedule window, linked ticket |
| Assets | `assets`, `asset_types` | `UNIQUE(org_id, tag)`; `assigned_to_user_id`; assignment history lives in `audit_events` |
| Projects | `projects`, `milestones`, `tasks` | tasks carry `depends_on_task_id` (same project, cycle-checked in service), `status`, `assignee_id`, `due_date` |
| Knowledge | `kb_articles` | `UNIQUE(org_id, slug)`; generated `search_vector tsvector` (GIN) over title+body+tags |
| Automation | `automation_rules`, `automation_runs`, `recurring_tasks` | rule = trigger + conditions JSONB + actions JSONB, evaluated in-process |
| Messaging | `notifications`, `pending_emails` | outbox per BUILD.md `has_email`; `UNIQUE(to_address, template_key, business_ref_id)` |
| Governance | `audit_events` | append-only: RLS grants `INSERT` + `SELECT` only to `orbit_app`; no UPDATE/DELETE policy; **INV-5** forbids `update(auditEvents)` / `delete(auditEvents)` in code |
| Infra | `org_counters`, `scheduler_state` | per-org sequences; last-run bookkeeping |

### 4.3 State machines
Ticket, change, task, approval and project statuses are explicit enums with a transition table in `packages/shared/src/transitions.ts` (`canTransition(kind, from, to, role)`). The service layer rejects any transition not in the table with `409 INVALID_TRANSITION` (SPEC.md §5 lists every transition with pre/post conditions).

### 4.4 Concurrency & idempotency
- **Optimistic concurrency:** every mutable entity has `version int NOT NULL DEFAULT 1`. `PATCH` requires `If-Match: "<version>"` (or body `version`); the UPDATE is `… WHERE id=$1 AND org_id=$2 AND version=$3 RETURNING *`; zero rows → `409 VERSION_CONFLICT` with the current version in `errors[]`.
- **Per-org sequence numbers:** `UPDATE org_counters SET tickets = tickets + 1 WHERE org_id=$1 RETURNING tickets` inside the creating transaction — serialized by the row lock, gap-free enough for humans, never duplicated (backed by `UNIQUE(org_id, number)`).
- **Scheduler single-flight:** `pg_try_advisory_lock(hashtext('orbit:scheduler'))` per tick; the SLA breach sweep uses `SELECT … FOR UPDATE SKIP LOCKED` so overlapping ticks cannot double-escalate; breach is idempotent via `sla_breached_at IS NULL` in the WHERE clause.
- **Outbox drain:** `UPDATE pending_emails SET status='sending', attempts=attempts+1 WHERE id IN (SELECT id … WHERE status='pending' AND next_attempt_at <= now() ORDER BY created_at LIMIT 20 FOR UPDATE SKIP LOCKED) RETURNING *` — the claim is the idempotency guard.
- **Approvals:** decision is `UPDATE approvals SET status=$s … WHERE id=$1 AND status='pending' RETURNING *`; a second decision is a `409 ALREADY_DECIDED`.
- **Multi-row locks in deterministic order:** task dependency updates lock `tasks WHERE id IN (…) ORDER BY id FOR UPDATE`.

### 4.5 Migrations
Drizzle-kit generates `apps/api/drizzle/NNNN_*.sql`; each has a hand-written `NNNN_*.down.sql` (ADR-007). `apps/api/src/db/migrate.ts` applies forward migrations then RLS/role SQL (also a migration), then `seed.ts` inserts reference data (ticket categories, asset types, default SLA policies) and — when `SEED_DEMO=true` — a demo org with an admin. CI runs migrate + seed against an empty database.

---

## 5. API layer (`apps/api`)

### 5.1 Module layout
```
apps/api/src/
  app.ts                    # createApp(): middleware chain and module mounting
  server.ts                 # listen, graceful shutdown
  config/env.ts             # zod env schema, fail-fast (INV-4)
  db/{client,scope,migrate,seed,audit,schema/*}.ts
  http/{problem,requestId,logger,rateLimit,csrf,auth,pagination,validate}.ts
  modules/<name>/{routes,service,load,serialize,index}.ts   # auth, users, teams, tickets, sla, approvals, changes, assets, projects, kb, automation, notifications, events, analytics, audit, system
                            # larger modules split further (tickets adds lifecycle, sla, hooks, notify, categories.*)
  jobs/{scheduler,slaEngine,automationEngine,outbox,recurring}.ts
  lib/{sse,mailer,password,passwordPolicy,cookies,slaClock,emailTemplates}.ts
```
Each module exports an Express `Router` mounted under `/api/v1/<name>`; `system` mounts `/api/health` (unauthenticated) and `/api/openapi.json`.

Three capabilities named in earlier drafts of this diagram do not live in `lib/` and are listed here so the
layout above can be read literally: the HIBP breach check is part of `lib/passwordPolicy.ts`, because it is one
clause of the password rule rather than a service of its own; global search is
`modules/analytics/service.ts`, alongside the other cross-entity queries it shares SQL with; and the transition
table is `packages/shared/src/transitions.ts`, in the shared package because the web app enforces the same table
when deciding which transitions to offer.

### 5.2 Middleware chain (order is normative)
`requestId` → `pino-http` access log → `helmet` header set → `express.json({limit:'256kb'})` → `csrfOriginGuard` (state-mutating methods require `Origin`/`Referer` matching `APP_ORIGIN`) → per-route `rateLimit` → `authenticate` (session cookie → principal) → `requireRole(...)` → `validate(zod)` → `load<Entity>` chokepoint → handler → `problemHandler` (maps every error to the SPEC §7 RFC 9457 envelope; **INV-6** forbids ad-hoc `res.status(4xx).json`).

### 5.3 Authentication
- Register creates `org` + first `admin` user in one transaction; login verifies Argon2id; sessions are opaque 32-byte random tokens, stored hashed, 30-day sliding expiry, rotated on privilege change; logout deletes the row.
- Cookie: `__Host-orbit_session`; `HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=2592000`.
- Anti-enumeration: login returns the same `401 INVALID_CREDENTIALS` for unknown email, wrong password and locked account, with a constant-time dummy hash verify on the unknown-email path; password-reset request always returns `202`.
- Rate limits: `/auth/login` 10 / 15 min per `(ip, email)`, `/auth/register` 5 / h per ip, `/auth/password/reset` 5 / h per `(ip, email)`; the limiter runs **before** the hash verify (**INV-8**). General API: 300 req / min per session; unauthenticated 60 / min per ip.
- Password verifier (ADR-012, NIST SP 800-63B-4): length 15–128, Unicode allowed, no composition rules, no forced rotation, HaveIBeenPwned range check (k-anonymity, 5-char SHA-1 prefix, `Add-Padding`, 3 s timeout, fail-open logged; bundled top-10k list when `HIBP_ENABLED=false`); Argon2id m=19456 KiB, t=2, p=1.

### 5.4 Authorization
- **Org scope** via `withOrgScope` (RLS + explicit filter).
- **Object-level (same-tenant) authorization** via one chokepoint per module: `load<Entity>` middleware (e.g. `loadTicket`) fetches the row inside the org scope and applies the ownership rule inline before the handler runs (there is no shared `assertResourceAccess` helper - each loader states its own rule, because they differ per entity and a single predicate would hide that); `requester` may only see tickets they requested, approvals addressed to them, assets assigned to them, published KB articles; `agent`+ see all org rows. Every `/:id` **and every sub-resource route** (`/:id/comments`, `/:id/timeline`) is registered with the same `load<Entity>` guard — **INV-14** forbids a `/:id…` route whose first guard is not a `load*` middleware.
- **Field-level gating:** staff-only fields (private comments, audit trail, other users' emails) are stripped by the module's `serialize(row, principal)` before `res.json`.

### 5.5 API conventions
`/api/v1` prefix; cursor pagination `?cursor=&limit=` (default 25, max 100, response `{data, nextCursor}`); allow-listed `?sort=field:dir` and filters; `X-Request-Id` echoed; `openapi.json` generated from the zod schemas via `zod-openapi` at build and served at `/api/openapi.json`; `Cache-Control: no-store` on all API responses.

### 5.6 Realtime (SSE)
`GET /api/v1/events/stream` (authenticated) → `text/event-stream` with `Cache-Control: no-store`, `X-Accel-Buffering: no`; the hub (`lib/sse.ts`) writes an initial `: connected` comment immediately (so Caddy flushes headers — research §3.7 caddy#4247), keeps `Map<orgId, Set<client>>`, caps 5 streams per user, emits `event: <entity>.<action>` with `id: <monotonic seq>` and a 20 s `: keepalive` comment; a per-org ring buffer of the last 500 events serves `Last-Event-ID` resume. Every service mutation calls `publish(orgId, event)` after commit. Caddy: `flush_interval -1` and no `encode` on `/api/v1/events/*`.

### 5.7 Service Floor
`GET /api/health` → `{status, version, commit, uptimeSec, checks:{db:{status,latencyMs}, scheduler:{status,lastTickAt}, mailer:{status}}}` 200/503, `Cache-Control: no-store`; env schema fail-fast naming the variable; SIGTERM drain ≤ 10 s; pino JSON with redaction of `password, token, secret, authorization, cookie, set-cookie, email, passwordHash, resetToken`.

---

## 6. Frontend (`apps/web`)

- **Next.js 16 App Router**, `output: 'standalone'`, `proxy.ts` mints the CSP nonce and sets the WEB.md §6 header set on every HTML response (request + response headers, per `reference/security-headers.md`).
- **Route groups:** `(public)` → `/`, `/login`, `/register`, `/forgot-password`, `/reset-password/[token]`, `/kb` (public read of published articles? no — KB requires login; the landing is the only public marketing surface). `(app)` → the authenticated shell: `/dashboard`, `/tickets`, `/tickets/new`, `/tickets/[id]`, `/approvals`, `/changes`, `/changes/[id]`, `/assets`, `/assets/[id]`, `/projects`, `/projects/[id]`, `/kb`, `/kb/[slug]`, `/kb/new`, `/analytics`, `/automation`, `/admin/users`, `/admin/teams`, `/admin/sla`, `/admin/audit`, `/settings/profile`.
- **Data access:** client components call the same-origin API through one typed client (`lib/api.ts`) that uses full literal paths (`/api/v1/tickets`) — never a double prefix — and maps §7 envelopes to `ApiError` with `fieldErrors`. A tiny `useResource` hook (fetch + revalidate on SSE event) replaces a data library; pages render `loading.tsx` skeletons and `error.tsx` boundaries.
- **Shell:** persistent sidebar (224 px, drawer below `md` with a 44 px toggle carrying `aria-expanded` + `aria-controls`, focus-trapped, `Escape` closes), top bar with global search (`⌘K` command palette), theme toggle, notifications bell (SSE-fed).
- **Design tokens** live once in `apps/web/src/styles/tokens.css` (`:root` + `[data-theme="dark"]`) and are re-exported to Tailwind through `@theme`. **INV-7** forbids hex literals in component files. Fonts via fontsource variable packages loaded in `layout.tsx`.
- **Primitives** in `components/ui/*` (Button with `data-primary`, Input/Select/Textarea with `aria-describedby` error wiring, Badge, Card, Table + `TableWrap` (scroll region with `tabindex=0`, `role=region`, `aria-label`), Dialog (`<dialog>` based), Drawer, Toast (`aria-live=polite`), Skeleton, EmptyState, Tabs, Kbd). Feature components compose primitives; no raw framework elements are styled elsewhere.
- **Metadata:** per-route `generateMetadata`, `robots.txt`, `sitemap.xml`, `manifest.webmanifest`, `icon.tsx`, `opengraph-image.tsx`, designed `not-found.tsx`; app routes `robots: noindex`.

---

## 7. Background processing (in-process, ADR-009)

`jobs/scheduler.ts` runs every 15 s under the advisory lock:
1. **SLA engine** — recompute due dates for tickets whose policy/priority changed (event-driven, synchronous in the service), sweep for `response_due_at < now() AND first_responded_at IS NULL AND sla_breached_at IS NULL` (and the resolution equivalent) → set `sla_breached_at`, emit `sla.breached`, run automation rules with trigger `sla.breached`. Business-hours calendars are per SLA policy (`{timezone, days, start, end}`); due-date math is in `lib/slaClock.ts` (pure, unit-tested with DST cases). Pausing: status `waiting_on_requester` stamps `sla_paused_at`; resuming adds elapsed pause to `sla_paused_ms` and shifts due dates.
2. **Automation engine** — evaluates active rules for `ticket.created`, `ticket.updated`, `ticket.transitioned`, `sla.breached`, `approval.decided` and `change.submitted` (event-driven from services), and `schedule` (tick-driven). The event-driven path asks "this happened to one ticket, which rules care?"; the scheduled path is the transpose — "the clock struck, which tickets does this one rule match?" — so it has its own entry point (`runScheduledRule`) that sweeps the org's open tickets, excluding resolved, closed and cancelled ones, and stamps `last_run_at` only after the actions have run (ADR-043). Conditions are a small typed predicate language (`field op value`, `all/any`); actions: `assign_team`, `assign_user`, `set_priority`, `set_status`, `add_tag`, `notify_user/role`, `request_approval`, `create_task`. Each run logs to `automation_runs`; recursion depth capped at 3 to prevent rule loops.
3. **Recurring tasks** — `recurring_tasks WHERE next_run_at <= now()` → create the templated ticket/task, advance `next_run_at` (interval: daily/weekly/monthly/cron subset).
4. **Outbox drain** — see §4.4; Nodemailer SMTP; capped backoff 1 m / 10 m / 1 h then `dlq` with a structured error log.

All four steps are idempotent and safe to skip a tick.

---

## 8. Security architecture & threat model (comprehensive)

### 8.1 Trust boundaries
1. **Internet → Caddy (TLS).** Only 8443 (https) and 8080 (redirect) are exposed. HSTS 2 years, TLS 1.2+.
2. **Caddy → web / api (plain HTTP inside the compose network).** Not reachable from the host except through Caddy; `X-Forwarded-*` trusted only from Caddy (`trust proxy: 1`).
3. **Browser → API (session cookie).** CSRF guarded by `SameSite=Strict` + Origin check; CSP Profile N blocks injected script.
4. **API → Postgres (`orbit_app`, RLS-forced).** A logic bug in one module cannot read another org's rows.
5. **API → SMTP / HIBP (egress).** Fixed hosts from env; no user-controlled URLs are fetched anywhere (no SSRF surface).
6. **Scheduler (system principal).** Runs with `app.org_id` set per org it processes; never bypasses RLS.

### 8.2 STRIDE summary

| Threat | Vector | Control (evidence in SPEC.md §4) |
|---|---|---|
| Spoofing | Credential stuffing, session theft | Argon2id, `(ip,email)` limiter before verify, HIBP screening, HttpOnly/Secure/Strict cookie, hashed session tokens, rotation on privilege change |
| Tampering | Cross-tenant or cross-user writes; concurrent overwrite | RLS + explicit `org_id`; `load*` chokepoint; `If-Match` version; transition table |
| Repudiation | "I never approved that" | Append-only `audit_events` with actor, before/after, `requestId`; approvals record decider + timestamp |
| Information disclosure | IDOR on `/tickets/:id/comments`, private notes to requesters, PII in logs, error leaks | Same guard on sub-resources; `serialize()` field gating; pino redaction; §7 envelope never carries stack traces; cross-org → 404 |
| Denial of service | Login floods, SSE fan-out exhaustion, unbounded lists | Limiters, 5 streams/user + org cap, `limit ≤ 100`, body limit 256 kB, header buffer 32 kB, scheduler lock |
| Elevation of privilege | Requester changes own role; agent approves own change | Role changes admin-only and audited; approver ≠ requester enforced; RBAC matrix tested per endpoint |

### 8.3 Residual risks (tracked)
- At-rest encryption delegated to the deploy volume (⚠ in COMPLIANCE.md).
- Single-instance SSE hub (no cross-instance fan-out) — acceptable at `small`.
- HIBP is fail-open when unreachable (logged) — availability over strictness, documented.

---

## 9. Architectural Invariants

Machine-checkable form: `invariants.json` (pinned runner `scripts/invariant-lint.mjs`). `manual` entries: 2 of 18 (cap = min(3, 20 % of 18 = 3)).

| ID | Rule | Reference | Check type |
|---|---|---|---|
| INV-1 | Tenant-scoped data is accessed only through `withOrgScope` transaction handles; raw `db.*` calls live only in `apps/api/src/db/**`, the auth module and the system module | §4.1, §5.4 | forbidden-pattern |
| INV-2 | Passwords are hashed only with Argon2id; no bcrypt/bcryptjs/md5/sha1 password primitives anywhere | §5.3 | forbidden-pattern |
| INV-3 | `GET /api/health` exists at `apps/api/src/modules/system/health.ts` | §5.7 | required-file |
| INV-4 | Boot-time env schema exists at `apps/api/src/config/env.ts` and `.env.example` is committed | §5.7 | required-file ×2 (INV-4a, INV-4b) |
| INV-5 | `audit_events` is append-only: no `update(auditEvents)` / `delete(auditEvents)` in code | §4.2 | forbidden-pattern |
| INV-6 | Every non-2xx API response goes through `http/problem.ts` (RFC 9457 envelope) — no ad-hoc `res.status(4xx/5xx).json` | §5.2 | forbidden-pattern |
| INV-7 | Design tokens are the only source of color: no hex literals in `apps/web` component/page source | §6 | forbidden-pattern |
| INV-8 | The login route's first handler is `authLimiter`, so the limiter runs before the credential check | §5.3 | forbidden-pattern |
| INV-9 | Every SPEC §6 screen is routable, every non-internal endpoint is referenced by UI code, every `W<n>` has a substantive e2e spec | §6, SPEC §5/§6 | ui-coverage |
| INV-10 | `FRONTEND-AUDIT.json` passes, covers every §6 screen, and is fresh | WEB.md §8 | web-baseline |
| INV-11 | Pinned web-baseline runner present verbatim | WEB.md | required-file |
| INV-12 | Pinned invariant-lint runner present verbatim | BUILD.md appendix | required-file |
| INV-13 | Per-org uniqueness: `tickets(org_id, number)`, `changes(org_id, number)`, `assets(org_id, tag)`, `kb_articles(org_id, slug)` | §4.2 | required-unique-constraint ×4 (INV-13a–d) |
| INV-14 | Every `/:id…` route (including sub-resources) is guarded first by a `load<Entity>` chokepoint (pattern amended per ADR-029) | §5.4 | forbidden-pattern |
| INV-15 | Server code logs only through pino — no `console.*` in `apps/api/src` outside env fail-fast and the migrate/rollback/seed CLIs | §5.7 | forbidden-pattern |
| INV-16 | No secret literals: `SESSION_SECRET=`/`DATABASE_URL=`/`SMTP_URL=` assignments appear only in `.env.example`, compose, CI and docs | §8 | forbidden-pattern |
| INV-17 | The SSE hub enforces authentication, org scoping and the 5-streams-per-user cap on every connection | §5.6 | manual (verify `apps/api/src/lib/sse.ts`) |
| INV-18 | Every tenant table has RLS enabled **and forced** with the `app.org_id` policy, and `orbit_app` is `NOBYPASSRLS`. Outside the tenant set by design: the six system tables, and `pending_emails`, whose `org_id` is nullable because system mail belongs to no org (SPEC.md §2.2) | §4.1 | manual (verify `apps/api/drizzle/0001_rls.sql`) |

Design-token contract (DESIGN.md Part III) is INV-7; required CI gates are the `.github/workflows/ci.yml` sequence (Verification Gate) — checked by the Pre-Delivery gate rather than an invariant.

---

## 10. Dependencies (manifest reconciliation target)

Root workspace: `typescript@6.0.3`, `eslint@9.39.5`, `typescript-eslint@8.69.0`, `eslint-plugin-jsx-a11y@6.10.2`, `eslint-plugin-react-hooks@7.1.1`, `prettier@3.9.6`, `vitest@4.1.11`, `@vitest/coverage-v8@4.1.11`, `@playwright/test@1.62.1`, `@axe-core/playwright@4.13.0`, `lighthouse@13.4.1`, `autocannon@8.0.0`, `tsx@4.23.13`.
`apps/api`: `express@5.2.1`, `drizzle-orm@0.45.2`, `drizzle-kit@0.31.10`, `pg@8.23.0`, `zod@4.5.4`, `pino@10.3.1`, `pino-http@11.0.0`, `helmet@8.3.0`, `express-rate-limit@8.7.0`, `argon2@0.45.1`, `nodemailer`, `zod-openapi`, `supertest@7.2.2`, `@types/express@5.0.6`, `@types/pg@8.23.1`, `@types/supertest@7.2.1`, `@types/node@22.x`.
`apps/web`: `next@16.3.4`, `react@19.2.8`, `react-dom@19.2.8`, `tailwindcss@4.3.3`, `@tailwindcss/postcss@4.3.3`, `@next/eslint-plugin-next@16.3.4`, `markdown-it@14.2.0` + `@types/markdown-it@14.2.0` and `dompurify@3.4.14` (F-17 article rendering: raw HTML disabled at parse, sanitized again in the browser).
`packages/shared`: `zod@4.5.4`.
Any addition is reconciled per feature (banner row "Dep reconciliation") with an ADR.

---

## 11. Implementation notes & risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | Express 5 route syntax (path-to-regexp 8): `*` wildcards and optional segments changed | Only literal and `:param` segments are used; no regex routes |
| R2 | Next 16 `proxy.ts` must set CSP on **request and response** or hydration dies under `strict-dynamic` | Copy the reference pattern; Web Delivery Baseline asserts zero CSP violations |
| R3 | RLS + pooled connections: a leaked `SET` outlives the request | Always `SET LOCAL` inside a transaction (`withOrgScope`); connection reset asserted by an integration test |
| R4 | Local TLS trust for the audit/e2e browsers (`tls internal`) | Export Caddy's root CA to `caddy/root.crt`; `NODE_EXTRA_CA_CERTS` for Node fetch; Chromium trust via the `CACertificates` policy or, failing that, the audit runs inside the Playwright Linux image with the CA installed (ADR at Stage 3) |
| R5 | argon2 native build on Windows dev hosts | Prebuilt binaries ship for win32-x64 and linux-musl; never substitute bcryptjs (substitution discipline) |
| R6 | SSE through Next rewrites buffers in dev | Dev mode polls; production goes through Caddy |
| R7 | TypeScript 7 upgrade path | ADR-005; revisit when typescript-eslint supports it |
| R8 | Scheduler and HTTP in one process compete for the event loop | Ticks are bounded (`LIMIT 50` rows per sweep) and yield between steps |

---

## 12. Alternatives considered (comprehensive depth)

| Decision | Chosen | Alternatives | Why not |
|---|---|---|---|
| Tenancy | Shared schema + RLS | Schema-per-tenant; database-per-tenant | Migration fan-out and connection count explode; RLS gives the same isolation guarantee at `small` scale with one schema |
| Realtime | SSE | WebSockets (ws / Socket.IO); polling only | WS adds Origin-on-upgrade CSRF surface, message-rate budgets and a second auth path for one-directional push; polling alone misses the "real-time visibility" metric |
| Sessions | Opaque server-side sessions | JWT in cookie; JWT in localStorage | Server-side sessions revoke instantly and carry no secret in the token; localStorage tokens are forbidden by WEB.md §6 |
| ORM | Drizzle | Prisma; Kysely; raw SQL | Prisma's engine/binary and generated client complicate the Alpine image and RLS `SET LOCAL`; Kysely lacks migrations tooling; raw SQL loses type inference |
| Jobs | In-process scheduler + advisory lock | BullMQ + Redis; pg-boss; separate worker container | Tier table forbids Redis/queues at `small`; pg-boss is a fine `medium` upgrade |
| Password hashing | Argon2id (native) | bcrypt; scrypt | OWASP first choice; native prebuilds available; bcrypt's 72-byte limit and older design |
| Edge | Caddy | nginx + certbot; Traefik | Caddy's automatic HTTPS and `tls internal` remove the cert workflow entirely; nginx needs explicit header-buffer sizing |
| Frontend data | Hand-rolled `useResource` + SSE invalidation | TanStack Query; SWR | Both are excellent; a 60-line hook keeps the JS budget and avoids a cache layer that SSE already replaces |
| API topology | Browser → same-origin API | Next.js BFF proxying Express; separate API origin + CORS | BFF doubles hops and complicates SSE; separate origin weakens cookie posture (`SameSite=None`) |
| Search | Postgres tsvector | Meilisearch; pg_trgm only | No extra service at `small`; tsvector covers KB + tickets with ranking |
| Component library | None (own primitives on Tailwind) | shadcn/ui; Radix; MUI | DESIGN.md §8.3 forbids stock-library look; own primitives carry the measured a11y contract once |
