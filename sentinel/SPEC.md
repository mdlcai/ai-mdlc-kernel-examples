# SPEC.md — Sentinel

Behavioral contract for the Sentinel unified AppSec platform. Source of truth for behavior (precedence: RESEARCH > ARCHITECTURE > SPEC). Build depth: **comprehensive** — every state transition, error path, race/idempotency concern, input boundary, and trust transition is specified. Non-Goals (RESEARCH) are excluded.

---

## 1. Feature Inventory

| ID | Feature | Summary |
|---|---|---|
| F1 | Auth & sessions | Register, login, logout, session, current user. argon2id, anti-enumeration, rate-limited. |
| F2 | Orgs & membership | Org auto-created on register; roles owner/admin/member; invite (spec). Multi-tenant root. |
| F3 | Projects | CRUD projects within an org; group targets. |
| F4 | Targets | Add repo/domain/ip targets; ownership verification for live targets; SSRF validation. |
| F5 | Scans & orchestration | Create scan → fan-out per scanner → sandboxed exec → status lifecycle. |
| F6 | Normalization & dedup | Adapter per scanner → canonical Finding → enrich → fingerprint dedup. |
| F7 | Findings & triage | List/filter/detail; status transitions open/fixed/ignored/snoozed; re-scan reconciliation. |
| F8 | Dashboard rollups | Per-org/project posture: open criticals, severity breakdown, trend. |
| F9 | Webhooks ingress | GitHub/GitLab signed webhooks → trigger scans; idempotent. |
| F10 | Notifications | Email (Resend) + Slack on scan complete / new critical; guaranteed delivery. |
| F11 | Reports & export | JSON export (built); PDF export (server-rendered). |
| F12 | Audit log | Immutable record of privileged mutations; 1-year retention. |
| F13 | Rate limiting | Per-IP/route + auth `(ip,email)` limiter before hash compare. |
| F14 | Health & ops | `/api/health`, structured logs, metrics, retention jobs. |

---

## 2. Data Model (Prisma / PostgreSQL 18)

Scoped tables (org-isolated) marked **⊕**. Every ⊕ row carries `orgId` and is accessed only via `withOrgScope` (INV-1). RLS policies mirror app scoping (ADR-001).

```
User            id, email(unique, citext), passwordHash, name, createdAt, updatedAt
Org             id, name, slug(unique), createdAt
Membership      id, userId, orgId, role(owner|admin|member), createdAt   [unique(userId,orgId)]
Session         id, userId, tokenHash(unique), expiresAt, createdAt
Project       ⊕ id, orgId, name, slug, createdAt, updatedAt              [unique(orgId,slug)]
Target        ⊕ id, orgId, projectId, kind(repo|domain|ip), value,
                 ownershipVerified(bool), ownershipToken, createdAt       [unique(orgId,projectId,value)]
Scan          ⊕ id, orgId, projectId, targetId, status(queued|running|completed|failed|partial),
                 scanners(string[]), startedAt, finishedAt, error, createdAt
ScanJob       ⊕ id, scanId, scanner, status(queued|running|done|failed), durationMs, raw(jsonb), error
Finding       ⊕ id, orgId, projectId, targetId, scanId, scanner, type(sast|sca|dast|secret|iac|port),
                 ruleId, title, severity(critical|high|medium|low|info), cvss, cve,
                 location(jsonb), remediation, fingerprint, status(open|fixed|ignored|snoozed),
                 statusReason, firstSeenAt, lastSeenAt, createdAt
                 [unique(orgId, fingerprint)]   ← INV-4 dedup key
AuditLog      ⊕ id, orgId, actorUserId, action, entity, entityId, meta(jsonb), createdAt
WebhookDelivery id, provider(github|gitlab), deliveryId(unique per provider), receivedAt, processedAt
Notification  ⊕ id, orgId, channel(email|slack), subject, status(pending|sent|failed), attempts, createdAt
```

**Indexes:** `Finding(orgId, projectId, severity, status)`, `Finding(orgId, scanId)`, `Scan(orgId, projectId, createdAt)`, `WebhookDelivery(provider, deliveryId)` unique.

**Retention:** Finding rows soft-aged at 90 days (status history preserved on re-seen); AuditLog 1 year (`data_retention_policy`).

---

## 3. API Surface (REST, `/v1`, problem+json errors)

All mutating endpoints: authenticated (session cookie) unless noted; rate-limited (F13); org-scoped. Errors: RFC 9457 `application/problem+json` (§7).

### Auth (public, anti-enumeration)
- `POST /v1/auth/register` → `{email,password,name}` → 201 `{user, org}`; existing email returns the **same 201-shaped success envelope semantics** (no existence leak — actually creates session only on genuine new; duplicate returns generic 200 "check your email" shape). Rate-limited `(ip,email)`.
- `POST /v1/auth/login` → `{email,password}` → 200 sets `sid` httpOnly cookie; unknown email / wrong password / rate-limited all return the **same 401 envelope + constant-time path**. Limiter fires before argon2 verify.
- `POST /v1/auth/logout` → 204, clears session.
- `GET /v1/auth/me` → 200 `{user, memberships, activeOrg}` | 401.

### Orgs
- `GET /v1/orgs` → orgs for current user.
- `POST /v1/orgs` → `{name}` → 201 (creator = owner).
- `GET /v1/orgs/:orgId/members` → members (admin+).

### Projects (⊕)
- `GET /v1/orgs/:orgId/projects` → list.
- `POST /v1/orgs/:orgId/projects` → `{name}` → 201.
- `GET /v1/orgs/:orgId/projects/:id` → detail (assertResourceAccess).
- `PATCH /v1/orgs/:orgId/projects/:id` → `{name}` → 200.
- `DELETE /v1/orgs/:orgId/projects/:id` → 204.

### Targets (⊕)
- `POST /v1/orgs/:orgId/projects/:pid/targets` → `{kind,value}` → 201. `domain|ip` → SSRF-validated (INV-11), `ownershipVerified=false`, returns `ownershipToken`.
- `POST /v1/orgs/:orgId/targets/:id/verify-ownership` → checks DNS TXT / file token → 200 `{ownershipVerified:true}` | 422 problem.
- `GET /v1/orgs/:orgId/projects/:pid/targets` → list.
- `DELETE /v1/orgs/:orgId/targets/:id` → 204.

### Scans (⊕)
- `POST /v1/orgs/:orgId/targets/:id/scans` → `{scanners:[...]}` → 202 `{scan}` enqueued. **Rejects 422 if a live target has `ownershipVerified=false`** (INV-12).
- `GET /v1/orgs/:orgId/scans/:id` → status + per-job progress.
- `GET /v1/orgs/:orgId/projects/:pid/scans` → list.

### Findings (⊕)
- `GET /v1/orgs/:orgId/findings?projectId&severity&status&type&scanner&page` → paginated, deduped, severity-sorted.
- `GET /v1/orgs/:orgId/findings/:id` → detail (location, cve, remediation, history).
- `PATCH /v1/orgs/:orgId/findings/:id` → `{status, statusReason}` → 200. Transitions §5.W4. `ignored|snoozed` require `statusReason` (422 otherwise).
- `GET /v1/orgs/:orgId/findings/export?format=json|pdf` → export.

### Dashboard
- `GET /v1/orgs/:orgId/dashboard?projectId` → rollups `{openBySeverity, openCriticals, trend, scanActivity}`.

### Webhooks (public, signature-verified — INV-7)
- `POST /v1/webhooks/github` → verify `X-Hub-Signature-256` → idempotency on `X-GitHub-Delivery` → trigger scan.
- `POST /v1/webhooks/gitlab` → verify `X-Gitlab-Token` → idempotency → trigger scan.

### Ops
- `GET /api/health` → 200 `{status:'ok', version, services:{db, redis, queue}}`.

---

## 4. Security Requirements (mapped in COMPLIANCE.md)

| ID | Requirement | Baseline |
|---|---|---|
| SEC-1 | All tenant data org-scoped; cross-tenant access returns 404 not 403. | OWASP A01; SOC2 CC6.1 |
| SEC-2 | Object-level authz via `assertResourceAccess` on every owned-resource + sub-resource route. | OWASP A01 |
| SEC-3 | argon2id password hashing; min length 8, accept ≥64, no composition rules, no forced rotation. | NIST 800-63B; OWASP A07 |
| SEC-4 | Auth anti-enumeration in shape **and** timing; limiter before hash. | OWASP A07 |
| SEC-5 | Rate limiting on all public endpoints; auth `(ip,email)`. | OWASP A04 |
| SEC-6 | Webhooks: verify signature → idempotency → process. | OWASP A08 |
| SEC-7 | SSRF denylist on user-supplied target hosts before any live scan. | OWASP A10 |
| SEC-8 | Scanners run only via hardened SandboxRunner (untrusted-code isolation). | OWASP A05 |
| SEC-9 | Secrets via Secrets Manager / env; never hardcoded or logged. | OWASP A05; SOC2 CC6.1 |
| SEC-10 | Ownership verification before live (domain/ip) scans. | abuse/legal |
| SEC-11 | Immutable audit log of privileged mutations. `compliance: audit-logging` (1y). | SOC2 CC7.2 |
| SEC-12 | Findings retained 90d; audit 1y. `compliance: data-retention`. | SOC2 CC; GDPR |
| SEC-13 | TLS everywhere (HTTPS only), HSTS, HTTP→HTTPS redirect. | OWASP A02 |
| SEC-14 | Input validation (zod) with field-level problem+json errors. | OWASP A03 |
| SEC-15 | Security headers (CSP, X-Content-Type-Options, Referrer-Policy), CSRF/Origin guard on state-mutating cookie routes. | OWASP A05 |

---

## 5. Key Workflows (each gets an e2e test W1..W8; UI-completable to terminal step)

- **W1 — Onboarding & first project.** Register → auto-org → create project. Terminal: project visible in dashboard. (Auth bootstrap.)
- **W2 — Connect a target.** Add a repo target (or domain target → receive ownership token → verify). Terminal: target listed with verification state.
- **W3 — Run a scan.** Select scanners → POST scan → worker executes → findings appear. Terminal: scan `completed` + findings rendered (not raw JSON).
- **W4 — Triage a finding.** Open finding detail → change status to fixed/ignored/snoozed (reason required for ignore/snooze). Terminal: finding moves out of the open list; audit row written.
  - Transitions: `open → fixed | ignored | snoozed`; `snoozed → open` (on expiry/re-seen); `fixed → open` (re-seen by a later scan). Re-scan reconciliation: a fingerprint seen again updates `lastSeenAt`; a `fixed` finding re-seen reopens.
- **W5 — Dashboard posture.** View rollups; filter by project/severity. Terminal: filtered list renders, empty-set shows explanatory empty state.
- **W6 — Webhook-triggered scan.** GitHub push webhook → verify → idempotent → scan enqueued. Terminal: scan appears in project history. (Webhook reception flow.)
- **W7 — Notification on new critical.** Scan completes with a new critical → notification queued → sent. Terminal: notification row `sent` (or `[contract]` when Resend unprovisioned). (Background job flow.)
- **W8 — Export findings.** Dashboard → export JSON. Terminal: downloaded artifact reflects current findings. (Authorization isolation tested cross-tenant + same-tenant on findings + `/:id` sub-routes.)

---

## 6. State Machines, Edge Cases, Concurrency

- **Scan lifecycle:** `queued → running → (completed | partial | failed)`. `partial` when ≥1 child job failed but ≥1 succeeded. Aggregator runs only when all children settle (BullMQ flow parent).
- **Idempotency:** webhook `deliveryId` unique constraint → duplicate delivery is a no-op 200 (TOCTOU closed at DB, not app). Scan create is **not** idempotent by default (explicit user action); webhook-triggered scans dedupe on `(targetId, commitSha)` within a 60s window.
- **Concurrency:** two triagers on one finding → optimistic update on `lastSeenAt`/`status`; last write wins with audit trail; multi-row status updates lock `ORDER BY id ASC`.
- **Boundary validation:** email RFC-valid + ≤254 chars; password 8–1024 chars (no truncation); target `value` length-capped; `severity`/`status`/`kind` enum-validated; pagination `page≥1`, `pageSize≤100`.
- **Empty vs zero-result:** a findings query returning 0 rows for valid filters renders "0 findings match — why & what to try", never an unchanged surface (DESIGN.md §5).

---

## 7. Error Contract (RFC 9457 problem+json)

All errors: `Content-Type: application/problem+json`, body:
```json
{
  "type": "https://sentinel.dev/errors/<code>",
  "title": "<human summary>",
  "status": <http>,
  "detail": "<specific, recoverable message>",
  "instance": "<request path>",
  "code": "<MACHINE_CODE>",
  "errors": [{ "field": "password", "message": "must be at least 8 characters" }]
}
```
Typed codes: `VALIDATION_FAILED` (400, with `errors[]`), `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404, also for cross-tenant), `CONFLICT` (409), `RATE_LIMITED` (429), `OWNERSHIP_REQUIRED` (422), `SSRF_BLOCKED` (422), `SERVICE_UNAVAILABLE` (503, with `dependency`). The UI binds `errors[].field` to the offending control (DESIGN.md §5 error-contract consumption).

---

## 8. UI Surface

Persistent SaaS app shell (sidebar 224px + content, max 1440px). Dark + light, runtime toggle following `prefers-color-scheme`, persisted. Every screen uses the Design System tokens (§ below); no screen invents its own.

| Screen | Route | States | Binds endpoints | Workflow | RBAC |
|---|---|---|---|---|---|
| Landing/marketing | `/` | static | — | entry | public |
| Register | `/register` | default, validating, error(field), success→redirect | `POST /v1/auth/register` | W1 | public |
| Login | `/login` | default, error(generic), loading | `POST /v1/auth/login` | W1 | public |
| Dashboard | `/app` | empty(no projects), loading(skeleton), error, populated | `GET /dashboard` | W5 | member+ |
| Projects | `/app/projects` | empty, loading, error, list | `GET/POST projects` | W1 | member+ |
| Project detail | `/app/projects/:id` | loading, error, targets+scans | projects/targets/scans | W2,W3 | member+ |
| Add target | `/app/projects/:id/targets/new` | default, validating, ssrf-error, ownership-pending | `POST targets`, `verify-ownership` | W2 | member+ |
| Scan detail | `/app/scans/:id` | running(progress pulse), partial, completed, failed | `GET scan` | W3 | member+ |
| Findings | `/app/findings` | empty(no findings), zero-result(filtered), loading, error, list | `GET findings` | W5,W8 | member+ |
| Finding detail | `/app/findings/:id` | loading, error, detail + triage | `GET/PATCH finding` | W4 | member+ |
| Settings | `/app/settings` | default, saved, error | orgs/members/integrations | W2,W6 | admin+ |

Every Key Workflow W1–W8 is completable end-to-end through these rendered screens to its terminal step (INV-13).

### Design System (peer to the UI Surface; precedence-bound)

- **Archetype:** `saas` (DESIGN.md Part II) — persistent app shell, composed information hierarchy, first-class tables/forms, full dark mode.
- **Fonts (self-hosted/preconnect):** Space Grotesk (display/headings 600/700), Inter (body 400/500), JetBrains Mono (CVE IDs, file paths, code). Not default `Inter`-as-brand — Space Grotesk carries the voice.
- **Type scale (base 14, ratio 1.2):** 9.7 / 11.7 / 14 / 16.8 / 20.2 / 24.2 / 29 px.
- **Spacing unit:** 4px grid; comfortable steps 12/16/24/32.
- **Radius:** 6px (sm 2 / lg 10 / xl 14). **Shadows:** none (flat, elevation by surface/border).
- **Color tokens:** full light + dark palettes (RESEARCH Design Language / CSS custom properties). Accent `#0c7ed4` used decisively on primary actions, active nav, high/critical severity.
- **Severity encoding:** never color-only — each severity is color **+** icon **+** text label (a11y AAA intent).
- **Motion budget (moderate/functional):** 120–320ms ease-out on state changes; 2px row hover lift; loading pulse on in-flight scans; `prefers-reduced-motion` honored.
- **Signature components:** app shell + active-state nav; findings data table (sort, filter, pagination, row actions, empty/loading/error); forms with inline validation; metric cards with trend; toasts; finding detail drawer; severity badges.
- **A11y:** WCAG 2.2 AA min (text ≥4.5:1, UI ≥3:1), visible 3px accent `:focus-visible`, full keyboard operability, semantic landmarks, Lighthouse 95+ target.

---

## 9. Environment & Configuration (mandatory)

All configuration via environment; `.env.example` committed, `.env` gitignored. In cloud, secret-typed vars resolve through AWS Secrets Manager references (`packages/config`); locally a sealed `.env` is used.

| Var | Required | Default (dev) | Purpose | Secret |
|---|---|---|---|---|
| `NODE_ENV` | yes | development | runtime mode | no |
| `DATABASE_URL` | yes | postgres://sentinel:sentinel@localhost:5432/sentinel | Postgres 18 DSN | yes |
| `REDIS_URL` | yes | redis://localhost:6379 | BullMQ/cache | no |
| `SESSION_SECRET` | yes | (dev placeholder, rotated) | session signing | yes |
| `APP_URL` | yes | https://localhost:3000 | base URL, cookie domain, CORS | no |
| `API_URL` | yes | http://localhost:4000 | api base for web | no |
| `SCANNERS_DOCKER_ENABLED` | no | false | enable real engine containers (else demo SCA) | no |
| `SCANNER_RUNTIME` | no | runc | `runsc` (gVisor) when available | no |
| `RESEND_API_KEY` | no | (placeholder) | email; absent → notifications `[contract]` | yes |
| `SLACK_WEBHOOK_URL` | no | (placeholder) | Slack alerts | yes |
| `SENTRY_DSN` | no | (empty) | error reporting | yes |
| `GITHUB_APP_*` / `GITHUB_WEBHOOK_SECRET` | no | (placeholder) | GitHub integration + webhook verify | yes |
| `GITLAB_WEBHOOK_TOKEN` | no | (placeholder) | GitLab webhook verify | yes |
| `AWS_SECRETS_PREFIX` | no | (empty) | Secrets Manager namespace | no |
| `RATE_LIMIT_WINDOW_MS` / `RATE_LIMIT_MAX` | no | 60000 / 100 | rate limiter | no |

**Deployment:** `docker-compose.yml` brings up web, api, worker, postgres, redis, proxy (TLS). Production: orchestrated multi-instance; daily Postgres snapshots; HTTPS only at the proxy with HSTS + HTTP→HTTPS redirect.

---

## 10. Test Plan (test-after)

- **Unit:** Finding normalizer adapters (golden fixtures per scanner), dedup fingerprinting, SSRF denylist, CVSS/severity mapping, problem+json helper, scoped-query wrapper, rate limiter.
- **Integration (supertest):** auth (register/login/logout/me + anti-enumeration), project/target/scan/finding CRUD, cross-tenant 404, same-tenant sub-resource authz, webhook signature + idempotency, ownership gate on live scans.
- **e2e (Playwright):** one spec per Key Workflow `W1..W8` driving the rendered UI to its terminal step (`ui-coverage` / INV-13).
- **Smoke:** `smoke-test.sh` — health, auth flow, CRUD cycle, scan→finding, negative path asserting field-level problem+json on the control.
- **Perf (comprehensive):** assert health/list p50 < 200ms; queue latency < 5s (SPEC §performance acceptance).
