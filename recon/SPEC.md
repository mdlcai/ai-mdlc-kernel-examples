# SPEC.md — Recon

> Behavioral contract. Precedence: RESEARCH.md > ARCHITECTURE.md > SPEC.md.
> build_depth: **comprehensive** — every feature, state transition, error path, race condition, idempotency guarantee, and security boundary is specified.
> All API routes are prefixed `/v1`. Errors follow §7 (RFC 9457 `application/problem+json`).

---

## 1. Feature Inventory

| ID | Feature | Type | Slice |
|---|---|---|---|
| F-01 | Project scaffold, config, DB, RLS, health | infra | backend |
| F-02 | Auth (register/login/logout/me) + RBAC + tenant context | vertical | api + UI (auth screens) |
| F-03 | Organizations & membership (RBAC roles, invites) | vertical | api + UI (settings/members) |
| F-04 | Assets registry (domains + CIDR) + authorization-to-scan | vertical | api + UI (assets, asset detail) |
| F-05 | Scan engine (`@recon/scanner`) + SSRF guard | infra | worker/lib |
| F-06 | Scan orchestration (trigger, run, snapshot, schedule) | vertical | api + worker + UI (scans) |
| F-07 | Change detection (snapshot diff, baseline, change feed) | vertical | api + worker + UI (changes) |
| F-08 | Alert channels + outbox delivery (Slack/webhook/email) | vertical | api + worker + UI (alerts) |
| F-09 | Realtime WebSocket feed (scan/change/alert events) | infra | api + UI (live indicators) |
| F-10 | Reports & exports (PDF/JSON/CSV) | vertical | api + worker + UI (reports) |
| F-11 | Audit log (append-only) + query | vertical | api + UI (audit) |
| F-12 | Dashboard (overview metrics, hierarchy) | vertical | api + UI (dashboard) |
| F-13 | Landing/marketing surface (ported template) | vertical | UI (landing) |
| F-14 | invariant-lint runner + continuation scaffold | infra | tooling |

Non-Goals (excluded): endpoint protection, internal network monitoring, SIEM, EDR/XDR, pentest automation, log management.

---

## 2. Data Model (Postgres, all tenant tables carry `org_id` + RLS)

- **users**: id, email (citext unique), password_hash (argon2id), display_name, created_at, last_login_at, mfa_enabled. *(global, not tenant-scoped)*
- **orgs**: id, name, slug (unique), created_at.
- **memberships**: id, org_id, user_id, role ∈ {owner,admin,analyst,viewer}, created_at. unique(org_id, user_id).
- **sessions**: id, user_id, org_id, token_hash, expires_at, created_at, revoked_at.
- **assets**: id, org_id, kind ∈ {domain,cidr}, value, label, authorization_status ∈ {pending,verifying,authorized,failed,revoked}, authz_method ∈ {dns_txt,http_file}, authz_token, scan_cadence ∈ {manual,hourly,daily,weekly}, created_at, last_scan_at. unique(org_id, kind, value).
- **scans**: id, org_id, asset_id, status ∈ {queued,running,completed,failed,canceled}, trigger ∈ {manual,scheduled}, started_at, finished_at, error, snapshot_id.
- **snapshots**: id, org_id, asset_id, scan_id, hash, data (jsonb: hosts[]→{ip, geo, ports[]→{port, proto, service, version, state}, certs[]→{fingerprint, subject, issuer, not_before, not_after, sans[]}}), created_at. (monthly partition by created_at)
- **baselines**: id, org_id, asset_id (unique), snapshot_id, accepted_by, accepted_at.
- **change_events**: id, org_id, asset_id, scan_id, type ∈ {PORT_OPENED,PORT_CLOSED,SERVICE_CHANGED,CERT_ROTATED,CERT_EXPIRING,HOST_NEW,HOST_GONE}, severity ∈ {info,low,medium,high,critical}, detail (jsonb), prior (jsonb), status ∈ {open,acknowledged}, acknowledged_by, created_at.
- **alert_channels**: id, org_id, kind ∈ {slack,webhook,email}, name, config (jsonb: slack→{webhook_url}; webhook→{url, secret}; email→{to[]}), enabled, min_severity, created_at.
- **outbox_delivery**: id, org_id, event_id (fk change_events), channel_id (fk alert_channels), status ∈ {pending,delivering,delivered,failed,dead}, attempts, last_error, payload (jsonb), signature, next_attempt_at, created_at, delivered_at. **unique(event_id, channel_id)** (INV-7).
- **reports**: id, org_id, kind ∈ {audit,trend}, format ∈ {pdf,json,csv}, params (jsonb), status ∈ {queued,rendering,ready,failed}, file_path, created_by, created_at.
- **audit_log**: id, org_id, actor_user_id, action, target_type, target_id, before (jsonb), after (jsonb), ip, created_at. **append-only** (no UPDATE/DELETE; INV-12).

### RLS
Every tenant table has `ENABLE ROW LEVEL SECURITY` + policy `USING (org_id = current_setting('app.tenant_id')::uuid)` (INV-3). The app sets `SET LOCAL app.tenant_id = $1` inside each request transaction via `withTenantScope` (INV-2).

---

## 3. API Surface (`/v1`)

### auth
- `POST /auth/register` — body {email, password, display_name, org_name}. Creates user + org + owner membership. **Anti-enumeration:** existing email returns the SAME 201-shaped success envelope and sends a "someone tried to register" email to the existing address; never reveals existence. Rate-limited per (ip, email) BEFORE hashing.
- `POST /auth/login` — body {email, password}. Sets httpOnly secure session cookie. **Uniform error** for unknown-email / wrong-password / rate-limited (same envelope + timing padding). Throttle fires inside authorize() before argon2 verify (INV-10).
- `POST /auth/logout` — revokes session.
- `GET /auth/me` — returns {user, org, role, memberships[]}.

### orgs
- `GET /orgs` — orgs the user belongs to.
- `POST /orgs` — create org (user becomes owner).
- `GET /orgs/:id/members` · `POST /orgs/:id/members` (invite by email; role) · `PATCH /orgs/:id/members/:mid` (role) · `DELETE /orgs/:id/members/:mid`. RBAC: owner/admin only for mutations.

### assets
- `GET /assets` — list (paginated, filter by status/kind, search). Index-backed <50ms (§9).
- `POST /assets` — body {kind, value, label, scan_cadence}. Validates domain/CIDR; CIDR size ≤ /16; value passes SSRF guard reject of internal ranges (INV-4). Starts `pending`.
- `GET /assets/:id` · `PATCH /assets/:id` · `DELETE /assets/:id`.
- `POST /assets/:id/authorize` — body {method}. Issues authz_token; returns instructions (DNS TXT record `recon-verify=<token>` or HTTP file `/.well-known/recon-verify.txt`). Sets `verifying`.
- `POST /assets/:id/verify` — performs the verification check (worker resolves TXT/HTTP). On success → `authorized`; on failure → `failed` with reason.

### scans
- `POST /assets/:id/scans` — trigger manual scan. **Rejected (409 ASSET_NOT_AUTHORIZED) if asset.authorization_status !== authorized** (INV-5). Throttled. Acquires per-asset lock; if locked → 409 SCAN_IN_PROGRESS.
- `GET /assets/:id/scans` — scan history.
- `GET /scans/:id` — scan detail + snapshot.
- `PUT /assets/:id/schedule` — body {cadence}. Upserts BullMQ Job Scheduler.

### changes
- `GET /changes` — feed (filter asset/type/severity/status, paginated).
- `GET /changes/:id` — detail incl. prior state.
- `POST /changes/:id/ack` — acknowledge (analyst+).
- `POST /assets/:id/baseline` — accept current snapshot as baseline (analyst+).

### alerts
- `GET/POST /alert-channels` · `GET/PATCH/DELETE /alert-channels/:id`. Webhook URL validated by SSRF guard (INV-4). Slack/webhook config secrets stored via SecretsService.
- `POST /alert-channels/:id/test` — sends a test delivery.
- `GET /alert-deliveries` — delivery log (status, attempts, last_error).

### reports
- `POST /reports` — body {kind, format, params}. Enqueues render.
- `GET /reports` · `GET /reports/:id` · `GET /reports/:id/download` (streams file; 404 until ready).

### audit
- `GET /audit-log` — query (filter actor/action/target/date; paginated). admin+.

### health
- `GET /health` — {status, services:{db, redis, worker}, version, time}. No auth.

### ws
- `WS /ws` — authenticated (session cookie); joins room `org:<id>`; events: `scan.updated`, `change.created`, `alert.delivered`.

---

## 4. Security Requirements (drives Security Audit Gate + COMPLIANCE.md)

| ID | Requirement | Enforcement |
|---|---|---|
| SEC-1 | **Tenant isolation** — no cross-tenant read/write | Postgres RLS + `withTenantScope`; cross-tenant lookup returns NOT_FOUND not FORBIDDEN; runtime smoke test asserts B cannot read A |
| SEC-2 | **Authorization-to-scan** — scan only verified-owned assets | INV-5; scan orchestrator hard-gate; authz proof recorded |
| SEC-3 | **SSRF / internal-range deny** — scan targets + webhook URLs | INV-4 TargetGuard: deny RFC1918/loopback/link-local/`169.254.169.254`/ULA; resolve + re-resolve at exec (anti-rebinding); no redirects on webhook send |
| SEC-4 | **AuthN/Z** — authenticated + RBAC on every non-public endpoint | JwtAuthGuard + RolesGuard; anti-enumeration on auth (uniform shape + timing) |
| SEC-5 | **Rate limiting** — public + scan-trigger + auth | @nestjs/throttler (Redis storage); auth throttle before hash; scan-trigger stricter bucket |
| SEC-6 | **Password security** (NIST 800-63B) | argon2id; min 12; accept ≥64; no composition/rotation; HIBP k-anonymity breach screen; rate-limit before hash |
| SEC-7 | **Idempotency** — outbox + webhook delivery | INV-7 unique(event_id, channel_id); INSERT…ON CONFLICT; receivers get Idempotency-Key |
| SEC-8 | **Webhook signing** — outbound payloads signed before send | INV-8 HMAC-SHA256 `signPayload` before `fetch`; `X-Recon-Signature` header |
| SEC-9 | **Secrets management** | Vault KV v2 in prod; env fallback local (ADR-009); never in DB/logs; redaction |
| SEC-10 | **Audit logging** (SOC2 CC7/CC8) | append-only audit_log; INV-12; all mutating actions logged |
| SEC-11 | **Encryption in transit** | HTTPS-only edge; HSTS; HTTP→HTTPS redirect (INV-11) |
| SEC-12 | **Encryption at rest** (`compliance:` data) | Postgres volume encryption (delegated → COMPLIANCE ⚠ delegated); channel secrets encrypted via SecretsService |
| SEC-13 | **Data retention** (`compliance:` 12 months) | snapshots/change_events retained ≥12 months (monthly partition); purge job beyond retention |
| SEC-14 | **Concurrency safety** | per-asset scan lock (Redis SETNX); deterministic ordering on multi-row locks; outbox row-claim via `FOR UPDATE SKIP LOCKED` |

### Domain-signal controls
- `has_webhooks` / `has_webhook_send` → outbox `pending_webhooks` semantics: sign-at-insert, Idempotency-Key, ≤5 retries → dead-letter, retention ≥ retry window, anomaly flag on repeated failures.
- `has_email` → `pending_emails` outbox: insert-in-transaction, drain worker, capped backoff → DLQ, never synchronous send.
- `has_dual_write` → outbox (no lost/double writes).
- `has_geo` → GeoIP enrichment; store country/asn only (no precise geolocation PII beyond IP already handled).
- `has_websocket` → authenticated WS, room-per-tenant, Redis adapter fan-out.

---

## 5. Key Workflows (each maps 1:1 to an e2e test `W<n>` driven through the UI to its terminal step — INV-13)

- **W1 — Auth bootstrap & org creation.** Register (email/password/org) → land on dashboard → `GET /auth/me` resolves. Terminal: authenticated dashboard rendered for the new org.
- **W2 — Asset Discovery Journey.** Register an asset (domain) → prove authorization (verify token) → asset moves to `authorized` and appears in inventory. Terminal: asset detail shows `authorized` + ready to scan.
- **W3 — Continuous Scanning Journey.** Set a scan schedule (cadence) on an authorized asset → trigger a scan → scan completes → snapshot visible with discovered ports/services/cert. Terminal: scan detail screen renders the snapshot result.
- **W4 — Change Detection & Response Journey.** Accept a baseline → a subsequent scan produces a diff → change event appears in the change feed with prior state → analyst acknowledges it. Terminal: change detail shows delta + acknowledged status.
- **W5 — Alert Routing Journey.** Configure an alert channel (webhook/slack/email) → send a test delivery → delivery shows `delivered` in the delivery log (or contract-correct status if external unprovisioned). Terminal: alert delivery log row visible through UI.
- **W6 — Audit & Trend Reporting Journey.** Generate a report (format PDF/JSON/CSV) → report renders → download the artifact. Terminal: report marked `ready` and downloadable through UI.
- **W7 — Authorization isolation (multi-tenant).** User A creates an asset; user B (different org) cannot see or read it. Terminal: B receives 404 on A's asset through the UI/api.

---

## 6. AI/Decision Logic, Error Handling, Edge Cases

- **Scan engine decisioning:** for each target host, TCP-connect probe a configured port set (top-N + user ports); on open port, attempt lightweight banner/service heuristic + (for 443/8443/TLS ports) TLS handshake → parse cert (subject/issuer/SAN/validity/fingerprint). Domains: resolve A/AAAA, optional CT enrichment (crt.sh) for subdomains. GeoIP enrich each IP. Timeouts bounded; partial results recorded with per-host error.
- **Diff algorithm:** normalize snapshot (sort hosts/ports/cert fingerprints), hash; if hash == baseline hash → no change. Else compute set differences → emit typed events; CERT_EXPIRING fires when not_after within 30d; severity mapping table.
- **Edge cases:** asset with no resolvable hosts (scan completes, snapshot empty, surfaced as "0 hosts responded — why/what to try"); CIDR too large (reject at validation); scan timeout (status failed + retry policy); duplicate change event (idempotent via unique); external API down (degrade, mark enrichment partial); outbox channel disabled mid-flight (skip, log).

## 7. Structured Error Contract (RFC 9457 `application/problem+json`)

```json
{ "type": "https://recon.app/errors/<code>", "title": "<human>", "status": <http>,
  "code": "<MACHINE_CODE>", "detail": "<message>",
  "errors": [{ "field": "email", "message": "must be a valid email", "code": "invalid_email" }],
  "instance": "/v1/assets", "traceId": "<uuid>" }
```
Typed codes: `VALIDATION_FAILED`(400), `UNAUTHENTICATED`(401), `FORBIDDEN`(403), `NOT_FOUND`(404), `CONFLICT`(409), `ASSET_NOT_AUTHORIZED`(409), `SCAN_IN_PROGRESS`(409), `RATE_LIMITED`(429), `INTERNAL`(500), `SERVICE_UNAVAILABLE`(503, with `dependency`). Field-level `errors[]` populated on validation failures; the UI maps these onto the offending input (Per-Feature Design Pass error-contract rule).

## 8. UI Surface

Layout: persistent **app shell** (224px sidebar + topbar with org switcher, page title, primary action) per `saas` archetype. Tokens from DESIGN-TEMPLATE.html (`:root`, ADR-010). Light mode only. Every screen specifies empty/loading/error/success states.

| Route | Purpose | Key components | States | API | Workflow | RBAC |
|---|---|---|---|---|---|---|
| `/` (landing) | Marketing/home, ported template | nav, hero+console mock, features bento, CTA, footer | static; CTA → /register | — | — | public |
| `/register` | Create account + org | form (email, password w/ strength, org name) | empty/loading/error(field-mapped)/success→/dashboard | POST /auth/register | W1 | public |
| `/login` | Sign in | form | empty/loading/error(uniform)/success | POST /auth/login | W1 | public |
| `/dashboard` | Overview: lead metric (assets monitored) → supporting (open changes, recent scans) → detail (recent change feed) | metric cards w/ trend, change list, live indicator | empty(first-use: "Add your first asset")/loading(skeleton)/error/success | GET /assets,/changes,/scans | W1 terminal | viewer+ |
| `/assets` | Asset inventory table | data table (sort/filter/search/pagination, status badges, row actions), "Add asset" | empty/loading(skeleton rows)/error/success; 0-result search explained | GET/POST /assets | W2 | viewer+ |
| `/assets/:id` | Asset detail: authorization, schedule, scan history, current snapshot | authz panel + token instructions, cadence selector, scans table, snapshot viewer | states incl. pending-authorization, verifying, authorized; empty scans | GET /assets/:id, /scans, authorize/verify | W2 terminal, W3 | analyst+ for mutations |
| `/scans/:id` | Scan run detail + snapshot result | host/port/service/cert table, status, diff link | loading(running)/empty(0 hosts)/error(failed w/ reason)/success | GET /scans/:id | W3 terminal | viewer+ |
| `/changes` | Change feed | table (type/severity badges, asset, time, ack), filters | empty("No changes — baselines clean")/loading/error/success | GET /changes, ack | W4 | viewer+ |
| `/changes/:id` | Change detail | delta view (before/after), severity, ack action | states; ack success inline | GET /changes/:id, ack | W4 terminal | analyst+ |
| `/alerts` | Alert channels + delivery log | channel cards, add/edit drawer (slack/webhook/email), test button, deliveries table | empty("Add a channel")/loading/error(field-mapped)/success | alert-channels, /test, alert-deliveries | W5 terminal | admin+ |
| `/reports` | Reports list + generate | generate drawer (kind/format/params), reports table w/ status, download | empty/loading(rendering)/error/success(download) | reports, download | W6 terminal | analyst+ |
| `/audit` | Audit log viewer | table (actor/action/target/time), filters | empty/loading/error/success | GET /audit-log | — | admin+ |
| `/settings/members` | Org members + roles + invite | members table, invite form, role selector | empty/loading/error/success | orgs members | — | owner/admin |

Every non-internal endpoint is reachable from a screen; every §5 workflow completes end-to-end through the UI including its terminal step.

## 9. Performance Targets (per RESEARCH Success Metrics)

| Target | Endpoint/op | Approach |
|---|---|---|
| Asset inventory search <50ms | GET /assets | indexed (org_id, status), trigram index on value/label, keyset pagination |
| Change detection alert <15 min | scan→diff→outbox | queue priority + per-asset lock; diff O(n) over normalized sets |
| Scan completion <30 min / /24 | scan worker | bounded concurrency, per-host timeout, parallel host probing |
| Alert delivery <2 min | outbox relay | poll/notify ≤ seconds; retries with backoff |
| WS event broadcast | gateway | Redis adapter; room-per-tenant |

## 10. Environment & Configuration  *(mandatory section)*

All config via env (12-factor). `.env.example` documents every var (INV-14). Secrets resolved by `SecretsService` (Vault in prod, env fallback local — ADR-009).

| Var | Description | Example | Required |
|---|---|---|---|
| `NODE_ENV` | environment | production | yes |
| `API_PORT` | NestJS port | 3001 | yes |
| `WEB_PORT` | Next.js port | 3000 | yes |
| `DATABASE_URL` | Postgres DSN | postgres://recon:recon@db:5432/recon | yes |
| `REDIS_URL` | Valkey/Redis URL | redis://redis:6379 | yes |
| `JWT_SECRET` | session/JWT signing | (32+ random bytes) | yes |
| `SESSION_COOKIE_DOMAIN` | cookie domain | localhost | no |
| `APP_BASE_URL` | public base (https) | https://localhost | yes |
| `SCAN_PORTS` | default port set | 21,22,25,80,443,3389,8080,8443 | no |
| `SCAN_CONCURRENCY` | host probe concurrency | 50 | no |
| `SSRF_DENY_EXTRA` | extra CIDR deny list | 10.0.0.0/8 | no |
| `VAULT_ADDR` / `VAULT_TOKEN` | Vault (prod) | https://vault:8200 | no (prod yes) |
| `RESEND_API_KEY` | email (Resend) | re_xxx | no (email channel) |
| `SENTRY_DSN` | error reporting | https://…@sentry | no |
| `GEOIP_DB_PATH` | MaxMind mmdb path | /data/GeoLite2-City.mmdb | no |
| `HIBP_ENABLED` | breach screening on/off | true | no |
| `RETENTION_MONTHS` | data retention | 12 | no |

Deploy: `docker-compose.yml` (web, api, worker, db, redis, nginx). nginx terminates TLS (HTTPS only), redirects HTTP→HTTPS, sets HSTS, and sets header buffers (`large_client_header_buffers 8 32k`). Node serving paths use `--max-http-header-size=32768`.

## 11. Testing Strategy (TDD)
- Unit: services, SSRF guard, diff algorithm, scan engine, password policy, outbox.
- Integration: each controller against a test Postgres (RLS asserted), throttling, idempotency, concurrency race tests.
- e2e (Playwright): one spec per workflow W1–W7 to terminal step (INV-13).
- Verification Gate sequence: install → typecheck → lint (max-warnings 0) → tests → build → `lint:invariants`.
