# SPEC.md — Orbit IT Command Center

> Stage 2 output of the MDLC™ pipeline. **Source of truth for behavior.** Depth: `comprehensive` — every state transition carries pre/post conditions, every error path its recovery, every endpoint its validation bounds, trust boundary and performance expectation. Architecture is in `ARCHITECTURE.md`; decisions in `DECISIONS.md`.

Conventions used throughout: `org` = tenant; roles `admin` > `manager` > `agent` > `requester` (each role includes the rights of the roles below it unless a row says otherwise); `principal` = the authenticated user; all timestamps are UTC ISO-8601; all ids are UUID v4 unless stated; `§7 envelope` = the RFC 9457 error contract in section 7.

---

## 1. Overview & scope

**Problem.** IT teams juggle tickets, projects, changes, assets, approvals and documentation across disconnected tools with no shared ownership, SLA or audit trail.

**Product.** Orbit is a multi-tenant IT command center: one org-scoped data model where tickets, projects/tasks, changes, assets, KB articles and approvals link to one another, every mutation writes an immutable audit event, SLA timers run against business calendars, automation rules route and escalate, and open browsers receive live updates over Server-Sent Events.

**Personas → roles.**

| Persona | Role | Can |
|---|---|---|
| Employee | `requester` | Submit and follow own tickets; read published KB; decide approvals addressed to them; edit own profile |
| IT technician | `agent` | Everything above for all org tickets; work queues; manage changes, assets, projects, tasks, KB; request approvals |
| IT manager / lead | `manager` | Everything above + SLA policies, analytics, approvals inbox for changes/purchases, audit read |
| IT admin | `admin` | Everything + users, teams, catalog, automation, org settings, soft-delete |

**Non-Goals (restated from RESEARCH.md, binding):** SSO / OAuth / SAML; file attachments and image uploads; outbound Slack / Teams / PagerDuty webhooks; SMS; native mobile apps; billing or payments; internationalization; offline mode; CMDB auto-discovery; inbound email-to-ticket; multi-currency purchasing; custom scripting/plugins; multi-org membership per user. Nothing in this spec implements them.

**Scale tier:** small (< 1k concurrent) — single Postgres, in-process scheduler, no cache/queue.

---

## 2. Data model

All tables live in the `public` schema. `T` = tenant table (has `org_id uuid NOT NULL REFERENCES orgs(id)`, RLS enabled + forced, policy `org_id = NULLIF(current_setting('app.org_id', true), '')::uuid`). `S` = system table (no RLS; accessed only by the auth/system modules). Every `T` table has `created_at timestamptz NOT NULL DEFAULT now()`, `updated_at timestamptz NOT NULL DEFAULT now()` (touched by the service layer). Mutable entities carry `version int NOT NULL DEFAULT 1`. PII columns are tagged **PII** and redacted from logs.

### 2.1 System tables

| Table | Kind | Columns (name type constraints) | Ownership |
|---|---|---|---|
| `orgs` | S | `id uuid PK`, `name text NOT NULL (1–120)`, `slug citext NOT NULL UNIQUE (3–40, `[a-z0-9-]`)`, `timezone text NOT NULL DEFAULT 'UTC'`, `created_at` | — |
| `users` | S | `id uuid PK`, `org_id uuid NOT NULL FK orgs`, `email citext NOT NULL UNIQUE` **PII**, `name text NOT NULL (1–120)` **PII**, `role text NOT NULL CHECK IN (admin,manager,agent,requester)`, `password_hash text NOT NULL`, `is_active bool NOT NULL DEFAULT true`, `must_reset_password bool NOT NULL DEFAULT false`, `theme text NOT NULL DEFAULT 'system' CHECK IN (system,light,dark)`, `last_login_at timestamptz`, `version`, `created_at`, `updated_at` | org |
| `sessions` | S | `id uuid PK`, `user_id uuid NOT NULL FK users ON DELETE CASCADE`, `token_hash text NOT NULL UNIQUE` (HMAC-SHA256 of the cookie token, keyed by `SESSION_SECRET`), `expires_at timestamptz NOT NULL`, `last_seen_at timestamptz NOT NULL`, `ip inet`, `user_agent text (≤ 512)`, `created_at` | user |
| `password_reset_tokens` | S | `id uuid PK`, `user_id uuid NOT NULL FK users ON DELETE CASCADE`, `token_hash text NOT NULL UNIQUE`, `purpose text NOT NULL CHECK IN (reset,invite)`, `expires_at timestamptz NOT NULL`, `used_at timestamptz`, `created_at` | user |
| `org_counters` | S | `org_id uuid PK FK orgs`, `tickets int NOT NULL DEFAULT 0`, `changes int NOT NULL DEFAULT 0` | org |
| `scheduler_state` | S | `id text PK` (`'main'`), `last_tick_at timestamptz`, `last_error text` | — |

### 2.2 Tenant tables

| Table | Columns | Constraints / indexes |
|---|---|---|
| `teams` | `id`, `org_id`, `name text (1–80)`, `description text (≤ 500)`, `lead_user_id uuid FK users NULL`, `version`, timestamps | `UNIQUE(org_id, name)` |
| `team_members` | `team_id uuid FK teams ON DELETE CASCADE`, `user_id uuid FK users ON DELETE CASCADE`, `org_id`, `created_at` | `PK(team_id, user_id)`; index `(org_id, user_id)` |
| `ticket_categories` | `id`, `org_id`, `name text (1–80)`, `description text (≤ 300)`, `ticket_type text CHECK IN (incident,request,access,problem)`, `default_team_id uuid FK teams NULL`, `default_priority text CHECK IN (low,medium,high,critical) DEFAULT 'medium'`, `sla_policy_id uuid FK sla_policies NULL`, `requires_approval bool DEFAULT false`, `is_active bool DEFAULT true`, `sort_order int DEFAULT 0`, `version`, timestamps | `UNIQUE(org_id, name)` |
| `sla_policies` | `id`, `org_id`, `name text (1–80)`, `priority text CHECK IN (low,medium,high,critical)`, `response_minutes int CHECK 1–43200`, `resolution_minutes int CHECK 1–129600`, `business_hours jsonb NOT NULL` (`{timezone, days:[1..5], start:"09:00", end:"17:00"}` or `{"always":true}`), `is_default bool DEFAULT false`, `version`, timestamps | `UNIQUE(org_id, name)`; partial unique `(org_id, priority) WHERE is_default` |
| `tickets` | `id`, `org_id`, `number int NOT NULL`, `type text CHECK IN (incident,request,access,problem)`, `title text (3–200)`, `description text (0–20000)`, `status text CHECK IN (new,open,in_progress,waiting_on_requester,pending_approval,resolved,closed,cancelled)`, `priority text CHECK IN (low,medium,high,critical)`, `category_id uuid FK ticket_categories NULL`, `requester_id uuid FK users NOT NULL`, `assignee_id uuid FK users NULL`, `team_id uuid FK teams NULL`, `asset_id uuid FK assets NULL`, `sla_policy_id uuid FK sla_policies NULL`, `response_due_at timestamptz`, `resolution_due_at timestamptz`, `first_responded_at timestamptz`, `resolved_at timestamptz`, `closed_at timestamptz`, `sla_paused_at timestamptz`, `sla_paused_ms bigint DEFAULT 0`, `sla_response_breached_at timestamptz`, `sla_resolution_breached_at timestamptz`, `reassignment_count int DEFAULT 0`, `tags text[] DEFAULT '{}'` (≤ 10, each 1–30), `deleted_at timestamptz`, `version`, timestamps | `UNIQUE(org_id, number)`; indexes `(org_id, status, priority)`, `(org_id, assignee_id)`, `(org_id, requester_id)`, `(org_id, team_id)`, `(org_id, resolution_due_at) WHERE resolved_at IS NULL`; GIN on `to_tsvector('english', title || ' ' || description)` |
| `ticket_comments` | `id`, `org_id`, `ticket_id uuid FK tickets ON DELETE CASCADE`, `author_id uuid FK users`, `body text (1–10000)`, `is_private bool DEFAULT false` (staff-only note), `created_at` | index `(ticket_id, created_at)` |
| `approvals` | `id`, `org_id`, `subject_type text CHECK IN (ticket,change,project,purchase)`, `subject_id uuid NOT NULL`, `title text (1–200)`, `requested_by uuid FK users`, `approver_id uuid FK users`, `status text CHECK IN (pending,approved,rejected,cancelled)`, `note text (≤ 2000)`, `decision_note text (≤ 2000)`, `decided_at timestamptz`, `expires_at timestamptz`, `version`, timestamps | partial `UNIQUE(org_id, subject_type, subject_id, approver_id) WHERE status = 'pending'`; index `(org_id, approver_id, status)` |
| `changes` | `id`, `org_id`, `number int`, `title (3–200)`, `description (0–20000)`, `risk text CHECK IN (low,medium,high)`, `status text CHECK IN (draft,submitted,approved,rejected,scheduled,implementing,completed,rolled_back,cancelled)`, `requester_id FK users`, `implementer_id FK users NULL`, `ticket_id uuid FK tickets NULL`, `asset_id uuid FK assets NULL`, `planned_start timestamptz`, `planned_end timestamptz`, `implementation_plan text (0–20000)`, `rollback_plan text (0–20000)`, `completed_at`, `deleted_at`, `version`, timestamps | `UNIQUE(org_id, number)`; `CHECK (planned_end IS NULL OR planned_start IS NULL OR planned_end > planned_start)`; index `(org_id, status)` |
| `assets` | `id`, `org_id`, `tag text (1–60)`, `name text (1–120)`, `type text CHECK IN (computer,server,software,network,peripheral,other)`, `status text CHECK IN (in_stock,assigned,in_repair,retired)`, `serial text (≤ 120)`, `model text (≤ 120)`, `location text (≤ 120)`, `assigned_to_user_id uuid FK users NULL`, `purchase_date date`, `warranty_end date`, `notes text (≤ 5000)`, `deleted_at`, `version`, timestamps | `UNIQUE(org_id, tag)`; index `(org_id, assigned_to_user_id)`, `(org_id, status)` |
| `projects` | `id`, `org_id`, `name (1–120)`, `description (0–20000)`, `status text CHECK IN (planning,active,on_hold,completed,cancelled)`, `health text CHECK IN (on_track,at_risk,late)` (derived nightly + on task change), `owner_id FK users`, `start_date date`, `due_date date`, `deleted_at`, `version`, timestamps | index `(org_id, status)` |
| `milestones` | `id`, `org_id`, `project_id FK projects ON DELETE CASCADE`, `name (1–120)`, `due_date date`, `status text CHECK IN (open,completed)`, `sort_order int`, `version`, timestamps | index `(project_id)` |
| `tasks` | `id`, `org_id`, `project_id FK projects ON DELETE CASCADE`, `milestone_id FK milestones NULL ON DELETE SET NULL`, `title (1–200)`, `description (0–10000)`, `status text CHECK IN (todo,in_progress,blocked,done)`, `priority text CHECK IN (low,medium,high)`, `assignee_id FK users NULL`, `due_date date`, `depends_on_task_id uuid FK tasks NULL`, `completed_at`, `version`, timestamps | index `(org_id, assignee_id, status)`, `(project_id, status)`; `CHECK (depends_on_task_id <> id)` |
| `kb_articles` | `id`, `org_id`, `slug text (3–120)`, `title (3–200)`, `body text (1–100000, Markdown)`, `category text (≤ 60)`, `tags text[]`, `status text CHECK IN (draft,published,archived)`, `author_id FK users`, `published_at`, `view_count int DEFAULT 0`, `search_vector tsvector GENERATED ALWAYS AS (setweight(to_tsvector('english', title),'A') || setweight(to_tsvector('english', coalesce(array_to_string(tags,' '),'')),'B') || setweight(to_tsvector('english', body),'C')) STORED`, `deleted_at`, `version`, timestamps | `UNIQUE(org_id, slug)`; GIN `(search_vector)` |
| `automation_rules` | `id`, `org_id`, `name (1–120)`, `trigger text CHECK IN (ticket.created,ticket.updated,ticket.transitioned,sla.breached,approval.decided,change.submitted,schedule)`, `conditions jsonb NOT NULL` (`{all:[{field,op,value}], any:[...]}`; ≤ 20 predicates), `actions jsonb NOT NULL` (array ≤ 10 of `{type, ...}`), `is_active bool DEFAULT true`, `sort_order int`, `schedule jsonb NULL` (`{every:'day'|'week', at:'HH:MM'}` when trigger = schedule), `last_run_at`, `version`, timestamps | `UNIQUE(org_id, name)` |
| `automation_runs` | `id`, `org_id`, `rule_id FK automation_rules ON DELETE CASCADE`, `trigger text`, `entity_type text`, `entity_id uuid`, `matched bool`, `actions_applied jsonb`, `error text`, `depth int`, `created_at` | index `(org_id, created_at DESC)`, `(rule_id, created_at DESC)` |
| `recurring_tasks` | `id`, `org_id`, `title (1–200)`, `description (0–5000)`, `kind text CHECK IN (ticket,task)`, `template jsonb NOT NULL` (ticket: `{type, priority, categoryId, teamId}`; task: `{projectId, assigneeId, priority}`), `interval text CHECK IN (daily,weekly,monthly)`, `next_run_at timestamptz NOT NULL`, `last_run_at`, `is_active bool DEFAULT true`, `version`, timestamps | index `(org_id, next_run_at) WHERE is_active` |
| `notifications` | `id`, `org_id`, `user_id FK users ON DELETE CASCADE`, `type text` (`ticket.assigned`, `ticket.commented`, `ticket.transitioned`, `sla.breached`, `approval.requested`, `approval.decided`, `change.transitioned`, `task.assigned`, `system`), `title text (≤ 200)`, `body text (≤ 1000)`, `link text (≤ 300)` (root-relative app path), `read_at`, `created_at` | index `(user_id, read_at, created_at DESC)` |
| `pending_emails` | `id`, `org_id NULL` (system mails allowed), `to_address citext NOT NULL` **PII**, `template_key text NOT NULL`, `payload jsonb NOT NULL`, `business_ref_id text NOT NULL`, `status text CHECK IN (pending,sending,sent,failed,dlq) DEFAULT 'pending'`, `attempts int DEFAULT 0`, `next_attempt_at timestamptz NOT NULL DEFAULT now()`, `sent_at`, `error_text`, `created_at` | `UNIQUE(to_address, template_key, business_ref_id)`; index `(status, next_attempt_at)` |
| `audit_events` | `id bigserial PK`, `org_id`, `actor_id uuid NULL` (NULL = system/scheduler), `actor_role text`, `entity_type text NOT NULL`, `entity_id uuid NOT NULL`, `action text NOT NULL` (`created`, `updated`, `transitioned`, `assigned`, `commented`, `approved`, `rejected`, `deleted`, `restored`, `login`, `logout`, `role_changed`, `rule_fired`, …), `before jsonb`, `after jsonb`, `request_id text`, `ip inet`, `created_at` | **append-only**: grants `INSERT, SELECT` only to `orbit_app`; no UPDATE/DELETE policy; index `(org_id, entity_type, entity_id, created_at DESC)`, `(org_id, created_at DESC)`, `(org_id, actor_id)` |

**Soft delete.** `tickets`, `changes`, `assets`, `projects`, `kb_articles` use `deleted_at`; all list/get queries filter `deleted_at IS NULL` unless the caller is `admin` and passes `includeDeleted=true`. Audit rows referencing a deleted entity remain resolvable.

**Ownership summary.** Per-user-owned (requester may see only own): tickets (as requester), approvals (as approver or requester), notifications (own), sessions (own). Org-shared for `agent`+: everything else. KB: `requester` sees `published` only.

---

## 3. API surface

Base path `/api/v1` (health and OpenAPI live under `/api`). Auth column: `public` (no session), `session` (any active session), or the minimum role. Every request/response body is JSON (`application/json`), max body 256 KB. Every list endpoint accepts `cursor` (opaque), `limit` (1–100, default 25) and returns `{ data: [...], nextCursor: string|null }`. `PATCH` on versioned entities requires `If-Match: "<version>"` (or `version` in body); mismatch → `409 VERSION_CONFLICT`. Cross-org or unowned resources → `404 NOT_FOUND`. `internal: yes` rows are not bound to a screen.

| Endpoint | Auth | Input | Output | internal |
|---|---|---|---|---|
| `GET /api/health` | public | — | `{status, version, commit, uptimeSec, checks:{db,scheduler,mailer}}` 200/503 | internal: yes |
| `GET /api/openapi.json` | public | — | OpenAPI 3.1 document | internal: yes |
| `POST /api/v1/auth/register` | public | `{orgName, orgSlug, name, email, password}` | 201 `{user, org}` + session cookie | no |
| `POST /api/v1/auth/login` | public | `{email, password}` | 200 `{user}` + session cookie | no |
| `POST /api/v1/auth/logout` | session | — | 204 + cookie cleared | no |
| `GET /api/v1/auth/me` | session | — | `{user, org, permissions[]}` | no |
| `PATCH /api/v1/auth/me` | session | `{name?, theme?}` + version | `{user}` | no |
| `POST /api/v1/auth/me/password` | session | `{currentPassword, newPassword}` | 204 (other sessions revoked) | no |
| `POST /api/v1/auth/password/forgot` | public | `{email}` | 202 `{}` always | no |
| `POST /api/v1/auth/password/reset` | public | `{token, password}` | 204 | no |
| `GET /api/v1/users` | agent | `q?, role?, teamId?, isActive?, sort?, cursor, limit` | list of `UserSummary` | no |
| `POST /api/v1/users` | admin | `{name, email, role, teamIds?}` | 201 `{user}`; invite email queued | no |
| `GET /api/v1/users/:id` | agent | — | `{user, teams[]}` | no |
| `PATCH /api/v1/users/:id` | admin | `{name?, role?, isActive?, teamIds?}` + version | `{user}` | no |
| `DELETE /api/v1/users/:id` | admin | — | 204 (deactivates; sessions revoked) | no |
| `GET /api/v1/teams` | session | `q?, cursor, limit` | list of `Team` (with member count) | no |
| `POST /api/v1/teams` | admin | `{name, description?, leadUserId?}` | 201 `{team}` | no |
| `GET /api/v1/teams/:id` | session | — | `{team, members[]}` | no |
| `PATCH /api/v1/teams/:id` | admin | `{name?, description?, leadUserId?}` + version | `{team}` | no |
| `DELETE /api/v1/teams/:id` | admin | — | 204 (tickets keep `team_id` NULLed) | no |
| `POST /api/v1/teams/:id/members` | admin | `{userId}` | 201 `{members[]}` | no |
| `DELETE /api/v1/teams/:id/members/:userId` | admin | — | 204 | no |
| `GET /api/v1/ticket-categories` | session | `includeInactive?` | list | no |
| `POST /api/v1/ticket-categories` | admin | `{name, description?, ticketType, defaultTeamId?, defaultPriority?, slaPolicyId?, requiresApproval?}` | 201 | no |
| `PATCH /api/v1/ticket-categories/:id` | admin | partial + version | 200 | no |
| `DELETE /api/v1/ticket-categories/:id` | admin | — | 204 (sets `is_active=false`) | no |
| `GET /api/v1/tickets` | session | `view=mine|team|all|unassigned`, `status[]?, priority[]?, type?, assigneeId?, teamId?, requesterId?, q?, sort=(createdAt|updatedAt|priority|resolutionDueAt):(asc|desc)`, `cursor, limit` | list of `TicketSummary` (with SLA state) | no |
| `POST /api/v1/tickets` | session | `{type, title, description, categoryId?, priority?, assetId?, tags?}` | 201 `{ticket}` | no |
| `GET /api/v1/tickets/:id` | session | — | `{ticket, requester, assignee, team, asset, sla, approvals[]}` | no |
| `PATCH /api/v1/tickets/:id` | session (requester: own + fields title/description only) | `{title?, description?, priority?, categoryId?, assigneeId?, teamId?, assetId?, tags?}` + version | `{ticket}` | no |
| `POST /api/v1/tickets/:id/transition` | session (requester: own → `cancelled`/`closed` only) | `{status, note?}` + version | `{ticket}` | no |
| `DELETE /api/v1/tickets/:id` | admin | — | 204 (soft) | no |
| `GET /api/v1/tickets/:id/comments` | session | `cursor, limit` | list of `Comment` (private notes stripped for requester) | no |
| `POST /api/v1/tickets/:id/comments` | session | `{body, isPrivate?}` (requester: `isPrivate` forbidden) | 201 `{comment}` | no |
| `GET /api/v1/tickets/:id/timeline` | session | `cursor, limit` | list of `AuditEvent` (requester: staff-only actions filtered) | no |
| `POST /api/v1/tickets/:id/approvals` | session | `{approverId, title?, note?}` | 201 `{approval}`; ticket → `pending_approval` | no |
| `GET /api/v1/sla-policies` | session | — | list | no |
| `POST /api/v1/sla-policies` | manager | `{name, priority, responseMinutes, resolutionMinutes, businessHours, isDefault?}` | 201 | no |
| `GET /api/v1/sla-policies/:id` | session | — | 200 | no |
| `PATCH /api/v1/sla-policies/:id` | manager | partial + version | 200 (open tickets under the policy get due dates recomputed) | no |
| `DELETE /api/v1/sla-policies/:id` | manager | — | 204 (409 `IN_USE` if a category references it) | no |
| `GET /api/v1/approvals` | session | `view=inbox|requested|all` (`all` manager+), `status?, cursor, limit` | list | no |
| `POST /api/v1/approvals` | agent | `{subjectType, subjectId, approverId, title, note?, expiresAt?}` | 201 | no |
| `GET /api/v1/approvals/:id` | session | — | `{approval, subject}` | no |
| `POST /api/v1/approvals/:id/decide` | session (approver only) | `{decision: approved|rejected, note?}` | 200 `{approval}` | no |
| `DELETE /api/v1/approvals/:id` | session (requester of it, or admin) | — | 204 (status → cancelled) | no |
| `GET /api/v1/changes` | agent | `status[]?, risk?, q?, sort?, cursor, limit` | list | no |
| `POST /api/v1/changes` | agent | `{title, description, risk, ticketId?, assetId?, plannedStart?, plannedEnd?, implementationPlan, rollbackPlan}` | 201 (status `draft`) | no |
| `GET /api/v1/changes/:id` | agent | — | `{change, approvals[], ticket?, asset?}` | no |
| `PATCH /api/v1/changes/:id` | agent | partial + version (only in `draft`/`submitted`/`rejected`) | 200 | no |
| `POST /api/v1/changes/:id/transition` | agent (approve/reject: manager via approvals) | `{status, note?, approverId?}` + version | 200 | no |
| `DELETE /api/v1/changes/:id` | admin | — | 204 (soft) | no |
| `GET /api/v1/changes/:id/timeline` | agent | `cursor, limit` | list of `AuditEvent` | no |
| `GET /api/v1/assets` | agent (requester: `view=mine` only) | `view=all|mine`, `type?, status?, q?, assignedToUserId?, sort?, cursor, limit` | list | no |
| `POST /api/v1/assets` | agent | `{tag, name, type, status?, serial?, model?, location?, purchaseDate?, warrantyEnd?, notes?}` | 201 | no |
| `GET /api/v1/assets/:id` | session (requester: assigned to self) | — | `{asset, assignedTo?, openTickets[]}` | no |
| `PATCH /api/v1/assets/:id` | agent | partial + version | 200 | no |
| `POST /api/v1/assets/:id/assign` | agent | `{userId: uuid|null, ticketId?}` + version | 200 `{asset}` (status ↔ assigned/in_stock; ticket linked) | no |
| `DELETE /api/v1/assets/:id` | admin | — | 204 (soft; status retired) | no |
| `GET /api/v1/assets/:id/timeline` | agent | `cursor, limit` | list of `AuditEvent` | no |
| `GET /api/v1/projects` | agent | `status?, ownerId?, q?, sort?, cursor, limit` | list (with progress %) | no |
| `POST /api/v1/projects` | agent | `{name, description?, ownerId?, startDate?, dueDate?}` | 201 | no |
| `GET /api/v1/projects/:id` | agent | — | `{project, milestones[], tasks[], owner}` | no |
| `PATCH /api/v1/projects/:id` | agent | partial (+ `status`) + version | 200 | no |
| `DELETE /api/v1/projects/:id` | admin | — | 204 (soft) | no |
| `POST /api/v1/projects/:id/milestones` | agent | `{name, dueDate?}` | 201 | no |
| `PATCH /api/v1/projects/:id/milestones/:milestoneId` | agent | `{name?, dueDate?, status?}` + version | 200 | no |
| `DELETE /api/v1/projects/:id/milestones/:milestoneId` | agent | — | 204 | no |
| `GET /api/v1/projects/:id/tasks` | agent | `status?, assigneeId?` | list | no |
| `POST /api/v1/projects/:id/tasks` | agent | `{title, description?, milestoneId?, assigneeId?, priority?, dueDate?, dependsOnTaskId?}` | 201 | no |
| `PATCH /api/v1/projects/:id/tasks/:taskId` | agent | partial (+ `status`) + version | 200 | no |
| `DELETE /api/v1/projects/:id/tasks/:taskId` | agent | — | 204 | no |
| `GET /api/v1/tasks` | session | `view=mine|all`, `status?, cursor, limit` | list (cross-project) | no |
| `GET /api/v1/kb/articles` | session | `q?, category?, status?` (requester: forced `published`), `cursor, limit` | list (ranked when `q`) | no |
| `POST /api/v1/kb/articles` | agent | `{title, slug?, body, category?, tags?, status?}` | 201 | no |
| `GET /api/v1/kb/articles/:idOrSlug` | session | — | `{article, author}` (increments `view_count` for published) | no |
| `PATCH /api/v1/kb/articles/:idOrSlug` | agent | partial (+ `status`) + version | 200 | no |
| `DELETE /api/v1/kb/articles/:idOrSlug` | admin | — | 204 (soft) | no |
| `GET /api/v1/automation/rules` | manager | — | list | no |
| `POST /api/v1/automation/rules` | admin | `{name, trigger, conditions, actions, isActive?, schedule?}` | 201 | no |
| `GET /api/v1/automation/rules/:id` | manager | — | 200 | no |
| `PATCH /api/v1/automation/rules/:id` | admin | partial + version | 200 | no |
| `DELETE /api/v1/automation/rules/:id` | admin | — | 204 | no |
| `GET /api/v1/automation/runs` | manager | `ruleId?, cursor, limit` | list | no |
| `GET /api/v1/automation/recurring` | manager | — | list | no |
| `POST /api/v1/automation/recurring` | admin | `{title, description?, kind, template, interval, nextRunAt}` | 201 | no |
| `PATCH /api/v1/automation/recurring/:id` | admin | partial + version | 200 | no |
| `DELETE /api/v1/automation/recurring/:id` | admin | — | 204 | no |
| `GET /api/v1/notifications` | session | `unreadOnly?, cursor, limit` | list + `unreadCount` | no |
| `PATCH /api/v1/notifications/:id` | session (own) | `{read: true}` | 200 | no |
| `POST /api/v1/notifications/read-all` | session | — | 204 | no |
| `GET /api/v1/events/stream` | session | header `Last-Event-ID?` | `text/event-stream` | no |
| `GET /api/v1/dashboard` | session | — | role-shaped `{kpis, myTickets, myApprovals, teamQueue, unassigned, slaAtRisk, myTasks, projects}` | no |
| `GET /api/v1/analytics/overview` | manager | `range=7d|30d|90d`, `teamId?` | `{mttrMinutes, firstResponseMinutes, slaCompliancePct, openByPriority, backlogTrend[], workloadByAgent[], firstContactResolutionPct, projectHealth, ticketsByType}` | no |
| `GET /api/v1/search` | session | `q (2–100)` | `{tickets[], assets[], articles[], changes[], projects[], users[]}` (≤ 5 each; role-filtered) | no |
| `GET /api/v1/audit` | manager | `entityType?, entityId?, actorId?, action?, from?, to?, cursor, limit` | list of `AuditEvent` | no |

Total: 96 endpoint rows — 2 system probes (`internal: yes`) + 94 screen-bound endpoints.

**Common validation rules.** Strings are trimmed; empty after trim = missing. `limit` outside 1–100 → `400 VALIDATION_FAILED`. Unknown query filter or sort field → `400 VALIDATION_FAILED` (`code: UNKNOWN_FIELD`). Unknown body keys → `400` (strict objects). UUID params must match v4 format else `404` (prevents probing). `If-Match` absent on a versioned PATCH → `428 PRECONDITION_REQUIRED`.

---

## 4. Security requirements

Each requirement has an id (`SEC-n`) traced by COMPLIANCE.md. `compliance:` tags mark rows that map to WEB.md contracts.

| ID | Requirement | Control |
|---|---|---|
| SEC-1 | Tenant isolation | All tenant tables RLS-forced; `withOrgScope` sets `app.org_id` via `SET LOCAL`; services filter by `org_id` explicitly; cross-org → 404. INV-1, INV-18 |
| SEC-2 | Object-level authorization | `load<Entity>` middleware on every `/:id` and sub-resource route calls `assertResourceAccess(row, principal, action)`; requester sees own tickets/approvals/assigned assets only; INV-14 |
| SEC-3 | Field-level gating | `serialize(row, principal)` strips private notes, other users' emails (requester sees names only), audit trail entries of staff-only actions |
| SEC-4 | Password verifier | 15–128 chars, Unicode, no composition rules, HIBP k-anonymity screening (fail-open logged), bundled top-10k blocklist fallback; Argon2id m=19456,t=2,p=1; INV-2 |
| SEC-5 | Anti-enumeration | Login: identical 401 envelope for unknown email / wrong password / inactive user with dummy argon2 verify on the miss path; register with an existing email → 201-shaped `202 {}`? No — register creates a new **org**, so email uniqueness is disclosed only after the rate limiter; response is `409 EMAIL_TAKEN` (accepted: org self-registration is a public sign-up and the per-IP limiter (5/h) makes enumeration impractical — ADR-022). Forgot-password always 202. |
| SEC-6 | Rate limiting (`rate_limiting: true`) | `/auth/login` 10 per 15 min per `(ip, email)`; `/auth/register` 5/h per ip; `/auth/password/*` 5/h per `(ip, email)`; API 300/min per session, 60/min per ip unauthenticated; SSE 5 concurrent streams per user; limiter before hash verify (INV-8); `draft-8` headers; 429 uses the §7 envelope |
| SEC-7 | Session management | Opaque 256-bit token, HMAC-SHA256 stored; cookie `__Host-orbit_session; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=2592000`; sliding expiry; rotated on password change and role change (other sessions revoked); logout deletes row |
| SEC-8 | CSRF | `SameSite=Strict` + `Origin`/`Referer` must equal `APP_ORIGIN` on POST/PATCH/PUT/DELETE (403 `CSRF_ORIGIN_MISMATCH`); `GET` never mutates |
| SEC-9 | Transport & headers `compliance: WEB.md §6` | HTTPS only (Caddy 8443, HSTS 2 y preload, 8080 redirect-only); CSP Profile N (nonce + strict-dynamic) from `proxy.ts` on HTML; helmet on API; `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `X-Frame-Options: DENY`, COOP `same-origin`; zero CSP violations measured |
| SEC-10 | Input validation | zod strict schemas at every boundary with the bounds in §2; body ≤ 256 KB; header buffer 32 KB; §7 envelope with field errors |
| SEC-11 | Injection | Parameterized queries only (Drizzle); tsvector search via `plainto_tsquery`; Markdown rendered with a sanitizing renderer (no raw HTML); React escaping |
| SEC-12 | Audit logging (`audit_logging: true`) | Every create/update/transition/assign/comment/approve/reject/delete/role change/login/logout writes `audit_events` in the same transaction; append-only (INV-5, RLS grants) |
| SEC-13 | Logging hygiene | pino redaction of `password, newPassword, currentPassword, token, secret, authorization, cookie, set-cookie, email, to_address, passwordHash`; no PII in access logs (user id only); no stack traces in responses |
| SEC-14 | Secrets | Env-only; `.env.example` committed; production requires `SESSION_SECRET ≥ 32` chars with no default; INV-16 |
| SEC-15 | Concurrency safety | `version` optimistic checks; approvals decided with `WHERE status='pending'`; per-org counters via row lock; scheduler advisory lock; `SKIP LOCKED` sweeps; multi-row task locks ordered by id |
| SEC-16 | Egress | No user-controlled URLs fetched; SMTP and HIBP hosts fixed by env (no SSRF surface) |
| SEC-17 | Least privilege DB | `orbit_app` role NOBYPASSRLS with table-level grants only; migrations run as owner in a separate one-shot container |
| SEC-18 | Supply chain (OWASP 2025 A03) | Lockfile committed; `npm ci`; pinned base image tags; `npm audit` in the Security Audit Gate |
| SEC-19 | Accessibility `compliance: WEB.md §2` | WCAG 2.2 AA measured per screen/viewport/theme (FRONTEND-AUDIT.json) |
| SEC-20 | SEO & metadata `compliance: WEB.md §3` | Public screens carry title/description/OG/canonical; app screens `noindex`; robots, sitemap, manifest, icons, designed 404 |
| SEC-21 | Fail-fast config | zod env schema exits ≠ 0 naming the variable; INV-4 |
| SEC-22 | Graceful shutdown | SIGTERM drains ≤ 10 s, closes pool, stops scheduler, exits 0 |
| SEC-23 | Data at rest | Delegated to deploy volume encryption (⚠ delegated); documented in QUICKSTART |

**RBAC matrix** (✓ allowed, own = only own resources, — denied):

| Capability | requester | agent | manager | admin |
|---|---|---|---|---|
| Create ticket | ✓ | ✓ | ✓ | ✓ |
| Read tickets | own | ✓ | ✓ | ✓ |
| Edit ticket fields | own (title/desc) | ✓ | ✓ | ✓ |
| Transition ticket | own → cancelled/closed | ✓ | ✓ | ✓ |
| Private notes | — | ✓ | ✓ | ✓ |
| Delete (soft) | — | — | — | ✓ |
| Request approval | on own ticket via category | ✓ | ✓ | ✓ |
| Decide approval | if approver | if approver | if approver | if approver |
| Cancel approval | if requester | if requester | ✓ | ✓ |
| Changes | — | ✓ | ✓ | ✓ |
| Assets read | assigned to self | ✓ | ✓ | ✓ |
| Assets write | — | ✓ | ✓ | ✓ |
| Projects/tasks | own tasks (read) | ✓ | ✓ | ✓ |
| KB read | published | ✓ | ✓ | ✓ |
| KB write | — | ✓ | ✓ | ✓ |
| SLA policies | read | read | ✓ | ✓ |
| Users/teams manage | — | read | read | ✓ |
| Catalog manage | — | — | — | ✓ |
| Automation | — | — | read | ✓ |
| Analytics | — | — | ✓ | ✓ |
| Audit read | — | — | ✓ | ✓ |
| Notifications | own | own | own | own |

**Memorized-secret checklist (BUILD.md):** breached-password screening ✓ (SEC-4); length not composition ✓ (15–128); rate-limit before hash ✓ (SEC-6, INV-8); no forced rotation, no knowledge-based recovery ✓ (email-link reset via outbox).

**`has_email` checklist:** `pending_emails` outbox ✓ (§2), insert in triggering transaction ✓ (F-06), drain worker ✓ (F-06/F-12 scheduler), capped backoff 1 m / 10 m / 1 h then `dlq` ✓, never synchronous send ✓, uniqueness `(to_address, template_key, business_ref_id)` ✓.

---

## 5. Features & Key Workflows

### 5.1 Feature list (build order = dependency order)

Each feature is a vertical slice: endpoints + screens + tests. Acceptance criteria are testable statements; "AC-n" numbering is per feature.

#### F-01 — Foundation & service floor
Monorepo (`apps/api`, `apps/web`, `packages/shared`), TypeScript 6 strict, ESLint 10 flat config (typescript-eslint type-checked, jsx-a11y, react-hooks), Prettier, `.editorconfig`, vitest workspace, Express app factory with the §5.2 middleware chain of ARCHITECTURE.md, `requestId`, pino JSON logging with redaction, helmet, body limit, CSRF origin guard, rate-limit factory (`ipKeyGenerator`), §7 `problem()` mapper, env schema (INV-4), `GET /api/health`, graceful shutdown, OpenAPI generation from zod.
- AC-1 `npm run typecheck && npm run lint && npm run format:check` exit 0 on the empty scaffold.
- AC-2 Starting with `DATABASE_URL` unset exits non-zero and prints `DATABASE_URL`.
- AC-3 `GET /api/health` responds ≤ 500 ms with `checks.db`; 503 when the DB is unreachable.
- AC-4 An unknown route returns the §7 envelope with `code: NOT_FOUND`; a thrown error returns 500 with no stack and the `requestId` echoed in `X-Request-Id`.
- AC-5 A POST without a matching `Origin` returns 403 `CSRF_ORIGIN_MISMATCH`.
- AC-6 SIGTERM during an in-flight request: request completes 2xx, process exits 0 within 10 s.

#### F-02 — Data layer, tenancy & audit core
Drizzle schema for every §2 table, migrations `0001_init.sql` + `0002_rls.sql` (+ `.down.sql`), `orbit_app` role, `withOrgScope`, `audit.record(tx, …)`, `publish()` SSE hub core (`lib/sse.ts`, no route yet), scheduler skeleton with advisory lock, seed (categories, asset types, default SLA policies; demo org when `SEED_DEMO=true`), transition tables in `packages/shared`.
- AC-1 Migrate + seed run twice without error (idempotent seed).
- AC-2 Integration test: with `app.org_id` = org A, a query for org B's ticket returns zero rows even without an explicit filter.
- AC-3 After a transaction commits, a new query on the same pooled connection without `SET LOCAL` returns zero tenant rows (fail-closed).
- AC-4 `UPDATE`/`DELETE` on `audit_events` as `orbit_app` is rejected by Postgres.
- AC-5 Two concurrent ticket creations in one org receive distinct consecutive numbers.
- AC-6 Every down migration applies cleanly on top of its up migration in CI.

#### F-03 — Design system, app shell & landing
Tokens (`tokens.css` light + dark, Tailwind `@theme`), fonts (Manrope display, Inter body, JetBrains Mono), primitives (`Button` with `data-primary`, `Input`, `Select`, `Textarea`, `Checkbox`, `Badge`, `StatusPill`, `PriorityPill`, `Card`, `KpiCard`, `Table`/`TableWrap`, `Dialog`, `Drawer`, `Toast`, `Skeleton`, `EmptyState`, `ErrorState`, `Tabs`, `Kbd`, `Avatar`, `Timeline`), app shell (sidebar 224 px → drawer < md with a 44 px `aria-expanded`/`aria-controls` toggle, focus trap, Escape), skip link, theme toggle (system/light/dark persisted in `localStorage` + `data-theme`, follows `prefers-color-scheme`), `proxy.ts` CSP Profile N + header set, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `robots.ts`, `sitemap.ts`, `manifest.ts`, `icon.tsx`, `opengraph-image.tsx`, landing page `/` (asymmetric hero with a live product surface mock, three-column "connected objects" story, single proof band, CTA to `/register`, footer with MDLC attribution).
- AC-1 Production build renders `/` with zero `securitypolicyviolation` events and the WEB.md §6 header set.
- AC-2 At 375 px the nav toggle opens a drawer containing every primary destination; Escape closes it and returns focus.
- AC-3 Theme toggle persists across reload and both themes pass axe serious/critical = 0 on `/`.
- AC-4 Primitives have state tests (default/disabled/loading/error) in `components/ui/*.test.tsx`.
- AC-5 Landing does not match the template landing skeleton (DESIGN.md §8.11): no eyebrow pill + accent-clause headline + ghost/filled CTA pair + 3-stat row combination.

#### F-04 — Registration & login sessions
`POST /auth/register`, `/auth/login`, `/auth/logout`, `GET /auth/me`; `authenticate` middleware; `requireRole`; screens `/register`, `/login`; `(app)` layout gate (unauthenticated → `/login?next=`); session cookie.
- AC-1 Register creates org + admin user + `org_counters` row in one transaction and sets the cookie; the response never includes `password_hash`.
- AC-2 Login with wrong password, unknown email, or inactive user returns the identical 401 body; timing difference between unknown-email and wrong-password paths < 50 ms median over 20 runs (dummy verify).
- AC-3 11th login attempt in 15 min for `(ip,email)` returns 429 before any hash verify (spy asserts `verifyPassword` not called).
- AC-4 Password of 14 chars → 400 with `errors[0].field = "password"`, `code: TOO_SHORT`; a password in the top-10k list → `code: BREACHED`.
- AC-5 Login form maps field errors onto inputs (`aria-invalid`, `aria-describedby`) and keeps the email value.
- AC-6 `/dashboard` unauthenticated redirects to `/login?next=/dashboard`, and after login returns to `/dashboard`.

#### F-05 — Profile settings & password change
`PATCH /auth/me`, `POST /auth/me/password`; screen `/settings/profile` (name, theme, password change).
- AC-1 Password change requires the current password; on success every other session row for the user is deleted and the current session token is rotated.
- AC-2 Theme choice persists server-side (`users.theme`) and is applied on next login on another device.
- AC-3 Wrong current password → 400 `errors[0].field="currentPassword"`, rendered on the control.

#### F-06 — Password reset & invites (outbox + mailer)
`POST /auth/password/forgot`, `/auth/password/reset`; `pending_emails` writer + drain job + Nodemailer SMTP transport (JSON transport when `SMTP_URL` unset); templates `password_reset`, `user_invite`, `ticket_assigned`, `approval_requested`, `sla_breached`; screens `/forgot-password`, `/reset-password/[token]`.
- AC-1 Forgot with unknown email returns 202 and enqueues nothing; with a known email enqueues exactly one row (unique constraint absorbs a repeat within the token TTL).
- AC-2 Reset token is single-use, 1 h TTL; reuse → 400 `TOKEN_INVALID`; success revokes all sessions and clears `must_reset_password`.
- AC-3 Drain job: a failing transport moves the row `pending → failed (attempt 1, next in 1 m) → … → dlq` after 3 attempts with `error_text` set; a structured `warn` log is emitted at `dlq`.
- AC-4 The reset screen renders `TOKEN_INVALID` as a human message with a link to request a new one.

#### F-07 — Users admin
`GET/POST /users`, `GET/PATCH/DELETE /users/:id`; screen `/admin/users` (table: search, role filter, active filter, sort; create dialog; edit drawer; deactivate confirm dialog naming the user).
- AC-1 Creating a user queues `user_invite` with a 24 h invite token and `must_reset_password=true`; the user cannot log in with any password until reset.
- AC-2 Role change writes `audit_events(action='role_changed', before, after)` and revokes the user's sessions.
- AC-3 An admin cannot deactivate or demote themselves (409 `SELF_MODIFICATION`).
- AC-4 Requester calling `GET /users` gets 404 (route hidden), agent gets names + roles but no emails.

#### F-08 — Teams admin
Teams CRUD + membership; screen `/admin/teams` (list, create dialog, detail drawer with member add/remove).
- AC-1 Deleting a team NULLs `tickets.team_id` and `ticket_categories.default_team_id` referencing it (in one transaction) and audits `deleted`.
- AC-2 Adding a user from another org → 404.
- AC-3 Duplicate team name → 409 `DUPLICATE` with `errors[0].field="name"` shown on the dialog input.

#### F-09 — Ticket catalog & creation
`ticket-categories` CRUD; `POST /tickets`; screens `/admin/catalog` (categories table + dialog with type, default team/priority, SLA policy, requires-approval), `/tickets/new` (two-step: pick type/category cards → form with title, description, priority (agents+), asset picker (search), tags; KB suggestions appear as the title is typed — W2 deflection).
- AC-1 Creating a ticket assigns `number = org_counters.tickets + 1`, applies category defaults (team, priority, SLA policy), computes `response_due_at`/`resolution_due_at` from the policy's business hours, writes `audit_events(created)`, publishes `ticket.created`, and runs `ticket.created` automation rules.
- AC-2 Category with `requires_approval` → ticket status `pending_approval` and an approval row addressed to the requester's team lead (fallback: any manager; if none, ticket opens as `new` and a `warn` is logged) — W4.1.
- AC-3 Title < 3 chars → 400 with the field error rendered under the title input; description over 20 000 chars → 400.
- AC-4 Requester cannot set `priority` (ignored → 400 `FORBIDDEN_FIELD`), cannot pick an asset not assigned to them.
- AC-5 Typing a title of ≥ 4 chars shows up to 3 matching published KB articles inline within 300 ms (debounced).

#### F-10 — Ticket queue & detail
`GET /tickets`, `GET/PATCH/DELETE /tickets/:id`; screens `/tickets` (queue: view tabs mine/team/all/unassigned, status & priority multi-filters, type, assignee, search, sort, cursor pagination "Load more", row click → detail, bulk select with bulk assign/priority (agent+), SLA countdown chips, empty/loading/error/zero-results states), `/tickets/[id]` (header with number/status/priority/SLA, description, properties panel with inline edit of priority/assignee/team/category/asset/tags, requester card, linked asset card, approvals list).
- AC-1 Requester `view=all` returns only own tickets (server-forced); `view=team` for a user with no team → `0 matched` explanation.
- AC-2 `PATCH` with stale `If-Match` → 409 `VERSION_CONFLICT`; the UI shows "This ticket changed — reload" with a reload action and keeps the user's edits in the form.
- AC-3 Assignee change increments `reassignment_count` when a previous assignee existed, audits `assigned`, notifies the new assignee (in-app + `ticket_assigned` email), publishes `ticket.updated`.
- AC-4 Sorting by an unknown field → 400; `limit=101` → 400.
- AC-5 Soft-deleted ticket → 404 for non-admins; admin sees it with `includeDeleted=true` and a "deleted" badge.
- AC-6 Bulk assign of 3 selected tickets issues 3 PATCHes and shows one toast summarising successes/failures.

#### F-11 — Ticket lifecycle: transitions, comments, timeline
`POST /tickets/:id/transition`, comments list/create, timeline, `POST /tickets/:id/approvals`; detail screen sections (transition buttons driven by the transition table, comment composer with private-note toggle for staff, timeline merging comments and audit events).
Ticket transitions (pre → post; guard):

| From | To | Guard / side effects |
|---|---|---|
| new | open | agent+; sets `first_responded_at` if null |
| new, open, in_progress, waiting_on_requester | in_progress | agent+; `assignee_id` required (400 `ASSIGNEE_REQUIRED`); resumes SLA if paused |
| open, in_progress | waiting_on_requester | agent+; pauses SLA (`sla_paused_at = now()`) |
| waiting_on_requester | open | requester comment auto-resumes, or agent+; adds pause to `sla_paused_ms`, shifts due dates |
| new, open, in_progress, waiting_on_requester | pending_approval | via `POST /tickets/:id/approvals`; SLA paused |
| pending_approval | open | approval decided `approved` (system) |
| pending_approval | cancelled | approval `rejected` (system) with decision note copied to a private comment |
| open, in_progress, waiting_on_requester | resolved | agent+; `resolved_at = now()`; resolution note required (≥ 3 chars); SLA stops |
| resolved | closed | requester (own) or agent+; or automatic after 5 days via scheduler; `closed_at` |
| resolved | open | requester reopen within 30 days ("not fixed") or agent+; clears `resolved_at`, SLA resumes |
| new, open, in_progress, waiting_on_requester, pending_approval | cancelled | requester (own) or agent+; note optional |
| closed, cancelled | — | terminal; any transition → 409 `INVALID_TRANSITION` |

- AC-1 Every allowed transition passes an integration test; every disallowed pair (exhaustive matrix) returns 409 `INVALID_TRANSITION` with `errors[0].code = "FROM_TO"` naming both states.
- AC-2 First agent comment (non-private) sets `first_responded_at` and stops the response clock; a private note does not.
- AC-3 Requester never receives private notes (list, timeline, SSE payloads) — asserted by a same-org cross-user test on `/comments`, `/timeline` and the stream.
- AC-4 Comment from the requester while `waiting_on_requester` transitions the ticket back to `open` and audits both events.
- AC-5 Concurrent transitions (two clients, same version) → one 200, one 409.

#### F-12 — SLA policies & engine
SLA CRUD; `lib/slaClock.ts` (pure: `addBusinessMinutes(start, minutes, calendar)`, `elapsedBusinessMs`), scheduler engine (breach sweep, auto-close resolved > 5 days, project health recompute), screen `/admin/sla` (table + dialog with business-hours editor and priority/default flags); SLA chips (`on track` / `due in 2h` / `breached`) on queue and detail; dashboard "SLA at risk" feed.
- AC-1 `addBusinessMinutes(Fri 16:30, 60, Mon–Fri 09–17 America/Chicago)` = Mon 09:30 local; DST transition weekends covered by tests (spring forward and fall back).
- AC-2 A ticket whose `resolution_due_at` passed with `resolved_at IS NULL` gets `sla_resolution_breached_at` set within one tick (15 s), an `sla.breached` event, a notification to assignee + team lead, and `sla.breached` rules evaluated; a second tick does not re-fire (idempotent).
- AC-3 Changing a policy's minutes recomputes due dates for open tickets under it (audited as `updated` by system).
- AC-4 Two scheduler instances started concurrently: only one acquires the advisory lock (test spawns two tick functions).
- AC-5 Deleting a policy referenced by a category → 409 `IN_USE` with the category names in `detail`.

#### F-13 — Approvals
Approvals CRUD/decide; screen `/approvals` (tabs: Inbox (pending for me), Requested by me, All (manager+); decide dialog with note; subject link).
Approval transitions: `pending → approved | rejected` (approver only, once; `decided_at`), `pending → cancelled` (requester/admin), `pending → cancelled` automatically when `expires_at` passes (scheduler). Decision side effects: subject `ticket` → transition per F-11; subject `change` → `submitted → approved|rejected`; subject `project`/`purchase` → audit only + notification.
- AC-1 A non-approver calling `/decide` → 404 (existence not revealed); the approver deciding twice → 409 `ALREADY_DECIDED`.
- AC-2 Approver must differ from `requested_by` → 400 `SELF_APPROVAL`.
- AC-3 Deciding writes `audit_events(approved|rejected)`, notifies the requester (in-app + email), publishes `approval.decided`, runs `approval.decided` rules — all in one transaction (email via outbox).
- AC-4 Inbox shows pending items first with subject type badge; empty inbox explains "Nothing waiting on you".

#### F-14 — Change management
Changes CRUD + transition + timeline; screens `/changes` (table with status/risk filters, create dialog), `/changes/[id]` (plan/rollback tabs, schedule window, approvals, linked ticket/asset, transition actions).
Change transitions:

| From | To | Guard |
|---|---|---|
| draft | submitted | requester/agent; `rollback_plan` and `implementation_plan` non-empty; creates approval addressed to `approverId` (manager+) |
| submitted | approved / rejected | via approval decision only (manager) |
| rejected | draft | agent (revise) |
| approved | scheduled | agent; `planned_start`/`planned_end` required |
| scheduled | implementing | implementer (or agent+) not before `planned_start − 1h` |
| implementing | completed / rolled_back | implementer or agent+; note ≥ 3 chars; `completed_at` |
| draft, submitted, rejected, approved, scheduled | cancelled | requester or manager+ |
| completed, rolled_back, cancelled | — | terminal → 409 |

- AC-1 Submitting without a rollback plan → 400 `errors[0].field="rollbackPlan"` rendered on the textarea.
- AC-2 Attempting `approved` via `/transition` directly → 409 `USE_APPROVAL`.
- AC-3 Full W5 path passes end-to-end with a timeline containing 6 audit events in order.
- AC-4 High-risk changes require `planned_end − planned_start ≤ 24 h` warning in UI (non-blocking) and manager approver.

#### F-15 — Assets
Assets CRUD + assign + timeline; screens `/assets` (table: type/status filters, search, create dialog, warranty-expiring badge), `/assets/[id]` (properties inline edit, assignment card with user picker, open tickets, timeline).
- AC-1 `assign` with a user sets `status=assigned`, `assigned_to_user_id`, audits `assigned` with before/after; with `null` sets `in_stock`; passing `ticketId` links `tickets.asset_id` and adds a private comment "Asset X assigned" (W4.3).
- AC-2 Assigning a retired asset → 409 `INVALID_STATE`; duplicate tag → 409 `DUPLICATE` on the `tag` field.
- AC-3 Requester `view=mine` returns assets assigned to them; requester requesting another asset id → 404.
- AC-4 Warranty ending within 30 days shows an "expiring" badge (computed client-side from `warrantyEnd`).

#### F-16 — Projects, milestones & tasks
Projects/milestones/tasks CRUD, `GET /tasks`; screens `/projects` (cards with progress bar + health, create dialog), `/projects/[id]` (overview with milestone timeline, task board grouped by status with drag-free status select, task drawer with dependency picker, owner/dates edit).
Task transitions: `todo → in_progress → done`, `in_progress → blocked → in_progress`, any → `todo`. Guard: a task cannot move to `in_progress`/`done` while `depends_on_task_id` is not `done` (409 `DEPENDENCY_OPEN` naming the blocking task). Dependency cycles → 400 `DEPENDENCY_CYCLE` (checked by walking up to 50 hops). Project `health`: `late` if any open task is past due or project `due_date` passed; `at_risk` if ≥ 30 % of tasks are blocked/overdue-within-3-days; else `on_track`. Progress = done / total.
- AC-1 Completing the last open task of a milestone marks the milestone `completed` and audits it.
- AC-2 Deleting a project soft-deletes and hides its tasks from `GET /tasks`.
- AC-3 Project status `completed` requires all tasks done or cancelled → 409 `TASKS_OPEN` with count.
- AC-4 Dashboard "My tasks" lists tasks assigned to the principal across projects ordered by due date.

#### F-17 — Knowledge base
KB CRUD + search; screens `/kb` (search box with ranked results, category chips, status filter for staff, "New article" for agents; requester sees published only), `/kb/[slug]` (rendered Markdown with sanitized renderer, TOC for h2/h3, author/date, "Was this helpful" no-op removed → not specced; Edit toggle for agents switches to editor in place; publish/archive actions), `/kb/new` (editor with live preview).
- AC-1 Search `q=vpn` ranks title matches above body matches (ts_rank with weights); results < 300 ms p95 on 1 000 articles (load test seeds 1 000 rows).
- AC-2 Slug auto-derived from title (`kebab-case`, ≤ 120), uniqueness per org → 409 on collision with a suggested alternative in `detail`.
- AC-3 Requester requesting a draft by slug → 404; agent → 200.
- AC-4 Raw HTML in Markdown (`<script>`) is escaped in the rendered output (test asserts no `<script` in DOM).
- AC-5 `view_count` increments only on published reads by a principal other than the author.

#### F-18 — Notifications & realtime
`GET /events/stream` route, client `useEventStream` hook (EventSource with `Last-Event-ID`, exponential retry, fallback 20 s polling when the stream errors 3× in a row), notifications API, bell menu (unread count badge, last 10, mark read), screen `/notifications`, `useResource` revalidation on matching events, toast for `ticket.assigned`/`approval.requested`/`sla.breached` addressed to the principal.
- AC-1 Unauthenticated `/events/stream` → 401 envelope; 6th concurrent stream for a user → 429 `TOO_MANY_STREAMS`.
- AC-2 An event published for org A is never received by org B's stream (integration test with two streams).
- AC-3 A client reconnecting with `Last-Event-ID` receives the missed events from the ring buffer (≤ 500) in order.
- AC-4 The stream writes `: connected` immediately and `: keepalive` every 20 s; `Content-Type: text/event-stream`, `Cache-Control: no-store`.
- AC-5 Assigning a ticket updates the assignee's open `/tickets` queue within 2 s without reload (e2e).

#### F-19 — Automation rules & recurring tasks
Rules CRUD + runs + recurring CRUD; engine (`jobs/automationEngine.ts`): predicate language `field ∈ {type, priority, status, categoryId, teamId, assigneeId, tags, title}`, `op ∈ {eq, neq, in, contains, isEmpty, isNotEmpty}`; actions `assign_team {teamId}`, `assign_user {userId}`, `set_priority {priority}`, `set_status {status}` (through the transition table), `add_tag {tag}`, `notify_user {userId}`, `notify_role {role}`, `request_approval {approverId}`, `create_task {projectId, title}`; screen `/automation` (tabs Rules / Runs / Recurring; rule builder with condition rows and action rows; enable toggle; run log table).
- AC-1 A rule `ticket.created` `categoryId eq X` → `assign_team T` assigns team T and records an `automation_runs` row with `matched=true` and `actions_applied`.
- AC-2 A rule whose action triggers the same trigger recursively stops at depth 3 with `error="MAX_DEPTH"` (no infinite loop; test constructs the loop).
- AC-3 Invalid condition field → 400 `errors[0].field="conditions[0].field"` rendered on the row.
- AC-4 Recurring ticket `daily` at `next_run_at` creates one ticket per day even if the scheduler was down for 3 days (catch-up creates one, then advances to the next future slot — no backlog flood).
- AC-5 Deactivated rule never runs; run log shows the last 50 runs with rule name and outcome.

#### F-20 — Dashboard, analytics & global search
`GET /dashboard`, `GET /analytics/overview`, `GET /search`; screens `/dashboard` (lead metric: open tickets with breach count → supporting KPI row (SLA compliance, MTTR, unassigned, pending approvals) → panels: my work / team queue / SLA at risk / my approvals / projects health; requester variant: my tickets, my approvals, KB search), `/analytics` (range selector 7/30/90 d, team filter, KPI cards with trend, backlog trend chart (SVG, legend, empty state), open-by-priority bars, workload-by-agent table, tickets-by-type, project health list); command palette (`⌘K` / `Ctrl+K`, search results grouped, keyboard navigation, quick actions "New ticket").
- AC-1 MTTR = mean(`resolved_at − created_at − sla_paused_ms`) over resolved tickets in range, in minutes; SLA compliance = resolved within `resolution_due_at` / resolved; both verified by a fixture with known values.
- AC-2 Requester calling `/analytics/overview` → 404; manager → 200 within 500 ms on 5 000 tickets (load test fixture).
- AC-3 Search returns role-filtered results (requester sees only own tickets and published articles); `q` of 1 char → 400.
- AC-4 Dashboard renders skeletons while loading and a designed error region (`role=alert`) with retry when the API fails (baseline probe).

#### F-21 — Audit trail
`GET /audit`; screen `/admin/audit` (filters entity type, action, actor, date range; table with before/after diff drawer; deep link from any entity timeline).
- AC-1 Every action listed in §2 `audit_events.action` appears at least once in the integration suite and is retrievable via `/audit` filters.
- AC-2 Manager sees audit; agent → 404.
- AC-3 Diff drawer shows changed keys only, with redaction of `passwordHash` (never stored in before/after anyway — test asserts absence).

#### F-22 — Delivery: containers, Caddy, CI, docs, e2e
Dockerfiles (api, web, migrate) multi-stage/non-root/HEALTHCHECK/tini, `docker-compose.yml` (+ `docker-compose.override.yml` for dev ports), `Caddyfile` (`localhost:8443`, `tls internal`, HSTS, SSE flush), `.github/workflows/ci.yml` (install → typecheck → lint → format → test (+ services: postgres) → build → web:baseline → lint:invariants), `.env.example`, `QUICKSTART.md` (with "Protocol & TLS"), `README.md` (previews, Built with), `NOTICE`, Playwright config (Desktop Chrome + Pixel 7, production webServer), e2e specs `W1`–`W9`, `web-baseline.config.json` covering every §6 screen, `smoke-test.sh` + `SMOKE-TEST.md`.
- AC-1 `docker compose up -d` from a clean checkout reaches `https://localhost:8443/api/health` 200 within 90 s.
- AC-2 A request with a 20 KB `Cookie` header returns 200 on `/` and `/tickets`.
- AC-3 `npm run test:e2e` passes all nine `W<n>` specs on both projects against the production stack.
- AC-4 `ci.yml` runs the exact Verification Gate sequence.

### 5.2 Key Workflows (1:1 with RESEARCH.md; terminal step named)

**W1 — Org onboarding** (admin)
- W1.1 Visit `/register`; submit org name, slug, name, email, password → org + admin created, signed in, lands on `/dashboard` (empty-state onboarding card).
- W1.2 Open `/admin/teams` → create team "Service Desk".
- W1.3 Open `/admin/users` → create user (agent) with team → invite email queued.
- W1.4 Create a second user (requester).
- W1.5 **Terminal:** `/admin/users` lists both users with roles and team; `/admin/audit` shows `created` events for org, team, users.

**W2 — Self-service request** (requester)
- W2.1 Log in at `/login` → `/dashboard` requester variant.
- W2.2 `/kb` search "vpn" → results (or zero-results explanation with "Open a ticket" action).
- W2.3 `/tickets/new` → choose "Incident" → title/description → inline KB suggestions shown → submit → ticket `#N` created, status `new`, SLA due dates shown.
- W2.4 `/tickets/[id]` → add a comment; see status change to `open` when an agent responds (live via SSE).
- W2.5 Agent resolves; requester sees `resolved` and a "Confirm & close" action.
- W2.6 **Terminal:** requester clicks "Confirm & close" → status `closed`; timeline shows the full history.

**W3 — Agent queue & SLA** (agent)
- W3.1 `/tickets?view=team` → queue with SLA chips, sort by resolution due.
- W3.2 Open a `new` ticket → assign to self (inline) → transition `open` (first response recorded).
- W3.3 Add a public comment → `first_responded_at` set; response clock stops.
- W3.4 Transition `waiting_on_requester` → SLA paused; requester comments → auto `open`, clock resumes.
- W3.5 Scheduler marks an overdue ticket breached → agent receives a toast + notification; ticket shows "breached".
- W3.6 **Terminal:** transition `resolved` with a resolution note; queue chip shows "resolved"; dashboard breach count decrements.

**W4 — Connected fulfillment** (requester → manager → agent)
- W4.1 Requester `/tickets/new` → category "Laptop request" (requires approval) → ticket `pending_approval`; approval addressed to the manager.
- W4.2 Manager `/approvals` inbox → approve with note → ticket `open`; requester notified.
- W4.3 Agent `/assets` → open an `in_stock` laptop → assign to requester with `ticketId` → asset `assigned`, ticket linked, private comment added.
- W4.4 Agent transitions ticket `in_progress` → `resolved` ("Laptop delivered").
- W4.5 Requester closes.
- W4.6 **Terminal:** `/assets/[id]` timeline shows `assigned` with before/after; `/tickets/[id]` shows linked asset and approval; `/admin/audit` filter by ticket shows every event.

**W5 — Change management** (agent → manager → implementer)
- W5.1 `/changes` → create change (risk high, plans, linked ticket/asset).
- W5.2 `/changes/[id]` → "Submit for approval" choosing a manager → status `submitted`; approval created.
- W5.3 Manager approves in `/approvals` → change `approved`.
- W5.4 Agent schedules (start/end) → `scheduled`.
- W5.5 Implementer starts → `implementing`.
- W5.6 **Terminal:** "Complete" (or "Roll back") with note → `completed`/`rolled_back`; timeline shows 6 audited steps.

**W6 — Project delivery** (manager)
- W6.1 `/projects` → create project with owner and due date.
- W6.2 `/projects/[id]` → add two milestones.
- W6.3 Add tasks with assignees, due dates and one dependency.
- W6.4 Move a dependent task to `in_progress` before its dependency is done → blocked with the named reason; complete the dependency → then allowed.
- W6.5 **Terminal:** complete all tasks of milestone 1 → milestone auto-completes; `/projects` card shows progress % and health; `/dashboard` shows the project.

**W7 — Knowledge publishing** (agent → requester)
- W7.1 `/kb/new` → write article (Markdown) → save draft.
- W7.2 Preview → publish → `/kb/[slug]` public to the org.
- W7.3 Requester `/kb` search by a synonym in the body → article found; opens it.
- W7.4 **Terminal:** requester starts `/tickets/new` and sees the article suggested inline; `view_count` incremented.

**W8 — Automation** (admin)
- W8.1 `/automation` → create rule: trigger `ticket.created`, condition `categoryId eq "Network"`, actions `assign_team Network`, `set_priority high`.
- W8.2 Create rule: trigger `sla.breached`, action `notify_role manager`.
- W8.3 Create a Network ticket (as requester) → rule 1 fires; ticket assigned to Network team with priority high.
- W8.4 Runs tab shows the run with `matched=true` and actions applied.
- W8.5 **Terminal:** `/admin/audit` shows `rule_fired` + `assigned` + `updated` events for the ticket.

**W9 — Leadership visibility** (manager)
- W9.1 `/analytics` → range 30 d → KPIs (MTTR, SLA %, backlog trend, workload).
- W9.2 Filter by team → values change; chart legend and empty state verified with a team that has no tickets.
- W9.3 `/dashboard` → lead metric and SLA-at-risk feed.
- W9.4 **Terminal:** `/admin/audit` → filter by actor → open a before/after diff drawer for a ticket update.

---

## 6. UI Surface

| Screen | Route | Example | Roles | Binds | Workflow steps | User controls | States |
|---|---|---|---|---|---|---|---|
| Landing | `/` | `/` | public | — | W1.1 (entry) | theme toggle, "Get started" CTA, "Sign in" link | success, reduced-motion still |
| Login | `/login` | `/login` | public | `POST /api/v1/auth/login` | W2.1 | email, password, submit, forgot link, `next` redirect | loading (submit pending), error (field-level + generic), success (redirect) |
| Register | `/register` | `/register` | public | `POST /api/v1/auth/register` | W1.1 | org name, slug (auto-suggested, editable), name, email, password (strength meter, breach message), submit | loading, error (field-level incl. slug taken), success |
| Forgot password | `/forgot-password` | `/forgot-password` | public | `POST /api/v1/auth/password/forgot` | — (F-06) | email, submit | loading, success (always "If that email exists…"), error (network) |
| Reset password | `/reset-password/[token]` | `/reset-password/abc123` | public | `POST /api/v1/auth/password/reset` | — (F-06) | new password, submit | loading, error (invalid/expired token with "request new" link; field errors), success (→ login) |
| Dashboard | `/dashboard` | `/dashboard` | requester, agent, manager, admin | `GET /api/v1/dashboard`, `GET /api/v1/tasks`, `PATCH /api/v1/notifications/:id` | W1.1, W2.1, W3.6, W6.5, W9.3 | role-shaped panels, "New ticket" quick action, panel links, notification bell | empty (onboarding card / "nothing needs you"), loading (skeleton grid), error (`role=alert` + retry), success |
| Ticket queue | `/tickets` | `/tickets?view=team` | requester, agent, manager, admin | `GET /api/v1/tickets`, `PATCH /api/v1/tickets/:id`, `GET /api/v1/users`, `GET /api/v1/teams` | W3.1, W3.6 | view tabs (mine/team/all/unassigned), status[] , priority[], type, assignee, team, search, sort, limit/load more, bulk select + bulk assign/priority, row open | empty (first-use vs 0 matched), loading (skeleton rows), error, success |
| New ticket | `/tickets/new` | `/tickets/new` | requester, agent, manager, admin | `GET /api/v1/ticket-categories`, `POST /api/v1/tickets`, `GET /api/v1/kb/articles`, `GET /api/v1/assets` | W2.3, W4.1, W7.4, W8.3 | type, category, title, description, priority (staff), asset picker (search), tags, submit | loading (categories skeleton), error (field-level), success (→ detail), suggestions empty/loaded |
| Ticket detail | `/tickets/[id]` | `/tickets/00000000-0000-4000-8000-000000000001` | requester (own), agent, manager, admin | `GET /api/v1/tickets/:id`, `PATCH /api/v1/tickets/:id`, `POST /api/v1/tickets/:id/transition`, `GET /api/v1/tickets/:id/comments`, `POST /api/v1/tickets/:id/comments`, `GET /api/v1/tickets/:id/timeline`, `POST /api/v1/tickets/:id/approvals`, `DELETE /api/v1/tickets/:id`, `GET /api/v1/users`, `GET /api/v1/teams`, `GET /api/v1/ticket-categories`, `GET /api/v1/assets` | W2.4–W2.6, W3.2–W3.6, W4.4–W4.6 | inline edit priority/assignee/team/category/asset/tags/title/description, transition buttons (per table) with note, comment body + private toggle (staff), request approval (approver picker), delete (admin, confirm dialog), timeline load more | loading (skeleton), error (404 designed, 409 conflict banner with reload), empty comments, success toasts |
| Approvals | `/approvals` | `/approvals` | requester, agent, manager, admin | `GET /api/v1/approvals`, `GET /api/v1/approvals/:id`, `POST /api/v1/approvals/:id/decide`, `DELETE /api/v1/approvals/:id`, `POST /api/v1/approvals` | W4.2, W5.3 | tabs inbox/requested/all, status filter, decide dialog (approve/reject + note), cancel (confirm), new approval dialog (subject type/id, approver, title, note, expiry) | empty ("Nothing waiting on you"), loading, error, success |
| Changes | `/changes` | `/changes` | agent, manager, admin | `GET /api/v1/changes`, `POST /api/v1/changes`, `GET /api/v1/tickets`, `GET /api/v1/assets` | W5.1 | status[], risk, search, sort, create dialog (title, description, risk, ticket picker, asset picker, planned window, plans) | empty, loading, error (field-level in dialog), success |
| Change detail | `/changes/[id]` | `/changes/00000000-0000-4000-8000-000000000002` | agent, manager, admin | `GET /api/v1/changes/:id`, `PATCH /api/v1/changes/:id`, `POST /api/v1/changes/:id/transition`, `GET /api/v1/changes/:id/timeline`, `DELETE /api/v1/changes/:id`, `GET /api/v1/users` | W5.2, W5.4–W5.6 | edit fields (draft/submitted/rejected), submit-for-approval (approver picker), schedule (start/end), start, complete/roll back (note), cancel, delete (admin) | loading, error (409 conflict, 400 plan missing), success, timeline empty |
| Assets | `/assets` | `/assets` | requester (mine), agent, manager, admin | `GET /api/v1/assets`, `POST /api/v1/assets` | W4.3 | view all/mine, type, status, search, assignedTo, sort, create dialog (tag, name, type, status, serial, model, location, dates, notes) | empty, loading, error, success; warranty-expiring badge |
| Asset detail | `/assets/[id]` | `/assets/00000000-0000-4000-8000-000000000003` | requester (assigned), agent, manager, admin | `GET /api/v1/assets/:id`, `PATCH /api/v1/assets/:id`, `POST /api/v1/assets/:id/assign`, `GET /api/v1/assets/:id/timeline`, `DELETE /api/v1/assets/:id`, `GET /api/v1/users`, `GET /api/v1/tickets` | W4.3, W4.6 | inline edit, assign dialog (user picker, optional ticket picker), unassign, retire (admin confirm), timeline | loading, error (404 designed, 409), empty timeline, success |
| Projects | `/projects` | `/projects` | agent, manager, admin | `GET /api/v1/projects`, `POST /api/v1/projects`, `GET /api/v1/users` | W6.1 | status, owner, search, sort, create dialog (name, description, owner, dates) | empty, loading, error, success (progress + health) |
| Project detail | `/projects/[id]` | `/projects/00000000-0000-4000-8000-000000000004` | agent, manager, admin | `GET /api/v1/projects/:id`, `PATCH /api/v1/projects/:id`, `DELETE /api/v1/projects/:id`, `POST /api/v1/projects/:id/milestones`, `PATCH /api/v1/projects/:id/milestones/:milestoneId`, `DELETE /api/v1/projects/:id/milestones/:milestoneId`, `GET /api/v1/projects/:id/tasks`, `POST /api/v1/projects/:id/tasks`, `PATCH /api/v1/projects/:id/tasks/:taskId`, `DELETE /api/v1/projects/:id/tasks/:taskId`, `GET /api/v1/users` | W6.2–W6.5 | project edit (status/owner/dates), milestone add/edit/complete/delete, task board status select, task drawer (title, description, milestone, assignee, priority, due, dependency), delete (confirm) | loading, error (409 dependency/tasks open), empty board, success |
| Knowledge base | `/kb` | `/kb` | requester, agent, manager, admin | `GET /api/v1/kb/articles` | W2.2, W7.3 | search, category chips, status filter (staff), "New article" | empty (no articles yet vs 0 matched with "open a ticket"), loading, error, success |
| KB article | `/kb/[slug]` | `/kb/getting-started` | requester (published), agent, manager, admin | `GET /api/v1/kb/articles/:idOrSlug`, `PATCH /api/v1/kb/articles/:idOrSlug`, `DELETE /api/v1/kb/articles/:idOrSlug` | W7.2, W7.3 | edit toggle (staff) → editor with preview, publish/unpublish/archive, delete (admin confirm), TOC | loading, error (404 designed), success |
| New KB article | `/kb/new` | `/kb/new` | agent, manager, admin | `POST /api/v1/kb/articles` | W7.1 | title, slug (auto), category, tags, Markdown body with live preview, save draft / publish | loading, error (slug taken suggestion, field-level), success (→ article) |
| Analytics | `/analytics` | `/analytics?range=30d` | manager, admin | `GET /api/v1/analytics/overview`, `GET /api/v1/teams` | W9.1, W9.2 | range 7/30/90 d, team filter | empty (no data in range), loading (skeleton charts), error, success (charts with legend) |
| Automation | `/automation` | `/automation` | manager (read), admin | `GET /api/v1/automation/rules`, `POST /api/v1/automation/rules`, `GET /api/v1/automation/rules/:id`, `PATCH /api/v1/automation/rules/:id`, `DELETE /api/v1/automation/rules/:id`, `GET /api/v1/automation/runs`, `GET /api/v1/automation/recurring`, `POST /api/v1/automation/recurring`, `PATCH /api/v1/automation/recurring/:id`, `DELETE /api/v1/automation/recurring/:id`, `GET /api/v1/teams`, `GET /api/v1/users`, `GET /api/v1/ticket-categories`, `GET /api/v1/projects` | W8.1, W8.2, W8.4 | tabs rules/runs/recurring; rule dialog (name, trigger, condition rows field/op/value, action rows type/params, active); runs filter by rule; recurring dialog (title, kind, template, interval, next run) | empty, loading, error (field-level per row), success |
| Admin · Users | `/admin/users` | `/admin/users` | admin (agent/manager read-only list) | `GET /api/v1/users`, `POST /api/v1/users`, `GET /api/v1/users/:id`, `PATCH /api/v1/users/:id`, `DELETE /api/v1/users/:id`, `GET /api/v1/teams` | W1.3, W1.4, W1.5 | search, role filter, active filter, sort, create dialog (name, email, role, teams), edit drawer, deactivate confirm | empty, loading, error, success |
| Admin · Teams | `/admin/teams` | `/admin/teams` | admin (others read) | `GET /api/v1/teams`, `POST /api/v1/teams`, `GET /api/v1/teams/:id`, `PATCH /api/v1/teams/:id`, `DELETE /api/v1/teams/:id`, `POST /api/v1/teams/:id/members`, `DELETE /api/v1/teams/:id/members/:userId`, `GET /api/v1/users` | W1.2 | search, create dialog, detail drawer (name, description, lead, members add/remove), delete confirm | empty, loading, error, success |
| Admin · Catalog | `/admin/catalog` | `/admin/catalog` | admin | `GET /api/v1/ticket-categories`, `POST /api/v1/ticket-categories`, `PATCH /api/v1/ticket-categories/:id`, `DELETE /api/v1/ticket-categories/:id`, `GET /api/v1/teams`, `GET /api/v1/sla-policies` | W4.1 (setup) | include inactive, dialog (name, description, type, default team, default priority, SLA policy, requires approval), deactivate | empty, loading, error, success |
| Admin · SLA | `/admin/sla` | `/admin/sla` | manager, admin | `GET /api/v1/sla-policies`, `POST /api/v1/sla-policies`, `GET /api/v1/sla-policies/:id`, `PATCH /api/v1/sla-policies/:id`, `DELETE /api/v1/sla-policies/:id` | W3.5 (setup) | dialog (name, priority, response/resolution minutes, business hours editor: timezone, days, start/end or 24×7, default flag), delete confirm | empty, loading, error (409 in use), success |
| Admin · Audit | `/admin/audit` | `/admin/audit` | manager, admin | `GET /api/v1/audit` | W1.5, W4.6, W8.5, W9.4 | entity type, action, actor, from/to, load more, diff drawer | empty (0 matched explanation), loading, error, success |
| Notifications | `/notifications` | `/notifications` | all | `GET /api/v1/notifications`, `PATCH /api/v1/notifications/:id`, `POST /api/v1/notifications/read-all` | W3.5 | unread only, mark read, read all, load more | empty ("You're all caught up"), loading, error, success |
| Settings · Profile | `/settings/profile` | `/settings/profile` | all | `GET /api/v1/auth/me`, `PATCH /api/v1/auth/me`, `POST /api/v1/auth/me/password`, `POST /api/v1/auth/logout` | — (F-05) | name, theme (system/light/dark), current/new password, save, sign out | loading, error (field-level), success (toast) |

Shell-level bindings (present on every `(app)` screen): `GET /api/v1/auth/me`, `GET /api/v1/notifications` (bell), `GET /api/v1/events/stream` (live), `GET /api/v1/search` (command palette), `POST /api/v1/auth/logout` (user menu). 28 screens (5 public, 23 authenticated). The not-found and error boundaries are routes but not screens.

**Coverage cross-check:** every §3 endpoint that is not marked as a system probe appears in a `Binds` cell or the shell list above (`GET /api/v1/tasks` → Dashboard; `GET /api/v1/users/:id` → Admin · Users drawer; `GET /api/v1/teams/:id` → Admin · Teams drawer; `GET /api/v1/approvals/:id` → Approvals decide dialog; `GET /api/v1/sla-policies/:id` → Admin · SLA dialog; `GET /api/v1/automation/rules/:id` → Automation dialog).

### Design System (DESIGN.md Part III)

- **Archetype:** `saas` — persistent app shell (sidebar 224 px + contextual header with title, breadcrumb, primary action), composed dashboard hierarchy (lead metric → supporting → detail), engineered tables and forms, moderate functional motion. Below `md`: sidebar → focus-trapped drawer; KPI rows wrap 2-up then 1-up; tables scroll horizontally inside `TableWrap` (`role=region`, `tabindex=0`, `aria-label`) or collapse to card lists where a row has ≤ 5 columns.
- **Tokens** (`apps/web/src/styles/tokens.css`, consumed via Tailwind `@theme inline`): colors exactly as RESEARCH.md Design Language (light `:root`, dark `[data-theme="dark"]`), plus derived surfaces `--color-surface-2` (elevated), `--color-border` (`color-mix` of text 12 %), `--color-primary-fg` (`#161b1d` on primary — contrast checked), status colors (`--color-status-new` = secondary, `open` = primary, `in-progress` = primary, `waiting` = warning, `pending` = warning, `resolved` = success, `closed` = text-secondary, `cancelled` = text-secondary, `breached` = error). Spacing unit 4 px; scale 4/8/12/16/24/32/48/64. Radius 8 (sm 4, lg 12, xl 16). Shadow `0 1px 3px rgb(0 0 0 / 0.10)` (light) / `0 1px 3px rgb(0 0 0 / 0.60)` (dark). Motion: 160 ms (feedback) / 240 ms (panel/drawer) `cubic-bezier(.2,.8,.2,1)`; one staggered reveal on dashboard first paint; everything zero under `prefers-reduced-motion`. Breakpoints 640/768/1024/1280.
- **Type:** Manrope 700/600 for h1–h3, page titles, KPI values (tabular numerals); Inter 400/500 body; JetBrains Mono for ids, tags, code, kbd. Scale (14 px base × 1.2): 9.7 (micro-label uppercase tracking .06em) / 11.7 (caption) / 14 (body) / 16.8 (lead) / 20.2 (h3) / 24.2 (h2) / 29 (h1). Body ≥ 16 px at ≤ 640 px (inputs 16 px always). Reading measure for KB articles 68ch.
- **Atmosphere:** background `#fafafa`/`#090b0c` with a very subtle radial primary tint behind the dashboard header (`color-mix` 6 %), no gradients on text, no template hero.
- **Signature components:** `StatusPill`/`PriorityPill` (dot + label, tokenized), `SlaChip` (countdown, breached state), `KpiCard` (value + delta + sparkline), `DataTable` (sort, filter bar, bulk bar sticky bottom on mobile, empty/loading/error/zero-results), `Timeline`, `Drawer`, `CommandPalette`, `Toast`.
- **States:** every screen designs empty (first-use vs 0-matched), loading (skeleton matching layout), error (human message, request id, retry), success (inline or toast proportional to action).

---

## 7. Error contract

Every non-2xx response from the API is `application/problem+json` (RFC 9457):

```json
{ "type": "https://orbit.app/errors/validation", "title": "Validation failed", "status": 400,
  "detail": "2 fields failed validation", "instance": "/api/v1/tickets",
  "code": "VALIDATION_FAILED", "requestId": "3f2b…",
  "errors": [ { "field": "title", "message": "Title must be at least 3 characters", "code": "TOO_SHORT" } ] }
```

| HTTP | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` (+ `errors[]` with per-field `code` ∈ REQUIRED, TOO_SHORT, TOO_LONG, INVALID_FORMAT, UNKNOWN_FIELD, FORBIDDEN_FIELD, BREACHED, OUT_OF_RANGE, DEPENDENCY_CYCLE, SELF_APPROVAL, ASSIGNEE_REQUIRED, TOKEN_INVALID) | zod failure or domain validation |
| 401 | `UNAUTHENTICATED`, `INVALID_CREDENTIALS` | no/expired session; bad login (uniform) |
| 403 | `CSRF_ORIGIN_MISMATCH`, `FORBIDDEN` | origin mismatch; role lacks capability on a resource whose existence is public to the role (e.g. requester posting a private note) |
| 404 | `NOT_FOUND` | unknown route; cross-org or unowned resource; hidden capability |
| 409 | `VERSION_CONFLICT` (errors[0] carries `currentVersion`), `INVALID_TRANSITION` (errors[0].code = `FROM_TO`), `DUPLICATE` (field), `ALREADY_DECIDED`, `IN_USE`, `INVALID_STATE`, `SELF_MODIFICATION`, `DEPENDENCY_OPEN`, `TASKS_OPEN`, `USE_APPROVAL`, `EMAIL_TAKEN`, `SLUG_TAKEN` | state conflicts |
| 413 | `PAYLOAD_TOO_LARGE` | body > 256 KB |
| 428 | `PRECONDITION_REQUIRED` | versioned PATCH without `If-Match`/`version` |
| 429 | `RATE_LIMITED`, `TOO_MANY_STREAMS` | limiter; SSE cap (`Retry-After` set) |
| 500 | `INTERNAL` | unexpected; no detail beyond "Something went wrong"; logged once at `error` with stack + requestId |
| 503 | `SERVICE_UNAVAILABLE` (`errors[0].field` names the dependency, e.g. `db`) | dependency down; health 503 |

Rules: `requestId` always echoed (also `X-Request-Id` header); the UI attaches each `errors[].field` message to the control via `aria-describedby` + `aria-invalid` and focuses the first invalid control; 5xx never carries internal messages; the Next.js server renders framework errors through `error.tsx` with the same tone.

---

## 8. Configuration & environment

| Variable | Purpose | Required | Default (dev/test only) | Service |
|---|---|---|---|---|
| `NODE_ENV` | runtime mode | optional | `development` | api, web |
| `PORT` | listen port | optional | api 4000, web 3000 | api, web |
| `LOG_LEVEL` | pino level | optional | `info` (`debug` dev) | api |
| `DATABASE_URL` | app-role connection (`orbit_app`) | **required** | — | api, migrate (seed) |
| `DATABASE_ADMIN_URL` | owner connection for migrations | **required** (migrate) | — | migrate |
| `SESSION_SECRET` `# SECRET` | HMAC key for session/reset tokens, ≥ 32 chars, no production default | **required** in production | dev constant | api |
| `APP_ORIGIN` | canonical origin for CSRF check and email links | **required** | `https://localhost:8443` | api |
| `TRUST_PROXY` | Express `trust proxy` hops | optional | `1` | api |
| `SMTP_URL` `# SECRET` | Nodemailer SMTP URL; absent → JSON transport (logs) | optional | — | api |
| `MAIL_FROM` | sender address | optional | `Orbit <no-reply@localhost>` | api |
| `HIBP_ENABLED` | breached-password screening via HIBP | optional | `true` | api |
| `AUTH_RATE_LIMIT_MAX` | login attempts / 15 min per (ip,email) | optional | `10` | api |
| `RATE_LIMIT_SCALE` | multiplier applied to every rate limit (tests set 1000) | optional | `1` | api |
| `SCHEDULER_ENABLED` | run in-process jobs | optional | `true` (`false` in test) | api |
| `SCHEDULER_INTERVAL_MS` | tick interval | optional | `15000` | api |
| `SEED_DEMO` | create demo org + admin on migrate | optional | `false` | migrate |
| `SEED_ADMIN_EMAIL` | demo admin email | required if `SEED_DEMO` | — | migrate |
| `SEED_ADMIN_PASSWORD` `# SECRET` | demo admin password (≥ 15) | required if `SEED_DEMO` | — | migrate |
| `API_INTERNAL_URL` | dev-only rewrite target for `/api` | optional | `http://localhost:4000` | web (dev) |
| `NEXT_PUBLIC_SITE_URL` | `metadataBase`, canonical, OG image | **required** | `https://localhost:8443` | web |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` `# SECRET` / `POSTGRES_DB` | container bootstrap | **required** (compose) | `orbit` / — / `orbit` | postgres |
| `APP_DB_PASSWORD` `# SECRET` | password for `orbit_app` role (set by migration) | **required** (compose) | — | migrate, api |
| `CADDY_SITE_ADDRESS` | Caddy site address (`localhost:8443` local, `ops.example.com` prod) | optional | `localhost:8443` | caddy |
| `WEB_BASELINE_USER` / `WEB_BASELINE_PASSWORD` `# SECRET` | audit login for authenticated screens | required for `web:baseline` | seeded demo admin | tooling |

Validated by `apps/api/src/config/env.ts` (zod); `.env.example` lists every key with a comment, required/optional and owning service.

---

## 9. Performance targets

Derived from RESEARCH.md metrics at tier `small`, measured on the local compose stack with autocannon (50 connections, 30 s) and the Web Delivery Baseline.

| Operation | Target |
|---|---|
| `GET /api/health` | p95 ≤ 100 ms, always ≤ 500 ms |
| `POST /auth/login` | p95 ≤ 400 ms (Argon2id ~150–250 ms) |
| `GET /tickets` (25 rows, filters) | p95 ≤ 250 ms at 5 000 tickets |
| `GET /tickets/:id` (with comments/timeline first page) | p95 ≤ 200 ms |
| `POST /tickets` (incl. SLA calc, audit, rules, publish) | p95 ≤ 400 ms |
| `POST /tickets/:id/transition` | p95 ≤ 300 ms |
| `GET /kb/articles?q=` | p95 ≤ 300 ms at 1 000 articles |
| `GET /analytics/overview` | p95 ≤ 500 ms at 5 000 tickets |
| `GET /dashboard` | p95 ≤ 400 ms |
| `GET /search` | p95 ≤ 300 ms |
| SSE propagation | publish → client event ≤ 2 s (≤ 200 ms in-process) |
| Scheduler tick | ≤ 2 s for 50 breaches; breach detected ≤ 30 s after due |
| Pages | LCP ≤ 2.5 s, CLS ≤ 0.1 at 375/768/1280; JS ≤ 400 KB compressed on landing and dashboard; Lighthouse ≥ 95 a11y/BP/SEO, ≥ 85 performance |
| Concurrency | 50 concurrent users mixed read/write without errors (0 non-2xx besides intended 4xx) |

---

## 10. Test plan

| Layer | Tooling | Coverage |
|---|---|---|
| Unit | vitest | `slaClock` (business hours, DST, pause), transitions table, predicate evaluator, `problem()` mapper, password policy + blocklist, cursor codec, serialize/field gating, Markdown sanitizer |
| Integration (API + Postgres on :5433) | vitest + supertest, each test file creates its own org(s) | Every endpoint happy path + every §7 error path listed per feature; RLS fail-closed (F-02 AC-2/3); same-org cross-user IDOR on every sub-resource route; cross-org 404 on every `/:id` route (generated matrix); rate-limit-before-hash; transition matrices (tickets, changes, tasks, approvals); concurrency races (ticket numbers, approvals double-decide, version conflicts, scheduler lock); outbox backoff/dlq; SSE org isolation + resume + cap; automation loop cap; analytics fixture values |
| Component | vitest + Testing Library (jsdom) | One state test per `components/ui` primitive; one validation-error test per form screen (login, register, reset, new ticket, change dialog, user dialog, rule dialog, KB editor, SLA dialog) |
| E2E | Playwright, Desktop Chrome + Pixel 7, production stack | `test('W1 …')` … `test('W9 …')` — one substantive spec per workflow driving the rendered UI to its terminal step; plus `auth-negative.spec.ts` (field-level error rendering) |
| Web delivery | `scripts/web-baseline.mjs` | Every §6 screen × 375/768/1280 × light/dark; Lighthouse on landing + dashboard |
| Smoke | `smoke-test.sh` (+ Playwright for UI flows) | Auth bootstrap, cross-tenant + same-tenant isolation incl. sub-resources, primary CRUD, background job (SLA breach + outbox), UI workflow completion, large-cookie replay, health + graceful shutdown, fail-fast config, web baseline `--site-only` on the deploy |
| Load | autocannon | §9 targets at 50 connections; results in REPORT.md |
| Coverage | `@vitest/coverage-v8` | ≥ 80 % statements on `apps/api/src/{modules,lib,http,jobs}` |

Mapping: F-01 → unit + integration (health, env, envelope, CSRF, shutdown); F-02 → integration (RLS, audit immutability, counters, migrations); F-03 → component + baseline; F-04–F-06 → integration + component + e2e W1/W2; F-07/F-08 → integration + e2e W1; F-09–F-11 → integration (transition matrix) + component + e2e W2/W3/W4; F-12 → unit + integration + e2e W3; F-13 → integration + e2e W4/W5; F-14 → integration + e2e W5; F-15 → integration + e2e W4; F-16 → integration + e2e W6; F-17 → integration + component + e2e W7; F-18 → integration + e2e W3; F-19 → unit + integration + e2e W8; F-20 → integration (fixtures) + e2e W9; F-21 → integration + e2e W9; F-22 → smoke + baseline + CI.
