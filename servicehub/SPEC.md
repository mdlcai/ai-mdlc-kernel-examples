# SPEC.md — ServiceHub (System Specification)

**Source of truth for behavior.** Architecture in `ARCHITECTURE.md`. Build depth: **comprehensive** — every state transition (pre/post), error path with recovery, race/idempotency guarantee, input validation with boundaries, security boundary per feature, per-endpoint perf. Another AI must be able to recreate the system from this document alone.

Conventions: all API routes are prefixed `/v1`. All errors are `application/problem+json` (RFC 9457, §7). All tenant-scoped reads/writes occur inside the tenant transaction (RLS). Timestamps are UTC ISO-8601. IDs are UUIDv4 unless noted.

---

## 1. Roles & RBAC

| Role | Description | Key capabilities |
|------|-------------|------------------|
| `REQUESTER` | Employee/end-user (self-service). | Create tickets; view/comment/attach on **own** tickets only; cannot see internal notes or other users' tickets. |
| `AGENT` | Help Desk agent. | View/triage team queue; assign self; comment (public+internal); change status; attach; cannot see other tenants. |
| `MANAGER` | IT manager/lead. | All AGENT caps + team-wide queue, SLA-breach dashboard, reports export, reassign across agents. |
| `ADMIN` | Tenant admin. | All MANAGER caps + manage users, categories, SLA policies within the tenant. |

RBAC is enforced by `RolesGuard` (`@Roles()`). **Object-level ownership** is enforced separately by `assertResourceAccess` for `REQUESTER` (own rows only). Org/tenant scope is enforced by RLS for everyone. Internal-only fields (internal comments, audit timeline, other users' PII) are role-gated out of REQUESTER payloads.

---

## 2. Domain rules

### 2.1 Ticket type & priority
- `type` ∈ {`INCIDENT`, `SERVICE_REQUEST`} (ITIL distinction).
- `priority` ∈ {`LOW`, `STANDARD`, `HIGH`, `CRITICAL`}.
- `category` references a tenant `categories` row (e.g. "Software & Licensing", "Hardware", "Network/VPN", "Access/Security").

### 2.2 Ticket status state machine
States: `NEW` → `TRIAGED` → `IN_PROGRESS` → `WAITING` ↔ `IN_PROGRESS` → `RESOLVED` → `CLOSED`. `RESOLVED` → `IN_PROGRESS` (reopen). Any non-terminal → `CANCELLED`.

| From | Allowed To | Who | Side effects |
|------|-----------|-----|--------------|
| NEW | TRIAGED, IN_PROGRESS, CANCELLED | AGENT+ | assign required for IN_PROGRESS; SLA response timer stops on first agent action |
| TRIAGED | IN_PROGRESS, WAITING, CANCELLED | AGENT+ | |
| IN_PROGRESS | WAITING, RESOLVED, CANCELLED | AGENT+ | RESOLVED stops SLA resolution timer; sends requester email |
| WAITING | IN_PROGRESS, RESOLVED, CANCELLED | AGENT+ | WAITING pauses SLA clock (`pause_started_at`) |
| RESOLVED | CLOSED, IN_PROGRESS (reopen) | AGENT+ / REQUESTER may reopen own within 7d | CLOSED is terminal; reopen restarts resolution timer |
| CLOSED | (none) | — | terminal |
| CANCELLED | (none) | — | terminal |

Illegal transitions → `409` problem+json `code=ticket.invalid_transition`. Transition guarded in `TicketsService.transition` with optimistic version check.

### 2.3 SLA model (single server-side evaluator, ADR-05)
Per `priority` (from tenant `sla_policies`, defaults): response & resolution targets.
| Priority | Response target | Resolution target |
|----------|-----------------|-------------------|
| CRITICAL | 15 min | 2 h |
| HIGH | 1 h | 8 h |
| STANDARD | 4 h | 24 h |
| LOW | 8 h | 72 h |

- On create: `response_due_at` = now + response target; `resolution_due_at` = now + resolution target.
- `WAITING` pauses both clocks: `pause_started_at` set; on resume `paused_total_seconds += now - pause_started_at` and due_ats shifted forward.
- Evaluator (BullMQ repeatable, 60s) computes `sla_state` ∈ {`OK`,`AT_RISK` (≥80% elapsed),`BREACHED` (past due)} and emits `sla.breached` events + manager notification + outbox email on first breach. Clients never set SLA fields.

### 2.4 Ticket number
Human-readable `SH-<seq>` per tenant; UNIQUE `(tenant_id, ticket_number)` (INV-4). Allocated via per-tenant sequence row with `UPDATE ... RETURNING` (atomic).

---

## 3. Data Model (DDL-level)

All tenant tables carry `tenant_id uuid NOT NULL` + RLS policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)`. Timestamps `created_at`, `updated_at` default `now()`.

### tenants
`id uuid PK · name text · slug text UNIQUE · created_at`

### users
`id uuid PK · tenant_id FK · email citext · password_hash text · full_name text · role text CHECK in (REQUESTER,AGENT,MANAGER,ADMIN) · is_active bool default true · created_at` — **UNIQUE (tenant_id, email)** (INV-7).

### refresh_tokens
`id uuid PK · tenant_id FK · user_id FK · token_hash text · expires_at · revoked_at null · created_at` — rotation: old token revoked on use.

### categories
`id uuid PK · tenant_id FK · name text · is_active bool` — UNIQUE (tenant_id, name).

### sla_policies
`id uuid PK · tenant_id FK · priority text · response_minutes int · resolution_minutes int` — UNIQUE (tenant_id, priority).

### tickets
`id uuid PK · tenant_id FK · ticket_number text · subject text · body text · type text · priority text · status text · category_id FK · requester_id FK(users) · assignee_id FK(users) null · response_due_at timestamptz · resolution_due_at timestamptz · sla_state text default OK · paused_total_seconds int default 0 · pause_started_at timestamptz null · resolved_at null · closed_at null · version int (@VersionColumn) · created_at · updated_at`
Constraints: **UNIQUE (tenant_id, ticket_number)** (INV-4). Indexes: `(tenant_id,status,priority,resolution_due_at)`, `(tenant_id,requester_id)`, `(tenant_id,assignee_id)`, GIN full-text on `(subject||' '||body)`.

### ticket_comments
`id uuid PK · tenant_id FK · ticket_id FK · author_id FK · body text · is_internal bool default false · created_at` — internal comments never serialized to REQUESTER.

### ticket_attachments
`id uuid PK · tenant_id FK · ticket_id FK · uploader_id FK · filename text · content_type text · size_bytes bigint · storage_key text · scan_status text default PENDING CHECK in (PENDING,CLEAN,INFECTED) · created_at`

### ticket_events (timeline/audit)
`id uuid PK · tenant_id FK · ticket_id FK · actor_id FK null · event_type text · payload jsonb · created_at` — append-only; staff-visibility gated.

### audit_logs
`id uuid PK · tenant_id FK · actor_id FK null · action text · resource_type text · resource_id uuid · ip text · metadata jsonb · created_at` — append-only.

### notifications
`id uuid PK · tenant_id FK · user_id FK · type text · title text · body text · ticket_id FK null · read_at null · created_at`

### pending_emails (outbox)
`id uuid PK · tenant_id FK · to_address text · template_key text · business_ref_id text · payload jsonb · status text default PENDING CHECK in (PENDING,SENT,FAILED,DLQ) · attempts int default 0 · next_attempt_at · last_error text · created_at` — **UNIQUE (to_address, template_key, business_ref_id)** (INV-5).

### pending_webhooks (outbox)
`id uuid PK · tenant_id FK · target_url text · event_type text · payload jsonb · signature text · status text default PENDING · attempts int · next_attempt_at · created_at`

### webhook_events (inbound inbox/dedup)
`id uuid PK · provider text · provider_event_id text · received_at · processed_at null · payload jsonb` — **UNIQUE (provider_event_id)** (INV-6).

---

## 4. Security Requirements (per-feature; COMPLIANCE.md maps each)

| ID | Requirement | Mitigates | Verification |
|----|-------------|-----------|--------------|
| SEC-1 | Tenant isolation: every tenant query inside tenant tx w/ RLS `SET LOCAL`. | R1 | INV-1/3; integration test cross-tenant 404. |
| SEC-2 | Object-level authz: REQUESTER accesses only own tickets + sub-resources; non-owner → 404. | R2/W3 | INV-2; e2e W3. |
| SEC-3 | AuthN on every protected endpoint; CSRF/Origin guard on state-mutating public surface. | — | JwtAuthGuard; supertest 401. |
| SEC-4 | Auth anti-enumeration: uniform response shape + timing on login/register/reset; rate-limit before hash. | R4 | INV-9; test identical envelopes. |
| SEC-5 | Password policy NIST 800-63B: min 8, accept ≥64, no composition/rotation, HIBP breach screening; argon2 hashing. | R4 | INV-13; unit HIBP mock. |
| SEC-6 | Rate limiting on all public endpoints w/ key dims `(ip,email)` for auth; upload frequency cap. | R4 | throttler config test. |
| SEC-7 | Attachment safety: magic-byte + extension allowlist, size+quota, randomized keys, scan_status quarantine, ownership-gated download. | R3 | INV-14; test reject spoofed type. |
| SEC-8 | Inbound webhook: HMAC verify over raw body BEFORE any DB read; reject stale ts; idempotent dedup. | R6 | INV-6/8; test spoof→401, replay→200 no-op. |
| SEC-9 | Outbound delivery: transactional outbox; idempotent (UNIQUE keys); capped backoff → DLQ; never sync send. | R5 | INV-5; test dup enqueue→1 send. |
| SEC-10 | Optimistic locking on ticket edits; 409 on version mismatch. | R7 | test concurrent update→one 409. |
| SEC-11 | Append-only audit + server-authoritative SLA; no client-settable SLA/audit fields. | R8 | whitelist ValidationPipe; test ignored fields. |
| SEC-12 | WebSocket: JWT handshake, Origin check on upgrade, per-room (tenant+ticket) authz on join. | R9 | test unauthorized join rejected. |
| SEC-13 | Secrets only from env (zod-validated); no secrets in repo/logs. | R10 | INV-13; Gitleaks. |
| SEC-14 | Security headers: HSTS, CSP, X-Content-Type-Options, frame-ancestors none (Helmet + Caddy). | — | header assertion test. |

**Compliance-tagged clauses:** `compliance:data_retention` (tickets/events retained indefinitely; no hard delete) · `compliance:audit_logging` (every state mutation + auth event logged) · `compliance:encryption_at_rest` (delegated to infra/managed DB+volume — COMPLIANCE ⚠ delegated, never ✅) · `compliance:soc2` (Security+Confidentiality+Availability controls mapped).

---

## 5. Key Workflows (each → e2e test driving UI to terminal step, INV-15)

**W1 — Requester submits, comments, attaches, tracks to resolution; sees only own tickets.**
Steps: register/login → `/portal/new` submit form (category, subject, body, optional attachment) → 201 w/ ticket number + expected resolution time → `/portal/tickets/:id` shows status timeline (public events only) → add comment → upload attachment → poll/realtime status until RESOLVED. Terminal: requester views RESOLVED ticket and its own attachment in the rendered UI. List view shows only own tickets.

**W2 — Agent picks up, adds internal note, resolves.**
Agent login → `/queue` team queue → open ticket → assign self → add internal comment (`is_internal:true`, hidden from requester) → set status RESOLVED. Terminal: status RESOLVED in agent UI; requester receives resolution email (outbox); requester portal does NOT show the internal note.

**W3 — Requester opening another employee's ticket/attachment is denied (404).**
Requester B authenticated → attempts `GET /v1/tickets/<A's id>` and `GET /v1/tickets/<A's id>/attachments/<id>` → 404. Terminal: UI shows a "Not found" state (not a stack trace) when navigating to a foreign ticket URL.

**W4 — Manager reviews team queue and SLA breaches across the org.**
Manager login → `/dashboard` metric cards (open count, AT_RISK, BREACHED, MTTR) → `/queue?slaBreached=true` filtered table → open a breached ticket → export report (CSV/PDF). Terminal: manager views SLA-breach dashboard + downloads an export artifact through the UI.

---

## 6. API Surface (`/v1`)

Auth (public, rate-limited `(ip,email)`):
- `POST /v1/auth/register` → 201 `{user}` (anti-enum: same shape if email exists). Body: `{email,password,fullName,tenantSlug}`.
- `POST /v1/auth/login` → 200 set refresh cookie + `{accessToken,user}`; uniform 401 envelope on any failure.
- `POST /v1/auth/refresh` → 200 rotate; revoked/expired → 401.
- `POST /v1/auth/logout` → 204 revoke refresh.
- `POST /v1/auth/forgot-password` → 202 (always, anti-enum). `POST /v1/auth/reset-password` → 204.
- `GET /v1/auth/me` → 200 `{user}` (auth).

Tickets (auth):
- `POST /v1/tickets` (REQUESTER+) → 201 `{ticket}`. Body validated; `tenant_id/requester_id/status/sla` server-set (whitelist strips client attempts, SEC-11).
- `GET /v1/tickets` → 200 paginated. Query: `scope` (`mine`|`team`), `status`, `priority`, `slaBreached`, `q` (search), `page`, `pageSize`. REQUESTER forced `scope=mine`.
- `GET /v1/tickets/:id` → 200 `{ticket, events(public unless staff)}`; non-owner REQUESTER → 404 (SEC-2).
- `PATCH /v1/tickets/:id` (AGENT+) → 200; requires `If-Match` version or body `version`; mismatch → 409 (SEC-10).
- `PATCH /v1/tickets/:id/assign` (AGENT+) → 200.
- `PATCH /v1/tickets/:id/status` → 200 or 409 invalid transition.
- `GET /v1/tickets/:id/comments` → 200 (internal filtered for REQUESTER). `POST /v1/tickets/:id/comments` → 201 (`is_internal` only honored for AGENT+).
- `GET /v1/tickets/:id/attachments` → 200 list. `POST /v1/tickets/:id/attachments` (multipart) → 201. `GET /v1/attachments/:id/download` → 200 stream (ownership-checked, SEC-7; child-by-id route also gated, SEC-2).
- `GET /v1/tickets/:id/timeline` → 200 (staff-gated fields).

Catalog & admin (auth):
- `GET /v1/categories` → 200. `GET /v1/sla-policies` (MANAGER+) → 200.
- `GET/POST/PATCH /v1/users` (ADMIN) → user mgmt.

Dashboard/reports (MANAGER+):
- `GET /v1/dashboard/metrics` → 200 `{open, atRisk, breached, mttrHours, byCategory[]}`.
- `POST /v1/reports/export` → 202 `{jobId}`; `GET /v1/reports/:jobId` → 200 status/artifact URL. Formats PDF/CSV/JSON.

Notifications (auth): `GET /v1/notifications` → 200; `PATCH /v1/notifications/:id/read` → 204.

Webhooks (public, signature-gated): `POST /v1/webhooks/resend` → 200 (verify→dedup→process, SEC-8). Internal — no UI binding.

Health/ops (public): `GET /api/health` → 200 `{status, db, redis, version}`. `GET /metrics` → Prometheus text.

Realtime: socket.io namespace `/events`; rooms `tenant:<id>`, `ticket:<id>`; events `ticket.created|updated|comment.added|sla.breached`.

---

## 7. Error Contract (RFC 9457, ADR-07)

All errors: `application/problem+json`:
```json
{ "type":"https://servicehub.app/problems/<code>", "title":"<human>", "status":<int>,
  "detail":"<specific, recoverable>", "instance":"/v1/...", "code":"<machine-code>",
  "errors": { "<field>": "<message>" } }
```
Catalog (codes): `validation.failed` (400 + per-field `errors`), `auth.invalid_credentials` (401 uniform), `auth.unauthorized` (401), `authz.forbidden` (403, role), `resource.not_found` (404; also for non-owner IDOR), `ticket.invalid_transition` (409), `resource.version_conflict` (409), `rate_limit.exceeded` (429 + `Retry-After`), `upload.unsupported_type`/`upload.too_large` (400/413), `webhook.invalid_signature` (401), `server.error` (500, no stack leak). Frontend maps `errors{}` onto offending form controls (no top-level-only collapse).

---

## 8. UI Surface (mandatory; saas archetype — persistent app shell, designed states)

Global: app shell = sidebar nav (224px) + contextual header (title, breadcrumb, primary action) + content; light+dark theme toggle (system-aware, persisted); tokens from `globals.css` (template `:root`). Every screen specifies empty/loading/error/success.

| # | Route | Purpose | Key components & states | API bound | Workflow | RBAC |
|---|-------|---------|-------------------------|-----------|----------|------|
| S1 | `/` | Marketing landing (ported from template). | Nav, hero+app-mockup, trust strip, feature cards, CTA, footer. | — | entry | public |
| S2 | `/login` | Sign in. | Form (email/pwd), inline field errors (consume `errors{}`), loading on submit, uniform error, success→redirect. | `POST /auth/login` | W1–W4 entry | public |
| S3 | `/register` | Sign up. | Form, HIBP-weak-password message, anti-enum success. | `POST /auth/register` | W1 | public |
| S4 | `/forgot-password`, `/reset-password` | Recovery. | Form; always-success messaging (anti-enum); token form. | auth reset | — | public |
| S5 | `/portal` | Requester: my tickets. | Data table (status/priority badges, SLA chip), empty ("No tickets yet — submit your first request"), loading skeleton, error, **only own tickets**. | `GET /tickets?scope=mine` | W1 | REQUESTER+ |
| S6 | `/portal/new` | Submit request. | Form (category select, type, subject, body, attachment dropzone w/ type/size validation), field errors, submitting state, success→ticket number + expected resolution. | `POST /tickets`, `POST .../attachments` | W1 | REQUESTER+ |
| S7 | `/portal/tickets/:id` | Requester ticket detail. | Status timeline (public events only), public comments, add-comment box, attachment list/upload, **no internal notes**; not-found state (W3); realtime updates. | `GET /tickets/:id`, comments, attachments | W1, W3 | REQUESTER (own) |
| S8 | `/queue` | Agent/manager team queue. | Data table: sort, filter (status/priority/`slaBreached`), search, pagination, bulk select, row actions (assign/open), empty/loading/error, SLA AT_RISK/BREACHED badges. | `GET /tickets?scope=team` | W2, W4 | AGENT+ |
| S9 | `/queue/:id` (or drawer) | Agent ticket detail/triage. | Full timeline incl. internal, public+internal comment composer (internal toggle), assign control, status dropdown (state-machine-aware, disabled illegal), attachments, version-conflict 409 surfaced, optimistic update. | tickets/comments/attachments | W2 | AGENT+ |
| S10 | `/dashboard` | Manager metrics. | Lead metric (open) + supporting (AT_RISK, BREACHED, MTTR) metric cards w/ trend, by-category chart (legend/empty/loading), SLA-breach quick link. | `GET /dashboard/metrics` | W4 | MANAGER+ |
| S11 | `/reports` | Export. | Format select (PDF/CSV/JSON), filters, generate button → async job progress → download link; empty/error. | `POST /reports/export`, `GET /reports/:id` | W4 | MANAGER+ |
| S12 | `/admin/users`, `/admin/categories`, `/admin/sla` | Tenant admin. | User table + create/edit; category mgmt; SLA policy editor. | users/categories/sla | — | ADMIN |
| S13 | `/notifications` (panel) | In-app notifications. | List, unread badge, mark-read; empty state. | notifications | W4 | auth |

Every non-internal endpoint has a UI affordance; every Key Workflow completes through the UI incl. terminal steps (S6 submit, S9 resolve, S7 not-found, S11 export download).

---

## 9. Performance Acceptance Criteria (from Success Metrics)
- Ticket submission p95 < 2s (synchronous path excludes email send — outbox async).
- Dashboard load p95 < 1.5s (cached SLA policy/category lookups; pre-aggregated metrics query).
- Search results p95 < 500ms (GIN full-text index).
- SLA compliance ≥95% measurement supported by server-side evaluator + breach reporting.
- MTTR tracked via `resolved_at - created_at` in dashboard.
- Comprehensive: load test (k6/autocannon) against ticket list + submit at medium-tier concurrency.

---

## 10. Environment & Configuration (mandatory)

All config via env vars (zod-validated at boot; fail-fast). `.env.example` lists every var.

| Var | Service | Required | Description / placeholder |
|-----|---------|----------|---------------------------|
| `NODE_ENV` | api/web | yes | `production`/`development` |
| `DATABASE_URL` | api | yes | `postgres://servicehub:CHANGE_ME@postgres:5432/servicehub` |
| `REDIS_URL` | api/worker | yes | `redis://redis:6379` |
| `JWT_ACCESS_SECRET` | api | yes | 32+ byte secret (placeholder `CHANGE_ME_ACCESS`) |
| `JWT_REFRESH_SECRET` | api | yes | 32+ byte secret (placeholder `CHANGE_ME_REFRESH`) |
| `JWT_ACCESS_TTL` | api | no | default `900s` |
| `JWT_REFRESH_TTL` | api | no | default `30d` |
| `RESEND_API_KEY` | api/worker | no (local: stub) | `re_REPLACE_ME` — when unset, email driver = console/log stub |
| `RESEND_WEBHOOK_SECRET` | api | no | `whsec_REPLACE_ME` |
| `EMAIL_FROM` | api/worker | no | `ServiceHub <noreply@servicehub.app>` |
| `STORAGE_DRIVER` | api | no | `local` (default) or `s3` |
| `STORAGE_LOCAL_DIR` | api | no | `/var/servicehub/uploads` (outside webroot) |
| `S3_*` | api | no | endpoint/bucket/keys when `STORAGE_DRIVER=s3` (placeholders) |
| `MAX_UPLOAD_BYTES` | api | no | default `10485760` (10MB) |
| `CORS_ORIGINS` | api | yes | `https://servicehub.localhost` |
| `APP_BASE_URL` | web | yes | `https://servicehub.localhost` |
| `NEXT_PUBLIC_API_URL` | web | yes | `https://servicehub.localhost/v1` |
| `SENTRY_DSN` | api/web | no | error reporting (env-gated) |
| `HIBP_ENABLED` | api | no | default `true`; offline → graceful skip + log |

Header buffers: Caddy generous defaults documented; Node `--max-http-header-size=32768` in api start command; nginx not used (Caddy edge).

TLS / Protocol (`protocol_support: HTTPS only`): Caddy serves HTTPS, internal local TLS via Caddy's local CA (or mkcert) for `servicehub.localhost`; forces HTTP→HTTPS redirect; HSTS enabled. Production: managed cert / Let's Encrypt via Caddy automatic HTTPS. Smoke test runs against the `https://` (or local `http://localhost` mapped behind the proxy) URL that the deploy resolves.

---

## 11. Testing Plan (TDD)
- **Unit:** services (SLA math incl. pause/resume, state-machine guards, password policy/HIBP, signature verify, outbox idempotency).
- **Integration (supertest + test DB):** each endpoint happy + error paths; cross-tenant 404; same-tenant cross-user 404 (W3); version-conflict 409; webhook spoof 401 + replay no-op; rate-limit 429; whitelist strips SLA fields.
- **e2e (Playwright):** one spec per Key Workflow W1–W4 driving the rendered UI to its terminal step (INV-15, ui-coverage).
- **Security tests** per feature (IDOR, enumeration, upload spoof).
- Regression anchor: full suite run before/after each feature; no pass-count decrease.
