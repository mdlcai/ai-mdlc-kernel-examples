# SPEC.md — ServiceHub (behavioral contract)

> Source of truth for **behavior**. Build depth: **comprehensive** — every state
> transition, error path, race condition, input boundary, and trust boundary is
> specified. Another AI must be able to recreate the system from this document.
> Precedence: RESEARCH > ARCHITECTURE > SPEC.

## 0. Glossary & Roles

**Roles (RBAC):**
- `end_user` — submits/views own tickets, comments, approves own pending approvals.
- `technician` — works tickets in their teams' queues, comments, assigns self,
  resolves; cannot delete tickets or manage users.
- `manager` — technician powers + reassign/escalate, view manager dashboard,
  manage team membership and SLA policies, run reports.
- `security_officer` — manager powers within scope + approve/deny access-request
  approval chains, view audit log.
- `admin` — full org administration: users, roles, categories, webhook endpoints,
  integrations, retention settings; executive dashboard.

Permissions are deny-by-default; every protected route declares a required
permission. RBAC is enforced server-side per route AND per org (tenant scope).

**Ticket statuses (state machine):**
`new → triaged → assigned → in_progress → pending_customer | pending_vendor →
resolved → closed`, plus `reopened` (from resolved/closed) and `cancelled`.
SLA clock pauses in `pending_customer`/`pending_vendor`.

**Severity / priority:** `critical | high | medium | low`. Default SLA targets
(response/resolution): critical 15m/2h, high 1h/8h, medium 4h/24h, low 8h/72h.

## 1. Data Model (PostgreSQL via TypeORM, snake_case columns)

All tenant tables include `organization_id uuid NOT NULL` (FK → organizations),
`created_at timestamptz`, `updated_at timestamptz`. RLS policy on each restricts
rows to `current_setting('app.current_org')::uuid`.

### organizations
`id uuid pk · name text · slug text unique · timezone text default 'UTC' ·
ticket_seq integer default 0 · retention_ticket_days int default 365 ·
retention_audit_days int default 2555 · created_at · updated_at`

### users
`id uuid pk · organization_id fk · email citext · email_normalized text ·
password_hash text (argon2id) · full_name text · role text (enum above) ·
status text (active|disabled) default active · totp_secret text null ·
totp_enabled bool default false · department text · last_login_at timestamptz null`
- UNIQUE `(organization_id, email_normalized)`.

### teams / team_members
teams: `id · organization_id · name · description`
team_members: `id · organization_id · team_id fk · user_id fk · skills text[]
default '{}'` — UNIQUE `(team_id, user_id)`.

### categories
`id · organization_id · name · default_team_id fk null · default_priority text ·
active bool default true` — UNIQUE `(organization_id, name)`.

### tickets
`id uuid pk · organization_id fk · number integer · subject text · description text ·
status text · priority text · category_id fk null · requester_id fk (users) ·
assignee_id fk null · team_id fk null · source text (portal|email|webhook|api) ·
system_impact text null · dependencies text[] · first_response_at timestamptz null ·
resolved_at timestamptz null · closed_at timestamptz null · reopen_count int default 0 ·
created_at · updated_at`
- UNIQUE `(organization_id, number)`  ← **INV-3**
- Indexes: `(organization_id, status)`, `(organization_id, assignee_id)`,
  `(organization_id, priority)`, `(organization_id, created_at)`.

### ticket_comments
`id · organization_id · ticket_id fk · author_id fk · body text · is_internal bool
default false · created_at`

### attachments
`id · organization_id · ticket_id fk · comment_id fk null · uploader_id fk ·
filename text · content_type text · byte_size int · storage_key text ·
sha256 text · created_at`

### sla_policies
`id · organization_id · name · priority text · response_target_mins int ·
resolution_target_mins int · business_hours_only bool default false ·
calendar_id fk null · active bool default true`

### business_hours_calendars
`id · organization_id · name · timezone text · weekly_hours jsonb
(e.g. {"mon":[["09:00","17:00"]], ...}) · holidays date[]`

### ticket_sla_states
`id · organization_id · ticket_id fk unique · policy_id fk ·
response_due_at timestamptz · resolution_due_at timestamptz ·
response_breached bool default false · resolution_breached bool default false ·
paused_at timestamptz null · accumulated_pause_ms bigint default 0 ·
warn_response_sent bool default false · warn_resolution_sent bool default false`

### escalations
`id · organization_id · ticket_id fk · reason text (sla_breach|manual) ·
from_user_id fk null · to_user_id fk null · level int · created_at`

### approvals (security access-request chains)
`id · organization_id · ticket_id fk · requested_by fk · approver_id fk ·
step int · status text (pending|approved|denied) · decided_at timestamptz null ·
decision_note text null` — UNIQUE `(ticket_id, step)`.

### notifications
`id · organization_id · user_id fk · type text · title text · body text ·
ticket_id fk null · read_at timestamptz null · created_at`

### notification_preferences
`id · organization_id · user_id fk unique · email_enabled bool default true ·
inapp_enabled bool default true · digest text (off|daily) default off`

### audit_logs  (append-only — INV-12)
`id bigserial pk · organization_id · actor_id fk null · action text ·
entity_type text · entity_id text · metadata jsonb · ip text · created_at`
- No UPDATE/DELETE grant for the app role.

### outbox_events  (ADR-04)
`id uuid pk · organization_id · kind text (email|webhook) · payload jsonb ·
status text (pending|delivering|delivered|failed) default pending ·
attempts int default 0 · idempotency_key text unique · next_attempt_at timestamptz ·
last_error text null · created_at`

### inbound_webhook_events  (idempotency — INV-4)
`id uuid pk · source text · external_id text · received_at timestamptz ·
processed_at timestamptz null · payload jsonb`
- UNIQUE `(source, external_id)`.

### webhook_endpoints (outbound) / webhook_deliveries
endpoints: `id · organization_id · url text · secret text · events text[] ·
active bool default true`
deliveries: `id · organization_id · endpoint_id fk · event text · payload jsonb ·
status text (pending|success|failed) · response_code int null · attempts int ·
created_at`

### saved_reports
`id · organization_id · name · type text (sla|volume|technician|executive) ·
params jsonb · created_by fk · created_at`

### refresh_tokens
`id uuid · organization_id · user_id fk · token_hash text · family_id uuid ·
expires_at timestamptz · revoked_at timestamptz null · created_at`
- Rotation with reuse detection (a revoked token presented ⇒ revoke whole family).

## 2. API Surface (REST, all under `/v1`, JSON, RFC 9457 errors)

Auth: `Authorization: Bearer <access>` unless noted. State-mutating
cookie-auth routes require an Origin/CSRF check. All list endpoints support
`?page&pageSize&sort` and return `{ data, page, pageSize, total }`.

### Auth
| Method | Path | Auth | Body / notes |
|---|---|---|---|
| POST | `/v1/auth/register` | public | `{ orgName, email, password, fullName }` → creates org + admin. Anti-enumeration. |
| POST | `/v1/auth/login` | public | `{ email, password, totp? }` → sets refresh cookie, returns access. Rate-limited before hash. |
| POST | `/v1/auth/refresh` | cookie | rotates refresh, returns new access. Reuse ⇒ family revoke. |
| POST | `/v1/auth/logout` | cookie | revokes refresh family. |
| GET | `/v1/auth/me` | bearer | current user + org + permissions. |
| POST | `/v1/auth/mfa/setup` | bearer | returns otpauth URL + QR data. |
| POST | `/v1/auth/mfa/verify` | bearer | `{ code }` enables TOTP. |

### Tickets
| Method | Path | Permission |
|---|---|---|
| POST | `/v1/tickets` | ticket.create |
| GET | `/v1/tickets` | ticket.read (scoped: own for end_user; team/all for staff) |
| GET | `/v1/tickets/:id` | ticket.read |
| PATCH | `/v1/tickets/:id` | ticket.update (status/priority/category/assignee/fields) |
| POST | `/v1/tickets/:id/assign` | ticket.assign `{ assigneeId }` |
| POST | `/v1/tickets/:id/escalate` | ticket.escalate `{ toUserId?, reason }` |
| POST | `/v1/tickets/:id/resolve` | ticket.update `{ resolutionNote }` |
| POST | `/v1/tickets/:id/reopen` | ticket.update |
| GET | `/v1/tickets/:id/timeline` | ticket.read (comments + audit + escalations merged) |
| POST | `/v1/tickets/:id/comments` | comment.create `{ body, isInternal? }` |
| POST | `/v1/tickets/:id/attachments` | attachment.create (multipart, image only) |
| GET | `/v1/attachments/:id` | attachment.read (streamed, Content-Disposition) |

### Approvals
| POST | `/v1/tickets/:id/approvals` | approval.create `{ approverId, step }` |
| POST | `/v1/approvals/:id/decide` | approval.decide `{ decision, note }` |
| GET | `/v1/approvals?status=pending` | approval.read (mine) |

### Dashboards / Reports
| GET | `/v1/dashboard/manager` | dashboard.manager (volume by category, MTTR by tech, SLA breach rate, queue depth) |
| GET | `/v1/dashboard/executive` | dashboard.executive (trends, utilization, SLA performance, cost-per-resolution) |
| GET | `/v1/reports/:type/export?format=pdf|csv` | report.export |

### Admin / Config
| CRUD | `/v1/users`, `/v1/teams`, `/v1/categories`, `/v1/sla-policies`, `/v1/calendars`, `/v1/webhook-endpoints`, `/v1/notification-preferences` | admin/manager scoped |
| GET | `/v1/notifications` , POST `/v1/notifications/:id/read` | self |
| GET | `/v1/audit-logs` | audit.read (security_officer/admin) |

### Webhooks
| POST | `/v1/webhooks/inbound/:source` | public + **HMAC** | monitoring alert → ticket. Verify signature (raw body) → dedupe → enqueue. **INV-5**. |

### Internal / ops (not user-facing, no UI binding required)
| GET | `/api/health` | public | `{ status, version, services:{ db, redis, queue } }` |
| GET | `/api/metrics` | internal | Prometheus/OTel metrics |

## 3. Key Workflows (SPEC §5) — each maps to UI + e2e test `W<n>`

- **W1 — End-user submits an IT request.** Portal form (category, priority,
  device, description, optional screenshot) → POST `/v1/tickets` → confirmation
  with ticket number + expected resolution time. UI: `NewTicketPage` →
  `TicketDetailPage`. Terminal: confirmation screen shows number + ETA.
- **W2 — Technician works a routed ticket.** Auto-routed ticket appears in
  technician queue; open detail with full history + related tickets; add comment;
  resolve. UI: `QueuePage` → `TicketDetailPage`. Terminal: ticket resolved,
  timeline shows resolution.
- **W3 — Technician escalates a ticket.** From the queue/detail, escalate to a
  priority queue → notifies specialist + manager; SLA countdown visible. UI:
  `TicketDetailPage` escalate action + `QueuePage` priority lane. Terminal:
  escalation recorded + visible countdown.
- **W4 — Manager real-time dashboard.** Live volume by category, MTTR by
  technician, SLA breach rate, queue depth. UI: `ManagerDashboardPage`. Terminal:
  metrics render and update via realtime.
- **W5 — Infra/Ops contextual ticket.** Ticket tagged with system impact +
  dependencies; related incidents shown; prioritize. UI: `TicketDetailPage`
  (impact/dependencies panel) + `QueuePage` filters. Terminal: impact saved,
  related shown.
- **W6 — Security access-request approval chain.** Create access-request ticket
  with approval steps; approver approves/denies; full audit trail; policy-flag.
  UI: `ApprovalsPage` + `TicketDetailPage` approvals panel. Terminal: approval
  decided + audit entry.
- **W7 — IT Leadership executive dashboard.** Volume trends, cost-per-resolution,
  utilization, SLA performance; export PDF/CSV. UI: `ExecutiveDashboardPage` +
  `ReportsPage`. Terminal: report exported (file downloaded).

## 4. Security Requirements (SPEC §4 — drives COMPLIANCE.md)

| ID | Requirement | Baseline |
|---|---|---|
| SEC-1 | Passwords hashed with Argon2id; length-based policy (min 12), HIBP-style breach screen optional | NIST 800-63B, OWASP A07 |
| SEC-2 | JWT access ≤15m; rotating refresh w/ reuse detection; httpOnly+Secure cookies | A07 |
| SEC-3 | Anti-enumeration on register/login (uniform response + timing); rate-limit before hash | A07 |
| SEC-4 | RBAC deny-by-default; per-route permission; per-org tenant scope | A01 |
| SEC-5 | Tenant isolation: org-scope wrapper + Postgres RLS | A01 |
| SEC-6 | Input validation (Zod) on every endpoint; boundary values rejected | A03 |
| SEC-7 | SQL via parameterized TypeORM only; no string SQL | A03 |
| SEC-8 | Distributed rate limiting on all public endpoints; stricter on auth/export | A05 |
| SEC-9 | Security headers via Helmet (CSP, HSTS); HTTPS-only + redirect | A05 |
| SEC-10 | Inbound webhook HMAC verified on raw body before DB; replay window ≤5m; dedupe | A08 |
| SEC-11 | Outbound webhooks signed; SSRF guard (block private/internal IPs, DNS-rebind) | A10 |
| SEC-12 | Uploads: type/size limits, image re-encode + EXIF strip, SVG blocked, isolated serving | A03/A08 |
| SEC-13 | Append-only audit log; no UPDATE/DELETE; covers auth, ticket, approval, config changes | A09 |
| SEC-14 | Secrets from env only; no secrets in repo/images; fail-fast env validation | A05 |
| SEC-15 | Dependency scanning in CI (npm audit) + SBOM | A06 |
| SEC-16 | Centralized error tracking (Sentry); structured JSON logs w/ request-id | A09 |
| SEC-17 | `compliance:` data retention — tickets 365d, audit 7y, PITR 7d (configurable) | SOC2 |
| SEC-18 | `compliance:` PII handling — names/emails/dept/device ids access-controlled, encrypted at rest (Postgres volume / disk encryption), TLS in transit | SOC2/GDPR |

## 5. Error Handling Contract (RFC 9457)

All errors: `Content-Type: application/problem+json`, body
`{ type, title, status, code, detail, errors?: { field: message } , traceId }`.
Typed codes: `E.VALIDATION` (400, with field `errors`), `E.UNAUTHENTICATED`
(401), `E.FORBIDDEN` (403), `E.NOT_FOUND` (404, also for cross-tenant to avoid
leakage), `E.CONFLICT` (409), `E.RATE_LIMITED` (429), `E.INTERNAL` (500),
`E.SERVICE_UNAVAILABLE` (503, dependency down — `{ dependency }`). Frontend maps
`errors` onto the offending control (per DESIGN.md §5).

## 6. State Transitions, Edge Cases, Race Conditions (comprehensive)

- **Ticket status machine** — only legal transitions accepted; illegal ⇒
  `E.CONFLICT`. `first_response_at` set on first staff comment. `resolved_at`/
  `closed_at`/`reopen_count` maintained. Reopen allowed within retention window.
- **Per-org ticket numbering race** — number allocated via `UPDATE organizations
  SET ticket_seq = ticket_seq + 1 RETURNING ticket_seq` inside the create
  transaction (atomic; no read-then-write gap). UNIQUE `(org, number)` is the
  backstop (INV-3).
- **Refresh-token rotation race** — concurrent refresh with the same token: first
  wins, second sees revoked ⇒ family revoke + `E.UNAUTHENTICATED`.
- **SLA pause/resume** — entering a paused status records `paused_at`; leaving
  adds elapsed to `accumulated_pause_ms` and reschedules due jobs. Escalation
  worker is idempotent (checks `*_breached` flags before acting).
- **Outbox delivery** — `idempotency_key` UNIQUE prevents double-send; relay uses
  `SELECT ... FOR UPDATE SKIP LOCKED` to claim rows; multi-row locks ordered by
  `id ASC` to avoid deadlock.
- **Inbound webhook duplicate** — UNIQUE `(source, external_id)`; duplicate
  insert ⇒ caught, returns `200` idempotently without reprocessing.
- **Assignment race** — two managers assigning concurrently: optimistic check on
  `assignee_id` version; last write wins, both audited.
- **Approval chain** — steps decided in order; deciding an out-of-order or
  already-decided step ⇒ `E.CONFLICT`.

## 7. Environment & Configuration  *(MANDATORY SECTION)*

All configuration is environment-based (no secrets in code). Required vars are
validated at boot (fail-fast). Full list in `.env.example`.

| Variable | Required | Default (dev) | Purpose |
|---|---|---|---|
| `NODE_ENV` | yes | development | runtime mode |
| `PORT` | no | 4000 | API port |
| `WEB_ORIGIN` | yes | http://localhost:5173 | CORS/CSRF origin allow-list |
| `DATABASE_URL` | yes | postgres://servicehub:servicehub@postgres:5432/servicehub | Postgres DSN |
| `DATABASE_SSL` | no | false | TLS to DB (true in prod) |
| `REDIS_URL` | yes | redis://redis:6379 | queue / rate-limit / socket adapter |
| `JWT_ACCESS_SECRET` | yes | (dev placeholder) | sign access tokens |
| `JWT_REFRESH_SECRET` | yes | (dev placeholder) | sign refresh tokens |
| `ACCESS_TTL` | no | 900 | access token seconds |
| `REFRESH_TTL` | no | 1209600 | refresh token seconds |
| `SMTP_HOST` | yes | mailhog | email transport host |
| `SMTP_PORT` | no | 1025 | email port |
| `SMTP_USER`/`SMTP_PASS` | no | (empty in dev) | SMTP auth (prod) |
| `MAIL_FROM` | no | ServiceHub <no-reply@servicehub.local> | email From |
| `WEBHOOK_INBOUND_SECRET` | yes | (dev placeholder) | HMAC secret for inbound webhooks |
| `UPLOAD_DIR` | no | /data/uploads | attachment storage root |
| `MAX_UPLOAD_BYTES` | no | 5242880 | per-file upload cap (5 MB) |
| `SENTRY_DSN` | no | (empty) | error tracking (disabled if empty) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | no | (empty) | APM traces (disabled if empty) |
| `RATE_LIMIT_WINDOW_MS` | no | 60000 | rate-limit window |
| `RATE_LIMIT_MAX` | no | 120 | requests/window/key |
| `AUTH_RATE_LIMIT_MAX` | no | 10 | auth attempts/window/key |
| `TICKET_RETENTION_DAYS` | no | 365 | retention (ADR-09) |
| `AUDIT_RETENTION_DAYS` | no | 2555 | audit retention (7y) |
| `LOG_LEVEL` | no | info | pino level |

## 8. UI Surface (screen inventory — peer to the API surface)

Persistent **app shell**: left sidebar nav (logo, role-filtered links, active
state) + top contextual header (page title, breadcrumbs, primary action,
notification bell, user menu) + content region. All screens use the bound
design tokens; every interactive element has default/hover/active/focus-visible/
disabled/loading states; every data view has designed empty/loading/error states.

| Route | Purpose | States | API bindings | Workflow | RBAC |
|---|---|---|---|---|---|
| `/` | Marketing landing (ported from template) | static | — | entry | public |
| `/login` | Sign in (email/password + TOTP) | default/loading/error(field) | `/v1/auth/login` | — | public |
| `/register` | Create org + admin | default/loading/error(field)/success | `/v1/auth/register` | — | public |
| `/app` | Redirect to role home | — | `/v1/auth/me` | — | auth |
| `/app/tickets/new` | Submit IT request (portal) | default/validating/error/success(number+ETA) | `POST /v1/tickets` | **W1** | end_user+ |
| `/app/tickets` | Ticket list (filters, search, pagination, bulk) | empty/loading/error/data | `GET /v1/tickets` | W1/W5 | all |
| `/app/tickets/:id` | Ticket detail (timeline, comments, attachments, SLA countdown, impact/deps, escalate, resolve, approvals panel) | loading/error/empty-timeline/data | ticket + timeline + comments + attachments + escalate + resolve + approvals | **W1,W2,W3,W5,W6** | scoped |
| `/app/queue` | Technician work queue (lanes by priority/status, drag-to-escalate) | empty/loading/error/data | `GET /v1/tickets?assignee/team` + assign/escalate | **W2,W3** | technician+ |
| `/app/approvals` | Pending approvals inbox + decide | empty/loading/error/data | `/v1/approvals`, `/v1/approvals/:id/decide` | **W6** | security_officer+ |
| `/app/dashboard` | Manager dashboard (metric cards w/ trend, charts, queue depth) | loading/empty/error/data | `GET /v1/dashboard/manager` | **W4** | manager+ |
| `/app/executive` | Executive dashboard (trends, utilization, SLA perf, cost) | loading/empty/error/data | `GET /v1/dashboard/executive` | **W7** | admin |
| `/app/reports` | Reports + export (PDF/CSV) | default/generating/error/success(download) | `GET /v1/reports/:type/export` | **W7** | manager+ |
| `/app/notifications` | Notification center | empty/loading/error/data | `/v1/notifications` | — | all |
| `/app/admin/users` | User & role management | empty/loading/error/data | `/v1/users` | — | admin |
| `/app/admin/teams` | Teams & skills | empty/loading/error/data | `/v1/teams` | — | admin/manager |
| `/app/admin/catalog` | Categories & SLA policies & calendars | empty/loading/error/data | `/v1/categories`,`/v1/sla-policies`,`/v1/calendars` | — | admin/manager |
| `/app/admin/integrations` | Outbound webhook endpoints | empty/loading/error/data | `/v1/webhook-endpoints` | — | admin |
| `/app/admin/audit` | Audit log viewer | empty/loading/error/data | `/v1/audit-logs` | — | security_officer/admin |
| `/app/settings` | Profile, MFA setup, notification prefs | default/loading/error/success | `/v1/auth/mfa/*`,`/v1/notification-preferences` | — | all |

Every Key Workflow W1–W7 has ≥1 screen and a terminal step reachable through the
UI. Every non-internal endpoint is reachable from ≥1 screen.

## 9. Performance Acceptance Criteria (from Success Metrics)
- API p95 < 200ms (non-export endpoints); dashboard load < 2s; ticket submit < 1s.
- Comprehensive depth: a representative load test (autocannon/k6) asserts these
  against the deployed stack at Stage 4.

## 10. Test Plan (test-after)
- **Unit**: services (SLA computation, routing scorer, password hashing, token
  rotation, webhook signature verify, SSRF guard, RFC9457 mapping).
- **Integration** (Supertest + ephemeral Postgres/Redis): auth flow, ticket CRUD,
  tenant isolation (A cannot read B), webhook idempotency + signature, rate limit,
  outbox delivery, approval chain.
- **e2e** (Playwright): exactly one `test('W<n> …')` per Key Workflow driving the
  rendered UI to its terminal step (enforced by `ui-coverage` INV-10).
- **Security**: anti-enumeration, RBAC denial, upload rejection (svg/oversize),
  concurrency race (double assign / duplicate webhook).
