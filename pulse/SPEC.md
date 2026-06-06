# SPEC.md — Pulse

Behavioral contract. Another engineer (or AI) could recreate Pulse from this document alone.
Precedence: RESEARCH > ARCHITECTURE > SPEC. Build depth: **comprehensive**.

---

## 1. Features (capabilities)

F-01 **Auth & org bootstrap** — register (creates org + owner user), login, logout, current-user, session lifecycle, account lockout, audit of auth events.
F-02 **Monitors CRUD** — create/list/get/update/delete monitors of type `http|tcp|icmp|dns|push`; pause/resume; per-monitor interval/timeout/assertions.
F-03 **Health-check execution** — scheduler enqueues jittered checks; checker runs the protocol probe; persists `check_results`; updates `monitors.status`; pushes realtime updates.
F-04 **Alert state machine & incidents** — flap-detecting OK→DEGRADED→DOWN→recovery; opens/acknowledges/resolves incidents with a timeline.
F-05 **Notification channels** — CRUD for email/SMS/Slack/Teams/custom-webhook channels; secrets encrypted at rest; test-send.
F-06 **Alert rules** — route incident state changes to channels (per-monitor or all), with cooldown.
F-07 **Notification delivery** — guaranteed-delivery fan-out with retries/backoff/DLQ + idempotency; per-channel adapters; delivery audit.
F-08 **Metric ingestion** — `POST /v1/ingest/metrics` (API-key) accepts OS metrics (cpu/mem/disk/custom) for push monitors.
F-09 **Dashboards & realtime** — live fleet overview (status counts, lead metric), per-monitor detail with latency/uptime charts, incidents list, map/topology, all updating over WebSocket.
F-10 **Reports & export** — uptime/incident reports as JSON or CSV.
F-11 **Inbound webhooks** — Twilio SMS status callback (signature-verified) updating delivery status.
F-12 **Audit log view** — list tenant audit events (admin+).
F-13 **API keys** — CRUD for ingest API keys (shown once).

Non-Goals (NOT specced/architected): SIEM, endpoint management/patching, configuration management, asset lifecycle, ITSM/service-desk.

## 2. Data Model
See ARCHITECTURE §2 for the table catalog. Constraints (migration-defined): `organizations.slug` UNIQUE; `users.email` UNIQUE (INV-3); `sessions.token_hash` UNIQUE; `notifications.idempotency_key` UNIQUE (INV-2); FKs `*.org_id→organizations.id`, `monitors.created_by→users.id`, `check_results.monitor_id`, `incidents.monitor_id`, `alert_rules.channel_id→notification_channels.id`. Indexes on every `org_id`, `check_results(monitor_id, checked_at)`, `incidents(org_id,status)`, `metrics(monitor_id, ts)`. RLS policy on all tenant tables.

## 3. Roles & RBAC
`owner` (full incl. org/billing), `admin` (full app config: channels, rules, api-keys, users, audit), `operator` (monitors CRUD, acknowledge/resolve incidents, test-send), `viewer` (read-only dashboards/incidents/reports). Enforced server-side per endpoint; UI hides controls the role can't use but server is authoritative.

## 4. Security Requirements (compliance-mapped)
- SEC-01 (compliance: access-control, OWASP A01) — every tenant resource reached via `scopedDb(orgId)` + RLS; object-level authz; cross-tenant access returns 404 `not_found` (no existence leak).
- SEC-02 (OWASP A07) — Argon2id password hashing; account lockout after 5 failures (`locked_until`); anti-enumeration: register with existing email returns the SAME shape as new; login failures return one generic envelope regardless of unknown-email/bad-password/locked; timing padded or per-IP rate-limited.
- SEC-03 (OWASP A05; compliance: rate-limiting) — Redis token-bucket per (org,ip), auth/read/write tiers, `429 + Retry-After`; auth limiter before password compare (INV-5).
- SEC-04 (OWASP A03) — Zod validation at every input boundary; parameterized queries (Drizzle); body-size limits.
- SEC-05 (OWASP A08) — inbound webhooks signature-verified before processing (INV-4); generic outbound webhooks HMAC-SHA256 signed with timestamp/replay window.
- SEC-06 (OWASP A10) — outbound custom-webhook URLs pass an SSRF egress guard (https-only, block private/loopback/link-local) (INV-15).
- SEC-07 (OWASP A02; compliance: encryption-at-rest, secrets) — secrets via env vars; channel secrets AES-256-GCM at rest (INV-14); Pino redaction; no secrets in logs/responses.
- SEC-08 (compliance: TLS) — HTTPS-only + HSTS + HTTP→HTTPS redirect in production (INV-13, SEC-08).
- SEC-09 (compliance: audit-logging) — all mutations + auth events written to `audit_logs` (actor, action, resource, ip, ts).
- SEC-10 (compliance: data-retention) — check_results/metrics retention configurable (`DATA_RETENTION_DAYS`), rollup/downsample job; documented retention posture.
- SEC-11 (OWASP A06) — CI dependency + secret scanning; pinned versions + lockfile.

## 5. Key Workflows (each completable end-to-end through the UI to its terminal step)

**W1 — Auth bootstrap & first sign-in.** Register (org+owner) → land on dashboard → logout → login. Terminal step: authenticated dashboard rendered. UI: `/register`, `/login`, `/app`. API: `POST /v1/auth/register`, `POST /v1/auth/login`, `POST /v1/auth/logout`, `GET /v1/auth/me`.

**W2 — Create & run a monitor (Infrastructure Discovery + Health Monitoring).** Operator creates an HTTP monitor → scheduler enqueues a check → checker probes → result + status appear live on the dashboard and monitor detail. Terminal step: monitor detail shows a check result and latency point. UI: `/app/monitors`, `/app/monitors/new`, `/app/monitors/:id`. API: `POST /v1/monitors`, `GET /v1/monitors`, `GET /v1/monitors/:id`, `GET /v1/monitors/:id/results`.

**W3 — Configure alerting end-to-end (Alerting & Escalation).** Admin creates a notification channel (e.g. custom webhook) → test-send succeeds → creates an alert rule binding it to a monitor/state → when the monitor goes DOWN an incident opens and a notification is delivered/recorded. Terminal step: incident detail shows the delivered (or recorded-pending) notification. UI: `/app/channels`, `/app/channels/new`, `/app/rules`, `/app/incidents/:id`. API: `POST /v1/channels`, `POST /v1/channels/:id/test`, `POST /v1/rules`, `GET /v1/incidents/:id`.

**W4 — Investigate & resolve an incident (Incident Investigation).** Operator opens an incident → reviews timeline + affected monitor metrics → acknowledges → adds a note → resolves. Terminal step: incident status = resolved with timeline entries. UI: `/app/incidents`, `/app/incidents/:id`. API: `GET /v1/incidents`, `POST /v1/incidents/:id/acknowledge`, `POST /v1/incidents/:id/notes`, `POST /v1/incidents/:id/resolve`.

**W5 — Operational visibility & export (Operational Visibility).** NOC views fleet overview + map/topology → opens a report → exports CSV. Terminal step: a CSV file is downloaded through the UI. UI: `/app` (overview), `/app/map`, `/app/reports`. API: `GET /v1/reports/uptime?format=csv`, `GET /v1/reports/incidents?format=json`.

**W6 — Ingest OS metrics via API key (capacity visibility).** Admin creates an API key → a push monitor receives metrics posted to `/v1/ingest/metrics` → cpu/mem/disk render on monitor detail. Terminal step: metric values visible on the push monitor's detail screen. UI: `/app/settings/api-keys`, `/app/monitors/:id`. API: `POST /v1/api-keys`, `POST /v1/ingest/metrics` (X-API-Key).

## 6. API Surface (REST, mounted at `/v1`)
All JSON. Auth via session cookie unless noted. Errors: RFC 9457 problem+json (see §7). RBAC per §3.

| Method | Path | Auth | Role | Body / query | Returns |
|--------|------|------|------|--------------|---------|
| POST | /v1/auth/register | none | — | {orgName,email,password,name} | 201 {user,org} + session cookie |
| POST | /v1/auth/login | none | — | {email,password} | 200 {user} + cookie / 401 generic |
| POST | /v1/auth/logout | cookie | any | — | 204 |
| GET | /v1/auth/me | cookie | any | — | 200 {user,org} |
| GET | /v1/monitors | cookie | viewer+ | ?status&type&page | 200 {data,page} |
| POST | /v1/monitors | cookie | operator+ | {name,type,target,intervalSeconds,timeoutMs,config,region} | 201 {monitor} |
| GET | /v1/monitors/:id | cookie | viewer+ | — | 200 {monitor} / 404 |
| PATCH | /v1/monitors/:id | cookie | operator+ | partial | 200 {monitor} |
| DELETE | /v1/monitors/:id | cookie | operator+ | — | 204 |
| POST | /v1/monitors/:id/pause | cookie | operator+ | — | 200 {monitor} |
| POST | /v1/monitors/:id/resume | cookie | operator+ | — | 200 {monitor} |
| GET | /v1/monitors/:id/results | cookie | viewer+ | ?from&to&limit | 200 {data} |
| GET | /v1/monitors/:id/metrics | cookie | viewer+ | ?from&to | 200 {data} |
| GET | /v1/incidents | cookie | viewer+ | ?status&page | 200 {data,page} |
| GET | /v1/incidents/:id | cookie | viewer+ | — | 200 {incident,events,notifications} |
| POST | /v1/incidents/:id/acknowledge | cookie | operator+ | — | 200 {incident} |
| POST | /v1/incidents/:id/notes | cookie | operator+ | {message} | 201 {event} |
| POST | /v1/incidents/:id/resolve | cookie | operator+ | {rootCause?} | 200 {incident} |
| GET | /v1/channels | cookie | admin+ | — | 200 {data} |
| POST | /v1/channels | cookie | admin+ | {type,name,config} | 201 {channel} (secrets redacted) |
| PATCH | /v1/channels/:id | cookie | admin+ | partial | 200 {channel} |
| DELETE | /v1/channels/:id | cookie | admin+ | — | 204 |
| POST | /v1/channels/:id/test | cookie | admin+ | — | 200 {delivered} / 502 {problem} |
| GET | /v1/rules | cookie | admin+ | — | 200 {data} |
| POST | /v1/rules | cookie | admin+ | {monitorId?,channelId,onStates,cooldownSeconds} | 201 {rule} |
| PATCH | /v1/rules/:id | cookie | admin+ | partial | 200 {rule} |
| DELETE | /v1/rules/:id | cookie | admin+ | — | 204 |
| GET | /v1/reports/uptime | cookie | viewer+ | ?from&to&format=json\|csv | 200 json or text/csv |
| GET | /v1/reports/incidents | cookie | viewer+ | ?from&to&format=json\|csv | 200 json or text/csv |
| GET | /v1/api-keys | cookie | admin+ | — | 200 {data} (prefix only) |
| POST | /v1/api-keys | cookie | admin+ | {name} | 201 {apiKey} (full key once) |
| DELETE | /v1/api-keys/:id | cookie | admin+ | — | 204 |
| POST | /v1/ingest/metrics | X-API-Key | — | {monitorId,cpu,mem,disk,custom?} | 202 {accepted} |
| GET | /v1/audit | cookie | admin+ | ?page | 200 {data,page} |
| POST | /v1/webhooks/twilio/status | signature | — | Twilio form | 204 (verify-before-write, INV-4) |
| GET | /api/health | none | — | — | 200 {status,services,version} |

Every non-internal endpoint above is bound to a UI screen in §UI Surface (`/api/health`, `/v1/webhooks/*`, `/v1/ingest/*` are internal/agent surfaces, not user screens).

## 7. Error Contract (RFC 9457 problem+json)
Shape: `{ "type": "about:blank", "title": <human>, "status": <int>, "detail": <human>, "code": <TYPED>, "errors"?: { <field>: <message> } }`.
Typed codes: `VALIDATION` (400, with field `errors`), `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `CONFLICT` (409), `RATE_LIMITED` (429, `Retry-After`), `UPSTREAM` (502, delivery/provider), `INTERNAL` (500). The web client maps `errors` onto the offending field controls; never collapses field errors to one line. Auth failures always return one `UNAUTHENTICATED` envelope (anti-enumeration).

## 8. State Machines
**Monitor/alert status.** `pending → up`. Failing checks: `up → degraded` after `1` failure, `degraded → down` after `failureThreshold` (default 3) consecutive failures OR failure-ratio ≥ 0.6 over the window. Recovery: `down/degraded → up` after `recoveryThreshold` (default 2) consecutive successes. `paused` is manual and suspends checks. Transition to `down` opens an incident; transition to `up` while an incident is open resolves it (auto-resolve) and records an event.
**Incident.** `open → acknowledged → resolved` (manual) or `open → resolved` (auto on recovery). Re-failure after resolve opens a new incident.
**Notification.** `pending → sent` | `pending → failed → (retry) → sent` | `failed → dead` after max attempts. Idempotency key `incident:<id>:channel:<id>:transition:<state>` prevents duplicates.

## 9. Performance Targets (acceptance criteria, from Success Metrics)
- API reads p95 < 200ms (excluding report exports) — PERF-01.
- Check-to-detection latency: a down target produces a `down` status within `interval + timeout` and an incident within one evaluation cycle; sub-minute for default 30s interval — PERF-02 (MTTD < 5 min).
- Dashboard realtime update within 2s of a check result — PERF-03.
- Platform availability target 99.9% (health-checked, graceful degradation when Redis/providers down) — PERF-04.
- Scale posture: indexed org_id, partition-ready metrics, stateless autoscaled workers, jittered scheduling for 50k+ monitors — PERF-05.

## 10. Environment & Configuration / Deployment
Local stack via `docker compose up`: services `db` (Postgres 16), `redis`, `api`, `worker`, `web` (built SPA served by nginx with TLS-ready config + header buffers), reverse proxy enforcing HTTPS in production. Every env var documented in `.env.example` (DATABASE_URL, REDIS_URL, SESSION_SECRET, SECRET_ENCRYPTION_KEY, PUBLIC_URL, PORT, NODE_ENV, COOKIE_DOMAIN, TWILIO_*, SMTP_*, DATA_RETENTION_DAYS, RATE_LIMIT_*, optional). Secrets via env only; `.env` gitignored. Migrations run on api start (or `npm run db:migrate`). QUICKSTART.md documents clean-clone-to-running + Protocol & TLS (HTTPS-only path).

---

## UI Surface

### Design System (binding; tokens = DESIGN-TEMPLATE.html `:root`, copied verbatim — see DECISIONS ADR-0002)
- **Tokens** in `apps/web/src/styles/tokens.css`: colors `--color-bg #f5f7fb`, `--color-surface #fff`, `--color-fg #0d1526`, `--color-muted #5b6b85`, `--color-border #e4e9f2`, `--color-accent #2563eb`, `--color-accent-2 #1d4ed8`, `--color-accent-soft #eaf1ff`, `--color-success #15a35b`, `--color-warning #d98307`, `--color-error #dc2b3d`, ink panel `--color-ink #0a1020` / `--color-ink-2 #111a30` / `--color-ink-line #1e2a45` / `--color-ink-muted #8a99b8`. Spacing `--space-1..9` (4/8/12/16/24/32/48/64/96). Radius `--radius-sm 6 / md 12 / lg 20 / pill 999`. Shadows `--shadow-sm/md/lg`.
- **Fonts**: Manrope (display 500–800), Inter (body 400–600), JetBrains Mono (labels/telemetry), `swap`, preconnect.
- **Theme**: light only (no switcher); ink panels for hero/CTA/console focal contrast.
- **Archetype (saas) Layout Doctrine**: persistent app shell — left sidebar nav + contextual header (title, primary action) + content region. Dashboard = composed hierarchy (lead metric → supporting → detail), NOT an undifferentiated tile grid. Tables and forms are first-class & fully-stated. Motion moderate/functional (route transitions, skeletons, optimistic feedback). Every screen: empty/loading/error/success designed; `:focus-visible` styled; ≥44px targets; WCAG 2.2 AA contrast.

### Screen Inventory
Every screen: route · purpose · components · states (empty/loading/error/success) · bound API · workflow step(s) · RBAC.

| Route | Purpose | Bound API | Workflow | RBAC | States |
|-------|---------|-----------|----------|------|--------|
| `/` | Marketing landing (ported template: nav, hero+console, trust, bento features, CTA, footer) | — | entry | public | static |
| `/register` | Create org + owner | POST /v1/auth/register | W1 | public | form: idle/validating/error(field-mapped)/success→/app |
| `/login` | Sign in | POST /v1/auth/login | W1 | public | idle/error(generic)/success |
| `/app` | Fleet overview: lead "monitors down/degraded/up" hierarchy, live status, recent incidents, latency sparkline | GET /v1/monitors, /v1/incidents (WS) | W2,W5 | viewer+ | empty(no monitors→CTA "Add your first monitor")/loading(skeleton)/error/populated |
| `/app/monitors` | Monitors table (sort/filter/status, row actions, bulk pause) | GET /v1/monitors | W2 | viewer+ | empty/loading/error/populated |
| `/app/monitors/new` | Create monitor form (type-aware fields, assertions) | POST /v1/monitors | W2 | operator+ | idle/validating/error(field)/success |
| `/app/monitors/:id` | Monitor detail: status, uptime %, latency chart, recent results, CPU/mem/disk (push), incidents, edit/pause/delete | GET /v1/monitors/:id, /results, /metrics | W2,W6 | viewer+ | loading/error/empty(no results yet)/populated |
| `/app/incidents` | Incidents list (status filter) | GET /v1/incidents | W4 | viewer+ | empty("No incidents — all healthy")/loading/error/populated |
| `/app/incidents/:id` | Incident detail: timeline, affected monitor, metrics, notifications, acknowledge/note/resolve | GET /v1/incidents/:id + actions | W3,W4 | viewer+ (actions operator+) | loading/error/populated |
| `/app/channels` | Notification channels list | GET /v1/channels | W3 | admin+ | empty(CTA)/loading/error/populated |
| `/app/channels/new` | Create channel (type-aware: email/sms/slack/teams/webhook) + test-send | POST /v1/channels, /test | W3 | admin+ | idle/validating/error/success(test result) |
| `/app/rules` | Alert rules (bind monitor↔channel↔states, cooldown) | GET/POST /v1/rules | W3 | admin+ | empty/loading/error/populated |
| `/app/map` | Geographic/topology view of monitors by region with status coloring | GET /v1/monitors | W5 | viewer+ | empty/loading/error/populated |
| `/app/reports` | Uptime/incident reports + export JSON/CSV | GET /v1/reports/* | W5 | viewer+ | empty/loading/error/populated(+download) |
| `/app/settings/api-keys` | API keys CRUD (full key shown once on create) | GET/POST/DELETE /v1/api-keys | W6 | admin+ | empty/loading/error/populated |
| `/app/settings/audit` | Audit log table | GET /v1/audit | F-12 | admin+ | empty/loading/error/populated |

Every §5 workflow terminal step is reachable through these screens (no API-only path). Every non-internal §6 endpoint is bound above.

## 11. Test Plan
- **Unit** (Vitest): alert state machine (flap/recovery thresholds, ratio), error-contract builder, crypto encrypt/decrypt round-trip, SSRF guard (rejects private ranges), HMAC sign/verify, scopedDb query shape, CSV serializer.
- **Integration** (Vitest + ephemeral Postgres/Redis or mocked): auth register/login/lockout/anti-enumeration, tenant isolation (A cannot read B → 404), monitors CRUD + RBAC, incident lifecycle, notification idempotency (duplicate transition → one row), ingest API-key auth, Twilio webhook signature verify-before-write, rate-limit 429.
- **E2E** (Playwright, one `test('W<n> …')` per workflow, INV-11): W1–W6 driven through the rendered UI to the terminal step (incl. CSV download for W5, metric render for W6).
- **Smoke** (Functional Smoke Test): auth bootstrap, authz isolation, primary CRUD, webhook reception, background job execution, UI workflow completion, large-header resilience.
