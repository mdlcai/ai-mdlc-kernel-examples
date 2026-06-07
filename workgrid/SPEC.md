# SPEC.md — WorkGrid

> Behavioral contract. Build depth: **comprehensive** — every state transition,
> error path, concurrency/idempotency concern, input-validation boundary, and
> trust transition is specified. Precedence: RESEARCH > ARCHITECTURE > SPEC.
> Non-Goals (RESEARCH §"Non-Goals") are intentionally NOT specced.

## 1. Glossary & roles

- **Organization (tenant):** top-level isolation boundary. All data is org-scoped.
- **Roles** (per membership, ascending privilege): `viewer` < `member` < `manager` < `admin` < `owner`.
  - viewer: read-only. member: create/update own + assigned work. manager: manage projects/tickets/workflows, assign, approve. admin: members, SLA policies, integrations, settings. owner: billing/org delete + everything.
- **Permission matrix** is enforced by `RbacGuard`; each endpoint lists its minimum role.

## 2. Data Model (entities, columns, constraints)

All tenant tables include `id uuid pk default gen_random_uuid()`,
`organization_id uuid not null` (RLS key), `created_at timestamptz`,
`updated_at timestamptz`. Mutable domain entities add `version int` (optimistic lock).

- **organizations**: `name`, `slug` (unique global), `settings jsonb`. (Not RLS-scoped by org; scoped by membership.)
- **users**: `organization_id`, `email citext`, `password_hash`, `name`, `status` (`active|invited|disabled`), `last_login_at`. **UNIQUE(organization_id, email)** (INV-10).
- **memberships**: `user_id`, `organization_id`, `role`, **UNIQUE(user_id, organization_id)**.
- **invitations**: `email`, `role`, `token_hash`, `expires_at`, `accepted_at`.
- **refresh_tokens**: `user_id`, `token_hash`, `expires_at`, `revoked_at`, `user_agent`. UNIQUE(token_hash).
- **projects**: `key` (short code, UNIQUE(organization_id,key)), `name`, `description`, `status` (`planning|active|on_hold|completed|archived`), `health` (`on_track|at_risk|off_track`, derived), `budget_cents bigint`, `spent_cents bigint`, `start_date`, `due_date`, `lead_user_id`.
- **tasks**: `project_id`, `title`, `description`, `status` (`todo|in_progress|blocked|in_review|done`), `priority` (`low|medium|high|urgent`), `assignee_id`, `estimate_points int`, `due_date`, `position numeric` (board ordering).
- **task_dependencies**: `task_id`, `depends_on_id`, **UNIQUE(task_id, depends_on_id)**; CHECK(task_id <> depends_on_id). Cycle prevented in service.
- **tickets**: `number int` (per-org sequence, UNIQUE(organization_id,number)), `subject`, `body`, `type` (`incident|request|change|problem`), `severity` (`p1|p2|p3|p4`), `status` (`new|triage|active|blocked|resolved|closed`), `assignee_id`, `requester_email`, `sla_policy_id`, `due_at timestamptz` (SLA target), `first_response_at`, `resolved_at`, `source` (`web|email|github|slack`).
- **ticket_comments**: `ticket_id`, `author_id` (null=system), `body`, `internal bool`.
- **sla_policies**: `name`, `severity`, `response_minutes int`, `resolve_minutes int`, `active bool`.
- **workflows**: `name`, `entity` (`ticket|task|project`), `active bool`.
- **workflow_states**: `workflow_id`, `key`, `name`, `is_initial bool`, `is_terminal bool`, `requires_approval bool`, `position int`.
- **workflow_transitions**: `workflow_id`, `from_state_id`, `to_state_id`, `name`.
- **workflow_runs**: `workflow_id`, `entity_type`, `entity_id`, `current_state_id`, `status` (`active|completed|cancelled`).
- **approvals**: `workflow_run_id`, `state_id`, `requested_by`, `approver_id`, `status` (`pending|approved|rejected`), `decided_at`, `note`.
- **notifications**: `user_id`, `kind`, `title`, `body`, `link`, `read_at`. (in-app center)
- **audit_log** (append-only): `actor_user_id`, `action`, `entity_type`, `entity_id`, `metadata jsonb`, `ip`, `at`. No update/delete (INV-11).
- **outbox_event**: `aggregate_type`, `aggregate_id`, `event_type`, `payload jsonb`, `idempotency_key` **UNIQUE** (INV-7), `status` (`pending|sent|dead`), `attempts int`, `next_attempt_at`, `last_error`.
- **webhook_delivery**: `source` (`github|slack|resend`), `external_id`, **UNIQUE(source, external_id)** (INV-8), `event_type`, `payload jsonb`, `status` (`received|processed|ignored|failed`).
- **report_jobs**: `kind` (`portfolio|tickets|tasks`), `format` (`pdf|csv|json`), `params jsonb`, `status` (`queued|running|done|failed`), `result_path`, `requested_by`.

## 3. API Surface (REST, all prefixed `/v1`)

Conventions: JSON; auth via HTTP-only `access_token` cookie (or `Authorization: Bearer`); active org via `X-Org-Id` header (must match a membership). Success envelope `{ data, meta? }`. Errors RFC 9457 `application/problem+json` (see §7). All list endpoints support `?page&pageSize&sort&q&filter`. Min role in parentheses.

**Auth** (public unless noted; rate-limited)
- `POST /v1/auth/register` — `{ email, password, name, orgName }` → 201 `{ user, organization }`. Creates org + owner membership. Anti-enumeration.
- `POST /v1/auth/login` — `{ email, password }` → 200 sets cookies. Uniform failure.
- `POST /v1/auth/refresh` — rotates refresh token → 200.
- `POST /v1/auth/logout` (auth) — revokes refresh token → 204.
- `GET /v1/auth/me` (auth) — `{ user, memberships, activeOrg }`.

**Organizations & members**
- `GET /v1/orgs` (auth) — orgs the user belongs to.
- `POST /v1/orgs` (auth) — create org → creator becomes owner.
- `GET /v1/orgs/:id` (member) · `PATCH /v1/orgs/:id` (admin) · settings.
- `GET /v1/members` (member) · `POST /v1/members/invite` (admin) · `POST /v1/members/accept` (public+token) · `PATCH /v1/members/:id/role` (admin) · `DELETE /v1/members/:id` (admin).

**Projects** (`/v1/projects`)
- `GET` (viewer) list · `POST` (manager) · `GET /:id` (viewer) · `PATCH /:id` (manager, version) · `DELETE /:id` (admin) · `GET /:id/health` (viewer).

**Tasks** (`/v1/tasks` and `/v1/projects/:id/tasks`)
- `GET` (viewer) · `POST` (member) · `GET /:id` (viewer) · `PATCH /:id` (member, version) · `DELETE /:id` (manager) · `POST /:id/dependencies` (member) · `DELETE /:id/dependencies/:depId` (member) · `PATCH /:id/move` (member; board reorder).

**Tickets** (`/v1/tickets`)
- `GET` (viewer) · `POST` (member) · `GET /:id` (viewer) · `PATCH /:id` (member, version) · `POST /:id/assign` (manager) · `POST /:id/comments` (member) · `GET /:id/comments` (viewer) · `POST /:id/resolve` (member) · `POST /:id/reopen` (member).

**Workflows** (`/v1/workflows`)
- `GET` (member) · `POST` (manager) · `GET /:id` (member) · `PATCH /:id` (manager) · `POST /:id/runs` (member; start a run on an entity) · `POST /runs/:runId/transition` (member) · `POST /approvals/:id/decide` (manager; `{decision, note}`).

**Dashboard** (`/v1/dashboard`)
- `GET /portfolio` (viewer) — aggregate metrics. `GET /sla` (viewer) — SLA breach summary. `GET /velocity` (viewer).

**Notifications** (`/v1/notifications`)
- `GET` (auth) · `POST /:id/read` (auth) · `POST /read-all` (auth) · unread count in `meta`.

**Reports** (`/v1/reports`)
- `POST` (member) `{kind, format, params}` → 202 `{ jobId }` · `GET /:jobId` (member) status · `GET /:jobId/download` (member) streams the artifact.

**Webhooks (inbound, public, signature-verified)**
- `POST /v1/webhooks/github` · `POST /v1/webhooks/slack` · `POST /v1/webhooks/resend`. Raw body; HMAC verified before processing (INV-4).

**Health** — `GET /v1/health` and alias `GET /api/health` → `{ status, services:{db,redis}, version, time }`.

**Internal** (not UI-bound): outbox relay, queue processors, reconciliation sweep.

## 4. Security Requirements (SPEC §4 — maps to COMPLIANCE.md)

| ID | Requirement | Tag |
|----|-------------|-----|
| SEC-1 | Tenant isolation via RLS FORCE + per-request GUC; cross-tenant read returns NOT_FOUND. | compliance:SOC2, OWASP-A01 |
| SEC-2 | AuthZ: RBAC guard on every non-public route; deny-by-default. | OWASP-A01 |
| SEC-3 | AuthN: Argon2id password hashing (NIST 800-63B); breached-password screening (HIBP k-anonymity) on register/change. | compliance:SOC2 |
| SEC-4 | Anti-enumeration on auth endpoints: uniform response shape + timing; rate-limit before hash compare. | OWASP-A07 |
| SEC-5 | Rate limiting (Redis-backed) on all public endpoints with correct key dimensions `(ip,email)` for auth. | OWASP-A04 |
| SEC-6 | Inbound webhook HMAC verification over raw body, before any DB read; Slack ±5-min replay window; dedup by (source,external_id). | OWASP-A08 |
| SEC-7 | Idempotency: outbox unique idempotency_key; webhook dedup; optimistic version on mutations (409 on conflict). | — |
| SEC-8 | Audit logging: append-only audit_log for every state-mutating action; 99% trail completeness. | compliance:SOC2, audit_logging |
| SEC-9 | Secrets via env vars; `.env` gitignored, `.env.example` committed; no secrets in code/images. | compliance:SOC2 |
| SEC-10 | Transport: HTTPS only; HSTS; helmet security headers; secure HTTP-only cookies; CSRF/Origin guard on state-mutating browser routes. | OWASP-A05 |
| SEC-11 | Input validation: class-validator DTOs on all inputs; boundary checks; reject unknown fields (whitelist). | OWASP-A03 |
| SEC-12 | PII handling: encryption at rest; PII redaction in logs; audit access to PII. | compliance:SOC2, pii_handling |
| SEC-13 | Dependency hygiene: pinned versions + lockfiles; `npm audit` in CI. | OWASP-A06 |
| SEC-14 | SSRF guard: outbound webhook/integration URLs validated against an allowlist; no internal addresses. | OWASP-A10 |
| SEC-15 | Error contract: RFC 9457 problem+json; no stack traces / internal detail leaked to clients. | OWASP-A05 |

## 5. Key Workflows (RESEARCH §"Users & Outcomes" → e2e, 1:1 with e2e tests)

Each `W<n>` has exactly one Playwright e2e test driving it through the rendered UI to its terminal step (INV-16).

- **W1 — Auth bootstrap & org creation.** Register (email+password+orgName) → land on dashboard → `GET /me` shows owner. Terminal: authenticated dashboard rendered.
- **W2 — Incident intake → auto-route → escalate.** Create P1 ticket → auto-routed to assignee by severity/skill → SLA `due_at` set → escalation job scheduled; manager views queue with routing chip. Terminal: ticket detail shows assignee + SLA timer; escalation visible on breach.
- **W3 — Project with linked tasks & dependencies.** Create project → add tasks → link dependency → blocked task surfaced; board view shows critical path/conflict flags. Terminal: dependency rendered + blocked badge in UI.
- **W4 — Sprint/workflow with blocked surfacing.** Configure workflow (states+transitions) → start run on a task → transition → blocked state notifies. Terminal: workflow run state advanced in UI.
- **W5 — Runbook with approval gates.** Workflow with `requires_approval` state → start run → reach gate → manager approves/rejects → run proceeds; audit trail recorded. Terminal: approval decided + run continues in UI.
- **W6 — Executive portfolio dashboard.** Open dashboard → live project health, budget burn, velocity, SLA breach widgets. Terminal: dashboard renders aggregated numbers (not raw JSON).
- **W7 — Service-desk resolution & customer notification.** Agent opens assigned ticket → adds comment → resolves → notification queued (in-app + email via outbox). Terminal: ticket status `resolved` + notification visible.
- **W8 — Report export (PDF/CSV/JSON).** From dashboard/tickets, request export → job runs → download artifact through the UI. Terminal: file downloaded via UI button.

Each workflow’s **terminal step is UI-reachable** (not API-only).

## 6. State machines & transitions (comprehensive)

**Ticket status:** `new → triage → active → (blocked ⇄ active) → resolved → closed`; `resolved → active` (reopen). Pre/post: entering `active` sets `first_response_at` if null; entering `resolved` sets `resolved_at` and cancels escalation job; `closed` is terminal (no further edits except reopen by manager). Invalid transition → `422 E.INVALID_TRANSITION`.

**Task status:** `todo → in_progress → (blocked) → in_review → done`. A task cannot enter `done` while it has unmet dependencies → `409 E.DEPENDENCY_UNMET`. Cyclic dependency on create → `422 E.DEPENDENCY_CYCLE`.

**Workflow run:** `active → completed` (reach terminal state) or `active → cancelled`. A transition into a `requires_approval` state creates a `pending` approval and **pauses** progression until decided; transitioning a paused run → `409 E.APPROVAL_PENDING`.

**Approval:** `pending → approved|rejected` (terminal). Re-deciding → `409 E.ALREADY_DECIDED`. Only `manager+` and not the requester (separation of duties) → `403 E.FORBIDDEN`.

**Outbox event:** `pending → sent` or `pending →(retries exhausted) dead`. Delivery is idempotent by `idempotency_key`.

**SLA:** on ticket create/route → compute `due_at = now + policy.response/resolve`; schedule delayed job. State change to `resolved` cancels. Reconciliation sweep recomputes from `due_at`.

## 7. Error contract & edge cases

RFC 9457 envelope:
```json
{ "type":"https://workgrid.app/errors/<code>", "title":"<human>", "status":<int>,
  "detail":"<message>", "instance":"<request-id>", "code":"E.<CODE>",
  "errors":[{"field":"email","message":"must be a valid email"}] }
```
Typed codes: `E.VALIDATION` (400 + field `errors[]`), `E.UNAUTHENTICATED` (401), `E.FORBIDDEN` (403), `E.NOT_FOUND` (404), `E.CONFLICT` (409, optimistic version / unique), `E.INVALID_TRANSITION` (422), `E.DEPENDENCY_UNMET` (409), `E.DEPENDENCY_CYCLE` (422), `E.APPROVAL_PENDING` (409), `E.ALREADY_DECIDED` (409), `E.RATE_LIMITED` (429 + `Retry-After`), `E.SERVICE_UNAVAILABLE` (503 + `dependency`). Auth failures use a single `E.AUTH_FAILED` (401) regardless of cause (anti-enumeration). The UI MUST map `errors[]` to the offending control.

**Concurrency/idempotency:** all PATCH carry `version`; mismatch → 409. Double-submit on create guarded by unique constraints. Webhook replay → 200 idempotent (no duplicate side effect). Multi-row locks acquired `ORDER BY id` to avoid deadlock.

## 8. UI Surface (Design System + screen inventory)

### 8.1 Design System (binding; tokens copied verbatim from DESIGN-TEMPLATE.html — ADR-0002)
- **Tokens:** the canonical `:root` set (colors `--color-*`, fonts `--font-*`, spacing `--space-1..9` = 4/8/12/16/24/32/48/64/96, radius sm/md/lg = 5/10/18, shadows sm/md/lg, `--maxw:1200px`, `--ring`). Dark mode via `[data-theme="dark"]` + `prefers-color-scheme` fallback. Defined once in `apps/web/src/app/globals.css` and mirrored in `tailwind.config.ts`; every screen consumes tokens (INV-9, INV-17).
- **Fonts:** Space Grotesk (display/headings/numerics), Inter (body), JetBrains Mono (eyebrows, IDs, badges) via `next/font`.
- **Archetype:** `saas` — persistent app shell (224px sidebar + contextual header + content). Dashboard = composed hierarchy (lead metric → supporting → detail), not a tile grid. Data tables and forms are first-class with every state designed.
- **Components:** button (primary/ghost), card, pill/badge (severity/status variants), data table (sort/filter/paginate/row actions/empty/loading/error), metric card w/ trend, form fields w/ inline validation, drawer/modal, toast, sidebar nav w/ active state, theme toggle, command palette (⌘K). All built on shadcn/Radix, restyled to tokens (no stock look).
- **Motion:** functional, 120–320ms eased; `prefers-reduced-motion` honored.
- **A11y:** WCAG 2.2 AA; visible `:focus-visible` ring (`--ring`); ≥44px hit targets; semantic landmarks; labelled controls.

### 8.2 Screen inventory
Each screen: route · purpose · components & states (empty/loading/error/success) · bound endpoints · workflow steps · RBAC.

| Route | Purpose | Key states | Endpoints | Workflows | RBAC |
|-------|---------|-----------|-----------|-----------|------|
| `/` (marketing) | Landing — ported from template (nav, hero+mockup, bento features, CTA, footer) | success | — | entry | public |
| `/login` | Sign in | empty/error(field)/loading/success | `POST /auth/login` | W1 | public |
| `/register` | Create account + org | empty/error(field)/loading/success | `POST /auth/register` | W1 | public |
| `/invite/accept` | Accept invitation | loading/error/success | `POST /members/accept` | W1 | public+token |
| `/app` (dashboard) | Executive portfolio: health, budget burn, velocity, SLA | empty/loading/error/success | `GET /dashboard/*` | W6 | viewer |
| `/app/tickets` | Ticket queue (table, filters, severity/status pills, auto-route chip) | empty/loading/error/success | `GET /tickets` | W2,W7 | viewer |
| `/app/tickets/:id` | Ticket detail: SLA timer, assign, comments, resolve/reopen, workflow | all + 404 | `GET/PATCH/POST /tickets/:id*` | W2,W7 | viewer |
| `/app/projects` | Projects list (health badges, budget) | empty/loading/error/success | `GET /projects` | W3 | viewer |
| `/app/projects/:id` | Project detail: tasks board/list, dependencies, critical path | all + 404 | `GET/PATCH /projects/:id`, `/tasks*` | W3 | viewer |
| `/app/tasks/:id` | Task detail: status, assignee, dependencies, blocked surfacing | all + 404 | `GET/PATCH /tasks/:id*` | W3,W4 | viewer |
| `/app/workflows` | Workflow definitions list | empty/loading/error/success | `GET /workflows` | W4,W5 | member |
| `/app/workflows/:id` | Workflow editor (states/transitions/approval gates) + runs | all | `GET/PATCH /workflows/:id`, `/runs` | W4,W5 | member |
| `/app/approvals` | Approval inbox (pending gates) → approve/reject | empty/loading/error/success | `POST /approvals/:id/decide` | W5 | manager |
| `/app/notifications` | In-app notification center | empty/loading/error/success | `GET /notifications`, read | W7 | auth |
| `/app/reports` | Build & download reports (PDF/CSV/JSON) | empty/loading/error/success | `POST /reports`, `GET /reports/:id*` | W8 | member |
| `/app/members` | Members & roles, invite | empty/loading/error/success | `GET/POST/PATCH /members*` | W1 | admin |
| `/app/settings` | Org settings, SLA policies, integrations, theme | loading/error/success | `PATCH /orgs/:id`, SLA, integrations | — | admin |
| `/app/audit` | Audit log viewer (filter by actor/entity/action) | empty/loading/error/success | `GET /audit` | — | admin |

Every Key Workflow (W1–W8) is completable end-to-end through the UI including its terminal step. Every non-internal endpoint is bound to ≥1 screen.

## 9. Performance & acceptance criteria

| Operation | Target |
|-----------|--------|
| API reads (p95) | < 200 ms |
| Dashboard initial load | < 2 s |
| Real-time event delivery | < 1 s |
| Ticket create→route | < 500 ms server |
| Report (CSV/JSON) job | < 5 s for 10k rows |

Each is an acceptance criterion; comprehensive depth adds a representative load test (autocannon) at Stage 4 smoke.

## 10. Environment & Configuration (mandatory)

All config via env vars; `.env.example` committed (real `.env` gitignored). Vars (★ = secret):

| Var | Purpose | Default (dev) | Required |
|-----|---------|---------------|----------|
| `NODE_ENV` | environment | development | yes |
| `API_PORT` | API listen port | 4000 | yes |
| `WEB_PORT` | web port | 3000 | yes |
| `DATABASE_URL` | Postgres DSN (app role, non-superuser) | postgres://workgrid:workgrid@postgres:5432/workgrid | yes |
| `DATABASE_SSL` | require TLS to DB | false (dev) | no |
| `REDIS_URL` | Redis DSN | redis://redis:6379 | yes |
| `JWT_ACCESS_SECRET` ★ | access token signing | dev-only value | yes |
| `JWT_REFRESH_SECRET` ★ | refresh token signing | dev-only value | yes |
| `ACCESS_TOKEN_TTL` | access TTL | 900s | no |
| `REFRESH_TOKEN_TTL` | refresh TTL | 30d | no |
| `APP_URL` | public base URL (HTTPS) | https://localhost | yes |
| `CORS_ORIGINS` | allowed origins | https://localhost | yes |
| `RESEND_API_KEY` ★ | email | (placeholder → no-op adapter) | no |
| `EMAIL_FROM` | sender | work@localhost | no |
| `GITHUB_WEBHOOK_SECRET` ★ | inbound HMAC | (placeholder) | no |
| `SLACK_SIGNING_SECRET` ★ | inbound HMAC | (placeholder) | no |
| `SLACK_WEBHOOK_URL` ★ | outbound | (placeholder → no-op) | no |
| `RESEND_WEBHOOK_SECRET` ★ | inbound svix | (placeholder) | no |
| `SENTRY_DSN` ★ | error reporting | (placeholder → no-op) | no |
| `RATE_LIMIT_WINDOW` / `RATE_LIMIT_MAX` | throttler | 60s / 100 | no |
| `LOG_LEVEL` | pino level | info | no |

**Startup:** migrations run before serve (`migration:run`); seed (`seed`) inserts a demo org, owner, sample projects/tasks/tickets/workflow for first-run + smoke. **Externals with placeholder secrets degrade to logging no-op adapters** (ADR-0008) so the system runs and is smoke-testable offline.

## 11. Test plan

- **Unit (Jest):** services (auth, tenant scope, tickets/SLA, tasks/deps, workflow/approval, outbox, signature verification). TDD per feature.
- **Integration (Jest + Testcontainers/compose pg+redis):** RLS isolation (cross-tenant denied), optimistic concurrency 409, webhook signature reject + dedup, outbox delivery idempotency, escalation scheduling.
- **e2e (Playwright):** one test per Key Workflow W1–W8 through the rendered UI to terminal step (INV-16), plus negative-path field-error rendering.
- **Invariant lint:** `npm run lint:invariants` evaluates `invariants.json` (blocking).
- **Smoke:** `SMOKE-TEST.md` + `smoke-test.sh` (Functional Smoke Test matrix) + large-cookie resilience.
