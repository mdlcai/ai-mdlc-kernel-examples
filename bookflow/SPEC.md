# SPEC.md — BookFlow

> Behavioral contract (source of truth for behavior). Depth: **comprehensive**. Another agent could rebuild the system from this document. Architecture in `ARCHITECTURE.md`; invariants in `invariants.json`.

## 1. Scope & Roles

**In scope:** multi-tenant resource booking (rooms / equipment / services), configurable approval workflows, availability + conflict prevention, audit trail, utilization/approval reporting (CSV/PDF), email notifications (Resend via outbox), inbound delivery webhooks, live updates (SSE).

**Out of scope:** see RESEARCH.md Non-Goals (billing/payments, HR, CRM, project mgmt, inventory beyond reservable resources, asset lifecycle, video/chat, workforce scheduling, ERP). These MUST NOT be implemented.

**Roles (RBAC, scoped within an org):**
| Role | Capabilities |
|---|---|
| `member` | Submit/list/cancel own bookings; browse resources & availability. |
| `approver` | All member caps + decide approvals routed to them. |
| `manager` | Approver caps + configure approval workflow rules; view team dashboards. |
| `ops` | Manager caps + manage resources, view utilization, resolve conflicts. |
| `admin` | Full org control: members/roles, resources, workflows, audit, settings. |

First user to register a new org becomes `admin`. Roles are additive by capability (admin ⊇ ops ⊇ manager ⊇ approver ⊇ member for the cap checks above).

## 2. Data Model (PostgreSQL, all `timestamptz` UTC)

All tenant-owned tables carry `org_id uuid NOT NULL` (FK → organizations) and `created_at`, `updated_at`. `id uuid` PK (default `gen_random_uuid()`).

### organizations
`id, name, slug (unique), settings jsonb, created_at, updated_at`

### users
`id, email (citext, unique), password_hash, name, status ('active'|'disabled'), created_at, updated_at`
(Users are global identities; org membership is via `memberships`.)

### memberships  *(tenant-scoped)*
`id, org_id, user_id, role enum('member','approver','manager','ops','admin'), created_at, updated_at`
Unique `(org_id, user_id)`.

### resources  *(tenant-scoped)*
`id, org_id, name, type enum('room','equipment','service'), description, capacity int null, location text null, attributes jsonb (e.g. {av:true}), requires_resource_ids uuid[] null (dependencies), maintenance_windows jsonb null, min_advance_minutes int default 0, max_duration_minutes int null, is_active bool default true, version int (@VersionColumn), created_at, updated_at`
Index `(org_id, type, is_active)`.

### bookings  *(tenant-scoped)*
`id, org_id, resource_id, requester_id (user), title, starts_at, ends_at, status enum('pending_approval','confirmed','rejected','cancelled'), notes text null, metadata jsonb, version int, created_at, updated_at`
**Constraint (INV-2 / ADR-0004):** `CHECK (ends_at > starts_at)` and
`EXCLUDE USING gist (resource_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&) WHERE (status NOT IN ('cancelled','rejected'))` (requires `btree_gist`).
Indexes: `(org_id, resource_id, starts_at)`, `(org_id, requester_id)`, `(org_id, status)`.

### workflow_rules  *(tenant-scoped)*
`id, org_id, name, resource_type enum|null (null=any), is_active bool, priority int, conditions jsonb (e.g. {min_advance_hours, max_duration_hours, capacity_gt, auto_approve}), approver_role enum('approver','manager','ops','admin'), step_order int default 1, created_at, updated_at`
Determines whether a booking needs approval and who approves. Multiple active rules → ordered by `priority` then `step_order` to build the approval chain. If no rule matches → auto-confirm.

### approvals  *(tenant-scoped)*
`id, org_id, booking_id, step_order int, approver_role enum, assigned_user_id uuid|null, status enum('pending','approved','rejected','changes_requested'), decided_by uuid|null, decision_note text null, decided_at timestamptz|null, version int, created_at, updated_at`
A booking has 1..n approval steps. Booking confirms only when all steps `approved`.

### audit_log  *(tenant-scoped, append-only — INV-8)*
`id, org_id, actor_user_id uuid|null, action text, entity_type text, entity_id uuid|null, before jsonb|null, after jsonb|null, ip text|null, created_at`
No update/delete path. Retained 7 years (no TTL purge).

### outbox  *(tenant-scoped — ADR-0005)*
`id, org_id, type enum('email','webhook'), payload jsonb, status enum('pending','sent','failed','dead'), attempts int default 0, next_attempt_at timestamptz, last_error text null, created_at, sent_at timestamptz|null`
Index `(status, next_attempt_at)`.

### webhook_events  *(idempotency for inbound — INV-4)*
`id, provider text, external_id text (svix-id), event_type text, processed_at timestamptz, payload jsonb`
Unique `(provider, external_id)`.

### refresh_tokens
`id, user_id, token_hash, expires_at, revoked_at timestamptz|null, created_at`

## 3. API Surface (REST, prefix `/v1`, Bearer JWT, `application/json` ↔ `application/problem+json`)

Conventions: list endpoints support `?page&pageSize&sort&q&filter`. All mutating endpoints require auth + RBAC + tenant scope. PATCH = partial (RFC 5789). Versioned via URL path.

### Auth (`/v1/auth`) — public
| Method | Path | Body | Returns | Notes |
|---|---|---|---|---|
| POST | `/register` | `{email,password,name,orgName}` | `201 {accessToken,refreshToken,user,org}` | Creates org + admin membership. **Anti-enumeration:** existing email returns same 201-shape envelope (no distinguishable error); confirmation handled idempotently. |
| POST | `/login` | `{email,password}` | `200 {accessToken,refreshToken,user,memberships}` | Uniform `401 invalid_credentials` for unknown email / wrong password / disabled. Rate-limit fires **before** bcrypt compare (INV-6). Timing padded. |
| POST | `/refresh` | `{refreshToken}` | `200 {accessToken,refreshToken}` | Rotates refresh token; revokes old. |
| POST | `/logout` | `{refreshToken}` | `204` | Revokes refresh token. |
| GET | `/me` | — | `200 {user,memberships,activeOrg,role}` | Requires auth. |

### Orgs (`/v1/orgs`)
| GET | `/current` | — | `200 org` | active org settings |
| PATCH | `/current` | `{name?,settings?}` | `200 org` | admin only |

### Users / Members (`/v1/members`)
| GET | `/` | — | `200 {items,total}` | list members (manager+) |
| POST | `/invite` | `{email,name,role}` | `201 membership` | admin; creates user if new, enqueues invite email |
| PATCH | `/:id` | `{role?,status?}` | `200 membership` | admin; cannot demote last admin |
| DELETE | `/:id` | — | `204` | admin; cannot remove last admin |

### Resources (`/v1/resources`)
| GET | `/` | — | `200 {items,total}` | filter by `type`,`q`,`is_active` |
| GET | `/:id` | — | `200 resource` | 404 if not in org (INV-14) |
| POST | `/` | resource body | `201 resource` | ops+ |
| PATCH | `/:id` | partial | `200 resource` | ops+; optimistic version |
| DELETE | `/:id` | — | `204` | ops+; soft-deactivate if it has future bookings |
| GET | `/:id/availability` | `?from&to` | `200 {busy:[{starts_at,ends_at,status}], maintenance:[...]}` | derived; `no-store` |

### Bookings (`/v1/bookings`)
| GET | `/` | — | `200 {items,total}` | members see own; approver/ops see org-wide (filter `status`,`resource_id`,`from`,`to`,`mine`) |
| GET | `/:id` | — | `200 booking{approvals,resource,requester}` | 404 if cross-tenant |
| POST | `/` | `{resource_id,title,starts_at,ends_at,notes?}` | `201 booking` | member+. Validates: ends>starts, advance/duration rules, dependencies, maintenance windows. DB conflict → `409 booking_conflict`. Triggers workflow eval → approvals + outbox notify. |
| PATCH | `/:id` | `{title?,notes?,starts_at?,ends_at?}` | `200 booking` | requester (own, before confirmed) or ops; time change re-runs conflict + workflow. |
| POST | `/:id/cancel` | `{reason?}` | `200 booking` | requester or ops; sets `cancelled`, frees the slot, notifies. |

### Approvals (`/v1/approvals`)
| GET | `/` | — | `200 {items,total}` | approver+: pending steps routed to my role; filter `status`. Includes booking context. |
| GET | `/:id` | — | `200 approval{booking,requester,resource,history}` | |
| PATCH | `/:id` | `{decision:'approved'|'rejected'|'changes_requested', note?}` | `200 approval` | approver+; version-locked; advances/finalizes booking; outbox notify + SSE + audit. Self-approval of own booking forbidden → `403`. |

### Workflows (`/v1/workflows`)
| GET | `/` | — | `200 {items}` | manager+ |
| POST | `/` | rule body | `201 rule` | manager+ |
| PATCH | `/:id` | partial | `200 rule` | manager+ |
| DELETE | `/:id` | — | `204` | manager+ |

### Audit (`/v1/audit`)
| GET | `/` | `?entity_type&entity_id&actor&from&to&page` | `200 {items,total}` | manager+ |
| GET | `/export.csv` | filters | `200 text/csv` | manager+ |

### Reports (`/v1/reports`)
| GET | `/utilization` | `?from&to&resource_type` | `200 {byResource:[{resource,booked_minutes,available_minutes,pct}],summary}` | ops+ |
| GET | `/approvals` | `?from&to` | `200 {median_cycle_minutes,p95_minutes,sla_breach_count,by_status}` | manager+ |
| GET | `/utilization.csv` | filters | `200 text/csv` | ops+ |
| GET | `/utilization.pdf` | filters | `200 application/pdf` | ops+ |

### Realtime (`/v1/stream`) — SSE
`GET /v1/stream` (Bearer via query/header) → `text/event-stream`; events scoped to the user's active org: `booking.created`, `booking.updated`, `booking.cancelled`, `approval.created`, `approval.decided`. Heartbeat every 25s.

### Webhooks (`/v1/webhooks/resend`) — public, raw body
`POST /v1/webhooks/resend` → verify Svix signature on raw bytes (INV-4) → idempotency insert on `svix-id` → update matching outbox/notification delivery status → `2xx`. Forged/invalid signature → `401`. Replay (seen `svix-id`) → `200` no-op.

### Health (`/api/health`) — public, no `/v1`
`GET /api/health` → `200 {status:'ok', db:'up'|'down', outbox_pending:int, version, uptime_s}`; `503` if db down.

### Error contract (RFC 7807 — INV-7)
All errors: `Content-Type: application/problem+json`, body `{type,title,status,detail,code,errors?}` where `errors` is `{field: message}` for validation (422). Codes: `invalid_credentials`(401), `forbidden`(403), `not_found`(404), `validation_error`(422), `booking_conflict`(409), `rate_limited`(429), `version_conflict`(409).

## 4. Security Requirements (compliance-tagged)

- **SEC-1** `compliance:access-control` — Every `/v1` endpoint except auth/webhooks/health requires a valid JWT (global guard, INV-5). RBAC enforced per endpoint.
- **SEC-2** `compliance:tenant-isolation` — All tenant data filtered by `org_id` via scoped repo (INV-1); cross-tenant reads → 404 (INV-14).
- **SEC-3** `compliance:auth` — Passwords bcrypt (cost ≥12); NIST 800-63B: min length 8, no forced rotation, no composition rules; HIBP screening optional (documented). Anti-enumeration on register/login (shape + timing). Rate-limit before hash compare (INV-6).
- **SEC-4** `compliance:rate-limiting` — Global throttler; stricter named limit on `/auth/*` keyed `(ip,email)`; webhook ingress limited.
- **SEC-5** `compliance:audit-logging` — Append-only audit on every state mutation; who/what/when/org; 7-year retention (INV-8).
- **SEC-6** `compliance:webhook-integrity` — Inbound webhooks verify-then-read + idempotency (INV-4).
- **SEC-7** `compliance:dual-write` — Email/outbound webhook via transactional outbox; no sync send (INV-3).
- **SEC-8** `compliance:secrets` — Secrets via env/vault; `.env.example` only placeholders; `.env` gitignored (INV-12).
- **SEC-9** `compliance:transport` — HTTPS only in prod (forced redirect + HSTS); header buffers sized (INV-15).
- **SEC-10** `compliance:encryption-at-rest` — Postgres volume on encrypted disk (prod); ⚠ delegated to infra layer, documented in COMPLIANCE.
- **SEC-11** OWASP Top 10 — input validation (class-validator), no injection (parameterized TypeORM), CSRF N/A for Bearer API + same-site, security headers (helmet), no secrets in logs.

## 5. Key Workflows (each maps to one e2e test `W<n>` — INV-13)

- **W1 — Auth bootstrap & org creation:** register (new org, becomes admin) → login → `GET /me` returns active org + role. Terminal: lands on dashboard.
- **W2 — Submit a booking (auto-approve path):** member browses resources → checks availability → submits booking for a resource with no matching workflow rule → booking `confirmed` immediately → appears in "My Bookings". Terminal: confirmed booking visible.
- **W3 — Submit a booking requiring approval → approver decides:** member submits booking matching an active workflow rule → status `pending_approval`, approval step created → approver sees it in Approvals queue → approves → booking `confirmed`, requester notified. Terminal: approver completes decision in UI; requester sees confirmed.
- **W4 — Reject with guidance:** approver opens a pending approval → rejects with a note → booking `rejected`, requester sees rejection + guidance note. Terminal: rejection note rendered to requester.
- **W5 — Conflict prevention:** two bookings overlap same resource/time → second is refused with `409 booking_conflict` and a clear UI message offering alternative times. Terminal: conflict surfaced, no double-booking persisted.
- **W6 — Configure approval workflow:** manager creates a workflow rule (e.g. "rooms > 4h need manager approval") → subsequent matching booking routes to that rule. Terminal: rule saved + visibly applied.
- **W7 — Manage resources:** ops creates/edits a resource (room with capacity + AV attribute + maintenance window). Terminal: resource appears in catalog and is bookable.
- **W8 — Utilization report + export:** ops opens utilization report → views per-resource % → exports CSV. Terminal: CSV downloaded with real rows.
- **W9 — Audit trail review:** manager opens audit log → filters by entity → sees who approved/rejected what and when. Terminal: audit entries rendered.

Quick-defer is **not** invoked (depth is comprehensive); all W1–W9 get e2e specs.

## 6. State Transitions (comprehensive)

### Booking status machine
```
(create) ──► pending_approval ──(all steps approved)──► confirmed
        └──► confirmed (no rule matched / auto_approve)
pending_approval ──(any step rejected)──► rejected
pending_approval ──(requester/ops cancel)──► cancelled
confirmed ──(requester before start / ops)──► cancelled
```
- Pre/post: a booking may only be inserted if `ends_at > starts_at` (CHECK) and no overlap on a non-cancelled/rejected row (EXCLUDE). On time-change PATCH, both re-checked; workflow re-evaluated (may revert `confirmed`→`pending_approval` if new time matches a rule — logged + notified).
- Terminal states: `rejected`, `cancelled` (free the slot — excluded from EXCLUDE predicate). `confirmed` is terminal unless cancelled.

### Approval step machine
```
pending ──approve──► approved        (if last step → booking confirmed)
pending ──reject──► rejected         (→ booking rejected, remaining steps voided)
pending ──changes_requested──► changes_requested (booking stays pending; requester notified to edit)
```
Optimistic `@VersionColumn`: stale decision → `409 version_conflict`. Self-approval of own booking → `403 forbidden`.

### Outbox row machine
```
pending ──send ok──► sent
pending ──send fail (attempts<5)──► pending (next_attempt_at = now + backoff)
pending ──send fail (attempts=5)──► dead (DLQ; surfaced in health/admin)
```

### Refresh token
`active ──refresh──► revoked (rotated)`, `active ──logout──► revoked`, expiry → rejected.

## 7. Error Paths & Recovery (per surface)

- **Validation (422):** field-level `errors{field:message}`; UI binds each to its control (no generic "Invalid input").
- **Booking conflict (409 booking_conflict):** UI shows "That slot is taken" + nearest free windows from availability; user retries.
- **Version conflict (409 version_conflict):** UI reloads the entity, shows "changed since you loaded", asks to redo.
- **Auth (401):** uniform message; rate-limited (429) shows retry-after.
- **Forbidden (403):** "You don't have access to this action."
- **Not found (404):** cross-tenant or deleted → "Not found" (no existence leak).
- **Email send failure:** outbox retries with backoff → DLQ; never blocks the user request; admin sees DLQ count.
- **Webhook bad signature (401):** rejected, logged, no state change. Replay → 200 no-op.
- **DB down:** `/api/health` 503; requests fail with 503 problem+json; web shows degraded banner.
- **SSE drop:** client auto-reconnects (EventSource); state reconciled via TanStack Query refetch on reconnect.

## 8. Concurrency / Races / Idempotency

- **Double-booking race:** two concurrent inserts → DB EXCLUDE rejects one (serialized at constraint); app maps to 409. No app-level TOCTOU.
- **Concurrent approval decisions:** optimistic version on `approvals` + `bookings`; second writer gets `version_conflict`.
- **Multi-row locks** (e.g. booking + its approvals): `SELECT ... FOR UPDATE` ordered by `id ASC` to avoid deadlock.
- **Outbox under multi-instance:** poller uses `FOR UPDATE SKIP LOCKED`; idempotent send keyed on outbox id.
- **Webhook idempotency:** unique `(provider, external_id)`; duplicate delivery → no-op.
- **Input validation boundaries:** times must be ISO-8601 with tz; duration within `max_duration_minutes`; advance ≥ `min_advance_minutes`; capacity/dependency checks; pagination `pageSize` 1–100 (default 25).

## 9. Environment & Configuration

> Mandatory section. Every var appears in `.env.example` (INV-12).

| Var | Service | Required | Default / example | Purpose |
|---|---|---|---|---|
| `NODE_ENV` | api,web | yes | `development` | runtime mode |
| `API_PORT` | api | yes | `4000` | API listen port |
| `WEB_PORT` | web | yes | `3000` | web listen port |
| `NODE_OPTIONS` | api,web | yes | `--max-http-header-size=32768` | header-buffer sizing (INV-15) |
| `DATABASE_URL` | api,worker | yes | `postgres://bookflow:bookflow@db:5432/bookflow` | Postgres DSN |
| `JWT_ACCESS_SECRET` | api | yes | `change-me-access` | access token signing |
| `JWT_REFRESH_SECRET` | api | yes | `change-me-refresh` | refresh token signing |
| `JWT_ACCESS_TTL` | api | no | `900` (15m) | access TTL seconds |
| `JWT_REFRESH_TTL` | api | no | `2592000` (30d) | refresh TTL seconds |
| `BCRYPT_COST` | api | no | `12` | bcrypt rounds |
| `RESEND_API_KEY` | worker | no (stub if blank) | `re_REPLACE_ME` | Resend API key; blank → stub transport (ADR-0009) |
| `RESEND_FROM` | worker | no | `BookFlow <noreply@bookflow.local>` | from address |
| `RESEND_WEBHOOK_SECRET` | api | no | `whsec_REPLACE_ME` | Svix verify secret |
| `THROTTLE_TTL` | api | no | `60` | throttle window seconds |
| `THROTTLE_LIMIT` | api | no | `120` | global req/window |
| `AUTH_THROTTLE_LIMIT` | api | no | `8` | auth req/window per (ip,email) |
| `CORS_ORIGIN` | api | yes | `http://localhost:3000` | allowed web origin |
| `NEXT_PUBLIC_API_URL` | web | yes | `http://localhost:4000` | API base for browser |
| `SENTRY_DSN` | api,web | no | (blank) | centralized error tracking (optional) |
| `LOG_LEVEL` | api,worker | no | `info` | pino level |

## 10. UI Surface

> Peer to §3 API surface, precedence-bound. Archetype: **admin** (DESIGN.md Part II) — sidebar+content shell, highest density, dense data tables, guarded destructive actions, near-zero motion. Every screen consumes the Design System tokens below; no screen invents its own.

### Design System (resolved — canonical from DESIGN-TEMPLATE.html)
- **Palette (light, verbatim):** bg `#f4f4f3`, surface `#fff`, surface-2 `#ebebe9`, fg `#111`, muted `#5c5c5c`, border `#dcdcd9`, accent `#000`, accent-soft `#e6e6e3`, success `#1c7c4a`, warning `#9a6400`, error `#b3261e`. **Dark:** derived (ADR-0003) — bg `#0c0c0b`, surface `#161615`, surface-2 `#1e1e1c`, fg `#ececea`, muted `#9a9a96`, border `#2c2c2a`, accent `#fff`(inverted), semantics lightened AA-safe.
- **Type:** display `IBM Plex Sans` (600/700), body `Geist` (400/500), mono `Geist Mono`. Tabular numerals for data/metrics. Tight scale 13px base (admin density). Self-host/preconnect Google Fonts.
- **Spacing unit:** 4px; scale 4/8/12/16/24/32/48/64. **Radius:** sm4/md6/lg10. **Shadow:** minimal (sm/md/lg from template). **Motion:** ≤150ms, feedback-only; `prefers-reduced-motion` still path. Skeletons for loads.
- **Layout:** sidebar (240px) + content; max content 1280px; breakpoints 640/768/1024/1280. Sidebar collapses to top nav < 768px.
- **Signature components:** dense data table (sort, filter, pagination, row actions, bulk where relevant, every state designed), detail/edit drawer, status/severity badges (ok/wait/err styling from template), confirm dialog for destructive actions, global search, metric cards with tabular numerals.

### Screen inventory
| Route | Purpose | States | API binding | Workflow step(s) | RBAC |
|---|---|---|---|---|---|
| `/` | Marketing landing (ported from template: nav, hero w/ app preview, bento features, dark CTA, footer) | static | none | entry → CTA to `/login`/`/register` | public |
| `/login` | Sign in | default, loading, error(field+401), success→redirect | `POST /v1/auth/login` | W1 | public |
| `/register` | Create org + admin | default, loading, validation(field), success | `POST /v1/auth/register` | W1 | public |
| `/app` (dashboard) | Metric cards (pending approvals, my bookings, utilization), recent activity | empty, loading(skeleton), error, success | `/v1/reports/*`, `/v1/bookings?mine`, `/v1/approvals` | W1 terminal, W3/W8 entry | member+ |
| `/app/resources` | Resource catalog table + filters; create/edit drawer | empty("No resources — add one"), loading, error, success | `/v1/resources` CRUD, `/availability` | W7 | browse: member+; edit: ops+ |
| `/app/resources/:id` | Resource detail + availability calendar | empty, loading, error, success | `/v1/resources/:id`, `/availability` | W2 entry, W7 | member+ |
| `/app/bookings` | Bookings table (mine/org), filters, status badges | empty("No bookings yet — book a resource"), loading, error, success | `/v1/bookings` | W2,W4,W5 terminal | member+ |
| `/app/bookings/new` | New booking form (resource, time, conflict + rule preview) | default, validation(field), conflict(409 inline), loading, success | `POST /v1/bookings`, `/availability` | W2,W3,W5 | member+ |
| `/app/bookings/:id` | Booking detail (status, approvals timeline, cancel) | loading, error, success | `/v1/bookings/:id`, `/cancel` | W3,W4 terminal (requester view) | member+ (own) / ops |
| `/app/approvals` | Approval queue (pending routed to me), decision drawer | empty("No approvals pending"), loading, error, success | `/v1/approvals`, `PATCH` | W3,W4 terminal (approver decides) | approver+ |
| `/app/workflows` | Workflow rules table + rule editor | empty("No rules — bookings auto-confirm"), loading, error, success | `/v1/workflows` CRUD | W6 | manager+ |
| `/app/reports` | Utilization + approval-cycle reports, CSV/PDF export buttons | empty(valid-zero: "0 bookings in range"), loading, error, success | `/v1/reports/*` | W8 terminal | ops+ (util), manager+ (approvals) |
| `/app/audit` | Audit log table + filters + CSV export | empty, loading, error, success | `/v1/audit`, `/export.csv` | W9 terminal | manager+ |
| `/app/members` | Members table, invite, role/status edit | empty, loading, error, success | `/v1/members` CRUD | (admin ops) | manager+ view; admin edit |
| `/app/settings` | Org settings, theme toggle (light/dark) | default, loading, success | `/v1/orgs/current` | — | admin |

Every non-internal endpoint binds to ≥1 screen; every Key Workflow W1–W9 completes end-to-end through rendered UI including its terminal step.

## 11. Test Plan (test-after)

- **Unit:** workflow rule matching, booking validation (advance/duration/dependency), availability computation, RBAC cap checks, Svix verify, outbox backoff, problem+json mapping.
- **Integration (supertest + test Postgres):** auth (register/login/anti-enumeration/rate-limit), tenant isolation (org A cannot read org B → 404), booking conflict (concurrent → one 409), approval transitions + version conflict, webhook idempotency.
- **e2e (Playwright, 1 per workflow — INV-13):** `W1`–`W9` through rendered UI to terminal step.
- **Acceptance criteria (perf):** list endpoints < 1s on seed data; health < 200ms; dashboard load < 2s.
