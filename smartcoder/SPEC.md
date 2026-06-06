# SPEC.md — SmarCoders

> Source of truth for **behavior**. Architecture lives in `ARCHITECTURE.md`.
> Build depth: **comprehensive** — every state transition, error path, race condition, and trust boundary is specified.
> Another AI must be able to recreate the system from this document alone.

Legend: roles = **M** Manager, **S** Supervisor, **C** Coder. All data is Synthea-synthetic.

---

## 1. Feature List (vertical slices)

| ID | Feature | Surface |
|----|---------|---------|
| F-01 | Project scaffold, config, health, DB migrations runner | infra/API |
| F-02 | Design tokens + app shell (nav, layout, landing page ported from template) | UI |
| F-03 | Auth: register/login/logout, session, RBAC, anti-enumeration, rate limiting | API + UI |
| F-04 | Users & org bootstrap, seed admin, role management | API + UI |
| F-05 | Synthea import: upload/seed FHIR bundle → charts (BullMQ worker) | API + UI |
| F-06 | Queue management & assignment (auto-distribute by specialty/workload) | API + UI |
| F-07 | Chart viewer + coding (assign ICD-10 dx/proc codes, rationale, doc-gap flags) | API + UI |
| F-08 | Validation engine (format, sequencing, comorbidity, combination checks) | API + UI |
| F-09 | Supervisor audit + structured feedback + scoring | API + UI |
| F-10 | Real-time queue/metrics dashboard (SSE) + CSV/PDF export | API + UI |
| F-11 | Notifications (in-app + email outbox) | API + UI |
| F-12 | Audit log (append-only, hash-chained) + viewer | API + UI |
| F-13 | Transactional outbox relay + inbound webhook receiver (verify-first, idempotent) | API |
| F-14 | Invariant-lint runner + smoke harness + Claude scaffold | infra |

Non-goals (v1): licensed CPT descriptor content; production EHR/billing integration; multi-region; mobile native; MFA enrollment UI (MFA-ready posture only).

---

## 2. Roles & RBAC

| Role | Capabilities |
|------|-------------|
| **Coder (C)** | View own assigned charts; code charts; submit for audit; view own feedback/metrics; receive returned charts |
| **Supervisor (S)** | All coder views (read); audit submitted charts; modify codes; approve/return; write feedback + score; view team metrics |
| **Manager (M)** | All of above (read); import charts; assign/rebalance queue; manage users; full metrics + exports; view audit log |

Default-deny: every endpoint declares allowed roles; tenant (`org_id`) always enforced. Cross-tenant resource → `NOT_FOUND` (never `FORBIDDEN`) to avoid existence leak.

---

## 3. REST API Surface

All under `/api`. JSON. Auth via httpOnly cookie session (JWT). Errors per §7. Mutating routes require CSRF/Origin guard. Rate limits per §4.

### Auth
| Method | Path | Roles | Body | 2xx | Notes |
|---|---|---|---|---|---|
| POST | /api/auth/register | public | `{email,password,name,orgName?}` | 201 `{user}` | First user of a new org = manager. Anti-enumeration: existing email returns **same shape** as new (sends notice email; no distinguishable error). |
| POST | /api/auth/login | public | `{email,password}` | 200 `{user}` + sets cookie | Uniform `INVALID_CREDENTIALS` for unknown email / wrong password / rate-limited. Rate-limit consumed **before** argon2 verify (INV-7). Timing padded. |
| POST | /api/auth/logout | auth | — | 204 | Clears cookie |
| GET | /api/auth/me | auth | — | 200 `{user}` | Current principal |

### Users
| Method | Path | Roles | 2xx |
|---|---|---|---|
| GET | /api/users | M,S | 200 `{users[]}` (org-scoped) |
| POST | /api/users | M | 201 `{user}` (invite/create coder/supervisor) |
| PATCH | /api/users/:id | M | 200 `{user}` (role, active) |

### Charts & Coding
| Method | Path | Roles | 2xx | Notes |
|---|---|---|---|---|
| GET | /api/charts | M,S,C | 200 `{charts[],page}` | Coder sees only own assigned; filters: status, specialty, assignee |
| GET | /api/charts/:id | M,S,C | 200 `{chart, codes, validation}` | Coder must be assignee or 404 |
| POST | /api/charts/:id/codes | C | 201 `{code}` | Add a code (dx/proc) w/ rationale, sequence, is_principal |
| PATCH | /api/charts/:id/codes/:codeId | C,S | 200 `{code}` | Edit; S may edit during audit |
| DELETE | /api/charts/:id/codes/:codeId | C,S | 204 | |
| POST | /api/charts/:id/validate | C,S | 200 `{validation}` | Runs validation engine (§8) |
| POST | /api/charts/:id/submit | C | 200 `{chart}` | status draft→pending_audit; blocked if validation has blocking errors |
| GET | /api/code-reference | M,S,C | 200 `{codes[]}` | ICD-10-CM lookup (search `q`) |

### Audit
| Method | Path | Roles | 2xx | Notes |
|---|---|---|---|---|
| GET | /api/audits/queue | S,M | 200 `{charts[]}` | Charts in pending_audit |
| POST | /api/charts/:id/audit | S,M | 201 `{audit}` | Open/record an audit (score, decision approve|return, findings[]) |
| POST | /api/charts/:id/audit/:auditId/feedback | S,M | 201 `{feedback}` | Structured feedback, optional training tag |
| POST | /api/charts/:id/audit/:auditId/decision | S,M | 200 `{chart}` | approve→completed, return→draft (re-assigned to coder) |

### Queue / Imports / Metrics
| Method | Path | Roles | 2xx | Notes |
|---|---|---|---|---|
| POST | /api/imports | M | 202 `{importJob}` | Upload Synthea FHIR bundle (JSON) or trigger seed; enqueues BullMQ job |
| GET | /api/imports/:id | M | 200 `{importJob}` | Status/progress |
| POST | /api/queue/assign | M | 200 `{assigned}` | Auto-distribute unassigned by specialty + workload balance |
| PATCH | /api/queue/charts/:id/assign | M | 200 `{chart}` | Manual (re)assign |
| GET | /api/metrics/summary | M,S | 200 `{counts, rates, throughput}` | Queue health + KPIs |
| GET | /api/metrics/stream | M,S | text/event-stream | **SSE** live queue counters (<1s) |
| GET | /api/metrics/export?format=csv\|pdf | M,S | 200 file | CSV native; PDF rendered |

### Notifications / Audit log / Webhooks / Health
| Method | Path | Roles | 2xx | Notes |
|---|---|---|---|---|
| GET | /api/notifications | auth | 200 `{notifications[]}` | In-app, own |
| POST | /api/notifications/:id/read | auth | 204 | |
| GET | /api/audit-log | M | 200 `{entries[]}` | Append-only viewer (org-scoped) |
| POST | /api/webhooks/:source | public (signed) | 200 | **Internal**: verify HMAC → dedupe → process (INV-6). Not UI-bound. |
| GET | /api/health | public | 200 `{status, services}` | DB, Redis, worker, externals |

Internal/non-UI endpoints (no screen binding required): `/api/webhooks/*`, `/api/metrics/stream` (consumed by dashboard), `/api/health`.

---

## 4. Security Requirements (source for COMPLIANCE.md + Audit Gate)

| ID | Requirement | compliance: |
|----|-------------|-------------|
| SEC-01 | Passwords hashed with argon2id; never a substituted primitive | HIPAA access |
| SEC-02 | Anti-enumeration on register & login (uniform shape **and** timing) | HIPAA access |
| SEC-03 | Rate limiting on all public endpoints; auth limiter fires before hash compare; key `(ip,email)` | HIPAA |
| SEC-04 | Session is httpOnly+Secure+SameSite cookie; CSRF/Origin guard on mutations | HIPAA transmission |
| SEC-05 | Default-deny RBAC; tenant (`org_id`) enforced on every scoped query | HIPAA access control (164.312(a)) |
| SEC-06 | Cross-tenant access returns NOT_FOUND, not FORBIDDEN | info-leak |
| SEC-07 | Append-only **hash-chained** audit log of every PHI-equivalent access/mutation | audit controls (164.312(b)) |
| SEC-08 | Input validation via zod on every endpoint; structured field errors (§7) | integrity |
| SEC-09 | Webhook handlers verify HMAC over raw body before any DB read; idempotent via UNIQUE(event_id) | integrity/auth |
| SEC-10 | Dual-write via transactional outbox; consumers idempotent | integrity |
| SEC-11 | Secrets via AWS Secrets Manager (env fallback local); none committed | HIPAA |
| SEC-12 | Encryption: TLS-only (HTTPS, HSTS); Postgres volume encrypted-at-rest (delegated ⚠ where managed) | encryption (164.312(a)(2)(iv)) |
| SEC-13 | Security headers (HSTS, CSP, X-Content-Type-Options, frame-deny) + header-buffer sizing | transmission |
| SEC-14 | Structured JSON logging without secrets/PHI payloads; trace-correlated | audit |
| SEC-15 | Rate-limit + size cap on import uploads (CPU/memory cost) | DoS |

---

## 5. Key Workflows (every one completable end-to-end through the rendered UI to its terminal step)

### W1 — Chart Intake & Queue Management *(Manager)*
1. M opens **/imports**, uploads a Synthea FHIR bundle (or clicks "Load demo data").
2. API enqueues an `import` BullMQ job → worker parses bundle → creates `charts` (specialty, difficulty inferred), emits `chart.imported` via outbox.
3. M opens **/queue**, clicks **Assign unassigned** → auto-distribute by specialty + workload balance.
4. **Terminal step:** M sees the live **/metrics** dashboard update (unassigned / in-progress / pending-audit / completed counts) via SSE.
- Error paths: malformed bundle → import job FAILED with reason surfaced on /imports; oversized upload → 413 structured error; zero coders → assign returns `NO_ELIGIBLE_ASSIGNEES`.

### W2 — Medical Chart Coding *(Coder)*
1. C opens **/queue** (own assigned), opens a chart at **/charts/:id**.
2. C reviews synthetic history/labs/note (rendered from JSONB payload).
3. C adds ICD-10 dx codes (3–7) + procedure codes (1–3) with rationale, sets principal + sequence, flags documentation gaps.
4. C clicks **Validate** → sees validation results inline.
5. **Terminal step:** C clicks **Submit for audit** → chart moves to pending_audit, confirmation shown; blocked with field errors if blocking validation issues remain.
- Race: two tabs editing same chart → optimistic version on `charts.version`; stale write → `CONFLICT` with reload prompt. Submitting an already-submitted chart → `INVALID_STATE`.

### W3 — Coding Validation & Quality Checks *(System, surfaced to Coder)*
1. On **Validate**, engine runs (§8): format regex, principal-first sequencing, required-secondary/comorbidity heuristics, unsupported-combination rules.
2. Results rendered with severity (error/warning/info), each mapped to the offending code row.
3. **Terminal step:** C sees a pass/fail summary and per-code messages; blocking errors disable Submit until resolved.
- Edge: chart with zero codes → `NO_CODES` warning; non-existent ICD-10 code → format error on that row.

### W4 — Supervisor Audit & Feedback *(Supervisor)*
1. S opens **/audit** (pending_audit queue), opens a chart at **/audit/:id** beside source documentation.
2. S reviews codes, may modify/sequence, records findings (category + note), sets an audit score.
3. S writes structured feedback (template + free text), optional training-module tag.
4. **Terminal step:** S clicks **Approve** (→ completed) or **Return to coder** (→ draft, re-assigned, coder notified in-app + email). Decision + score persisted; audit-log entry written; finding logged for trend analysis.
- Edge: approving with unresolved blocking validation → warning confirm. Returning requires ≥1 feedback item.

---

## 6. Data Model (authoritative DDL in `db/migrations`)

- **organizations**(id, name, created_at)
- **users**(id, org_id→, email **UNIQUE lower(email)**, password_hash, name, role[manager|supervisor|coder], active, failed_logins, locked_until, created_at)
- **charts**(id, org_id→, external_ref, specialty, difficulty, status[draft|pending_audit|completed|returned], assignee_id→users, payload JSONB, version int, imported_from, created_at, updated_at)
- **chart_codes**(id, chart_id→, org_id→, code, code_system[ICD10CM|PROC], description, rationale, is_principal bool, seq int, added_by→users, created_at)
- **validation_results**(id, chart_id→, org_id→, severity[error|warning|info], rule_id, code_ref, message, created_at)
- **audits**(id, chart_id→, org_id→, auditor_id→users, score numeric, decision[approve|return], created_at)
- **audit_findings**(id, audit_id→, category, note, code_ref)
- **feedback**(id, audit_id→, chart_id→, org_id→, body, training_tag, created_at)
- **notifications**(id, org_id→, user_id→, type, body, read bool, created_at)
- **audit_log**(id, org_id→, actor_id, action, entity, entity_id, meta JSONB, prev_hash, row_hash, created_at) — **append-only, hash-chained**
- **outbox_events**(id UUID PK, org_id, type, payload JSONB, published bool, created_at, published_at)
- **processed_events**(id, source, **event_id UNIQUE**, received_at)
- **pending_emails**(id, org_id, to_addr, subject, html, status[pending|sent|failed], attempts, next_attempt_at, created_at)
- **import_jobs**(id, org_id→, status[queued|running|completed|failed], total, processed, error, created_at)
- **code_reference**(code PK, system, description, billable bool) — ICD-10-CM seed (public domain)

Indexes: charts(org_id,status), charts(assignee_id), chart_codes(chart_id), audit_log(org_id,created_at), outbox_events(published), pending_emails(status,next_attempt_at).

---

## 7. Error Contract (structured)

All errors: HTTP status + body
```json
{ "error": { "code": "STRING_ENUM", "message": "human readable", "fields": { "email": "Required" } } }
```
Typed codes: `VALIDATION_ERROR` (422, with `fields`), `INVALID_CREDENTIALS` (401), `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `CONFLICT` (409), `INVALID_STATE` (409), `RATE_LIMITED` (429, + `Retry-After`/`RateLimit-*`), `PAYLOAD_TOO_LARGE` (413), `NO_ELIGIBLE_ASSIGNEES` (422), `SERVICE_UNAVAILABLE` (503, `{dependency}` for unprovisioned externals), `INTERNAL` (500, no stack leak). UI maps `fields` onto inputs; never surfaces a raw stack/`[object Object]`.

---

## 8. Validation Engine Rules

| rule_id | Severity | Check |
|---|---|---|
| FMT-001 | error | ICD-10-CM format: `^[A-TV-Z][0-9][0-9AB](\.[0-9A-Z]{1,4})?$` |
| SEQ-001 | error | Exactly one principal diagnosis, sequenced first |
| SEQ-002 | warning | Principal must be a valid billable code |
| CMB-001 | warning | Required secondary/comorbidity present for flagged principals (e.g. diabetes w/ manifestations) — heuristic table |
| CMB-002 | warning | No mutually-exclusive code pair present |
| CNT-001 | warning | 3–7 dx codes, 1–3 procedure codes (per RESEARCH workflows) |
| DOC-001 | info | Documentation-gap flag recorded by coder |
| REF-001 | error | Each code exists in code_reference (or flagged unknown) |

Blocking = any `error`. Submit disabled while blocking errors exist.

---

## 9. Performance Targets (from Success Metrics)

| Target | Budget | Approach |
|---|---|---|
| Chart load | < 2s | indexed query, JSONB payload single fetch, RSC first paint |
| Audit submission | < 500ms | single txn + outbox insert; async side-effects |
| Queue dashboard refresh | < 1s | SSE push from Redis-cached counters (metrics job) |
| Comprehensive: representative load test | k6 vs above | scripts/load (best-effort) |

---

## 10. UI Surface

Topnav + content shell (max-width 1240px), light theme, tokens from `packages/tokens` (DESIGN-TEMPLATE verbatim). Every screen designs empty/loading/error/success states; maps §7 `fields` to inputs; WCAG 2.2 AA.

| Route | Purpose | Roles | States | Binds endpoints | Completes workflow step |
|---|---|---|---|---|---|
| `/` | Landing (ported template: nav, hero, trust strip, feature grid, CTA, footer) | public | static | — | entry |
| `/login` · `/register` | Auth | public | empty/loading/error(field)/success | POST /auth/login, /auth/register | W1–W4 entry |
| `/queue` | Coder/manager work queue (table, filters) | C,S,M | empty("No charts assigned"+CTA)/loading(skeleton)/error/success | GET /charts, PATCH /queue/charts/:id/assign | W1.3, W2.1 |
| `/charts/:id` | Chart viewer + coding panel | C,S,M | loading/error(404)/success; inline validation | GET /charts/:id, POST codes, POST validate, POST submit | W2.2–W2.5, W3 (terminal Submit) |
| `/audit` | Supervisor audit queue | S,M | empty/loading/error/success | GET /audits/queue | W4.1 |
| `/audit/:id` | Audit workspace (codes + source + feedback + decision) | S,M | loading/error/success | POST /charts/:id/audit, /feedback, /decision | W4.2–W4 (terminal Approve/Return) |
| `/metrics` | Real-time queue + KPI dashboard, export | M,S | loading/empty/error/success; live SSE | GET /metrics/summary, /metrics/stream, /metrics/export | W1.4 (terminal) |
| `/imports` | Synthea import + status | M | empty/uploading/processing/error/success | POST /imports, GET /imports/:id | W1.1–W1.2 |
| `/users` | User & role management | M | loading/empty/error/success | GET/POST/PATCH /users | admin |
| `/audit-log` | Append-only audit log viewer | M | loading/empty/error/success | GET /audit-log | compliance |
| `/notifications` | In-app notifications (also navbar bell) | auth | empty/loading/success | GET /notifications, POST read | W4 return notice |

Every non-internal endpoint above is reachable from ≥1 screen; every W1–W4 terminal step (submit / approve / export / import-complete) is completable through the rendered UI.

---

## Environment & Configuration

Config source: `.env` locally → **AWS Secrets Manager** in prod (loader falls back to env when `AWS_*` absent — ADR-009). Every variable is enumerated in `.env.example` with owning service.

| Var | Required | Owning service | Description / placeholder |
|---|---|---|---|
| `NODE_ENV` | yes | runtime | development \| production |
| `APP_URL` | yes | web/api | https://localhost (HTTPS only) |
| `API_PORT` | yes | api | 3001 |
| `WEB_PORT` | yes | web | 3000 |
| `DATABASE_URL` | yes | postgres | postgres://user:pass@db:5432/smartcoders |
| `REDIS_URL` | yes | redis/bullmq | redis://redis:6379 |
| `JWT_SECRET` | yes | auth | 32+ byte random (Secrets Manager in prod) |
| `SESSION_COOKIE_NAME` | yes | auth | sc_session |
| `WEBHOOK_SIGNING_SECRET` | yes | webhooks | HMAC key |
| `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` | yes (local) | seed | first manager account |
| `RESEND_API_KEY` | optional | email | re_REPLACE_ME (else outbox stays queued) |
| `EMAIL_FROM` | optional | email | no-reply@smartcoders.local |
| `SENTRY_DSN` | optional | sentry | else no-op |
| `DD_AGENT_HOST` / `DD_TRACE_ENABLED` | optional | datadog | else no-op |
| `AWS_REGION` / `AWS_SECRET_ID` | optional | secrets | else env fallback |
| `RATE_LIMIT_WINDOW_MS` / `RATE_LIMIT_MAX` | optional | rate-limit | defaults 60000 / 100 |

Deployment: `docker compose up` brings db, redis, api, worker, web, nginx (TLS). HTTP→HTTPS redirect + HSTS. Header buffers sized (INV-13). Backups: daily pg volume snapshot (documented cron). CI: typecheck, lint, test, `lint:invariants`, build.

---

## 11. Domain & Compliance checklist mapping (SPEC §5 requirements)
- **has_webhooks** → F-13: verify-first HMAC, `processed_events` UNIQUE(event_id), replay rejection (SEC-09, INV-4/6).
- **has_dual_write** → F-13: transactional `outbox_events`, polling relay, idempotent consumers (SEC-10, INV-5).
- **has_email (guaranteed delivery)** → F-11: `pending_emails` outbox, drain worker, capped backoff → DLQ; never sync-send.
- **HIPAA baseline** → SEC-01..14; breached-password screening (HIBP k-anonymity) on register, length-not-composition min 8, rate-limit-before-hash, encryption posture, audit controls.
