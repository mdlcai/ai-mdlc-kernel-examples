# SPEC.md — CyberOps (comprehensive)

**Behavioral contract.** Another engineer/AI could recreate the system from this file. Precedence: RESEARCH > ARCHITECTURE > SPEC. Non-Goals are not specced.

---

## §1 Feature Inventory (build plan)

Vertical slices (each = endpoints + screens + e2e). Ordered by dependency.

| F | Feature | Type | Workflow |
|---|---------|------|----------|
| F-01 | Project scaffold, config, DB, health, error envelope, structured logging | infra | — |
| F-02 | Tenancy + RBAC + tenant-scope chokepoint + RLS | infra/backend | all |
| F-03 | Auth: register + login + logout + session | user-facing | all |
| F-04 | Auth: password reset (request + confirm) | user-facing | all |
| F-05 | Assets & Authorization Scope (CRUD + scope verification) | user-facing | W2 |
| F-06 | Scans (create/list/detail) + adapter engine + BullMQ + live WS progress | user-facing | W2 |
| F-07 | Findings (list/detail) + CVSS/severity + correlation | user-facing | W2,W5 |
| F-08 | AI Pentest (create/list/detail) + planner + attack paths + approval gate | user-facing | W1 |
| F-09 | OSINT & Threat investigation (submit indicator + STIX feed correlation) | user-facing | W3 |
| F-10 | Compliance mapping (frameworks/controls/posture + evidence) | user-facing | W4 |
| F-11 | Remediation & Continuous Retesting (status + schedule + revalidate) | user-facing | W5 |
| F-12 | Reports export (PDF/JSON/CSV) | user-facing | W1–W4 |
| F-13 | Alerts: email outbox worker + outbound webhooks | backend | W3,W5 |
| F-14 | Inbound webhooks receiver (verify→idempotency→process) | backend | W3 |
| F-15 | Audit log (append-only hash chain) + viewer | user-facing | all |
| F-16 | Dashboard (risk score, severity dist, recent scans/findings) | user-facing | all |
| F-17 | Marketing landing (ported from template) + settings/account/integrations | user-facing | — |

Interface Contract Validation cadence: N = max(1, min(7, ceil(17/4))) = **5** (run after features 5, 10, 15, and once after the final feature).

---

## §2 Data Model (PostgreSQL 18, TypeORM)

All tenant-owned tables carry `tenant_id uuid NOT NULL` (RLS policy `tenant_id = current_setting('app.tenant_id')::uuid`). PKs `id uuid DEFAULT uuidv7()`. Timestamps `created_at`, `updated_at timestamptz`.

- **organizations**(id, name, slug UNIQUE, created_at)
- **users**(id, email CITEXT UNIQUE, password_hash, name, mfa_secret NULL, created_at) — password_hash bcrypt cost 12
- **memberships**(id, tenant_id→org, user_id→user, role enum[owner,admin,analyst,viewer], UNIQUE(tenant_id,user_id))
- **sessions**(id, user_id, tenant_id, token_hash, expires_at, revoked_at NULL, ip, user_agent)
- **password_reset_tokens**(id, user_id, token_hash, expires_at, used_at NULL)
- **assets**(id, tenant_id, kind enum[domain,ip,cidr,url,repo,cloud], value, criticality enum[low,med,high,crit], tags jsonb, owner_user_id, created_at)
- **authorization_scopes**(id, tenant_id, asset_id, method enum[dns_txt,file,meta,manual], token, status enum[pending,verified,revoked], verified_at NULL) — deny-by-default; a scan needs a `verified` scope covering the target
- **hosts**(id, tenant_id, asset_id, ip, hostname, first_seen, last_seen) · **services**(id, tenant_id, host_id, port, protocol, service, product, version)
- **scan_runs**(id, tenant_id, asset_id, profile enum[recon,vuln,full], status enum[queued,running,succeeded,failed,cancelled], idempotency_key, requested_by, started_at, finished_at, stats jsonb) — UNIQUE(tenant_id, idempotency_key)
- **findings**(id, tenant_id, scan_run_id NULL, asset_id, host_id NULL, title, description, severity enum[critical,high,medium,low,info], cvss_score numeric(3,1), cvss_vector, cwe, source, validation_state enum[unvalidated,validating,validated,failed], status enum[open,triaged,remediating,resolved,accepted,false_positive], correlation_key, payload jsonb, first_seen, last_seen)
- **finding_validations**(id, tenant_id, finding_id, requested_by, method, result enum[validated,failed,inconclusive], evidence_id NULL, created_at)
- **retests**(id, tenant_id, finding_id, scheduled_for NULL, status enum[scheduled,running,done], result_finding_id NULL, created_at)
- **pentest_runs**(id, tenant_id, asset_id, status enum[queued,planning,running,awaiting_approval,succeeded,failed], phase enum[recon,analysis,validation,report], planner enum[llm,rule_based], created_by, created_at)
- **attack_paths**(id, tenant_id, pentest_run_id, steps jsonb[technique(ATT&CK id),cwe,target,outcome], risk_score numeric)
- **evidence**(id, tenant_id, pentest_run_id NULL, finding_id NULL, kind, blob_ref, encrypted bool, created_at)
- **approval_requests**(id, tenant_id, pentest_run_id, action, status enum[pending,approved,denied], decided_by NULL, decided_at NULL)
- **indicators**(id, tenant_id, type enum[domain,ip,url,hash,email], value, first_seen) · **threat_feeds**(id, tenant_id, name, kind enum[stix,taxii,manual], last_ingested_at) · **investigations**(id, tenant_id, indicator_id, summary, correlations jsonb, created_by, created_at)
- **frameworks**(id, key enum[soc2,pci_dss_v4,owasp_top10], name) [global seed] · **controls**(id, framework_id, ref, title, description) [global seed]
- **control_mappings**(id, tenant_id, control_id, finding_id NULL, status enum[passing,failing,at_risk,not_applicable], evidence_id NULL, updated_at)
- **posture_snapshots**(id, tenant_id, framework_id, passing, failing, at_risk, grade char(1), created_at)
- **reports**(id, tenant_id, kind enum[findings,compliance,pentest], format enum[pdf,json,csv], status enum[queued,ready,failed], file_ref, created_by, created_at)
- **webhook_endpoints**(id, tenant_id, url, secret, events text[], active bool)
- **pending_emails**(id, tenant_id NULL, to_address, template_key, payload jsonb, status enum[pending,sent,failed,dlq], attempts int, sent_at NULL, error_text NULL, business_ref_id) — **UNIQUE(to_address, template_key, business_ref_id)**, INDEX(status, created_at)
- **pending_webhooks**(id, tenant_id, target_url, event_type, payload jsonb, signature, status enum[pending,delivered,failed,dlq], attempts int, next_retry_at, delivered_at NULL, last_error NULL), INDEX(status, next_retry_at)
- **inbound_webhook_events**(id, provider, event_id, received_at, processed_at NULL) — **UNIQUE(provider, event_id)**
- **outbox_events**(id, tenant_id NULL, aggregate, event_type, payload jsonb, target enum[search,email,webhook], claimed_at NULL, processed_at NULL, created_at), INDEX(target, claimed_at)
- **audit_events**(id, tenant_id NULL, actor_type enum[user,agent,system], actor_id, action, target_scope, outcome, prev_hash, hash, created_at) — **append-only**

Migrations reversible. Seeds: roles, severities, frameworks+controls (SOC2 TSC, PCI DSS v4, OWASP Top10), demo org + admin user + sample assets/findings, offline Nuclei-style template corpus.

---

## §3 API Surface (REST, prefix `/v1`, JSON)

Auth = JWT httpOnly cookie unless marked public. All list endpoints paginated (`?page,limit`), filterable. Errors use §7 envelope.

**Auth (public unless noted):**
- `POST /v1/auth/register` {email,password,name,orgName} → 201 {user,org}; anti-enumeration (same shape for existing email; sends notice); breached-password screen; rate-limit (ip,email)
- `POST /v1/auth/login` {email,password} → 200 set-cookie; uniform error+timing; rate-limit before hash compare
- `POST /v1/auth/logout` (auth) → 204
- `GET /v1/auth/me` (auth) → 200 {user, memberships, activeTenant}
- `POST /v1/auth/password-reset/request` {email} → 202 (always, anti-enumeration) · `POST /v1/auth/password-reset/confirm` {token,password} → 200

**Assets/Scope (auth):** `GET/POST /v1/assets` · `GET/PATCH/DELETE /v1/assets/:id` · `GET /v1/assets/:id/hosts` · `POST /v1/assets/:id/scope` (start verification) · `POST /v1/scopes/:id/verify` · `GET /v1/scopes/:id`
**Scans (auth):** `GET/POST /v1/scans` (POST body {assetId,profile,idempotencyKey}) · `GET /v1/scans/:id` · `POST /v1/scans/:id/cancel` · `GET /v1/scans/:id/results` · WS `scan:<id>` progress
**Findings (auth):** `GET /v1/findings` · `GET /v1/findings/:id` · `PATCH /v1/findings/:id` (status/triage) · `POST /v1/findings/:id/validate` · `POST /v1/findings/:id/retest` · `GET /v1/findings/:id/timeline`
**Pentest (auth):** `GET/POST /v1/pentests` · `GET /v1/pentests/:id` · `GET /v1/pentests/:id/attack-paths` · `POST /v1/pentests/:id/approvals/:aid/decide` {approve|deny}
**OSINT (auth):** `POST /v1/osint/investigate` {type,value} · `GET /v1/osint/investigations` · `GET /v1/osint/investigations/:id` · `GET /v1/osint/feeds` · `POST /v1/osint/feeds/:id/ingest`
**Compliance (auth):** `GET /v1/compliance/frameworks` · `GET /v1/compliance/:frameworkKey/posture` · `GET /v1/compliance/:frameworkKey/controls` · `POST /v1/compliance/mappings/:id/reassess`
**Reports (auth):** `POST /v1/reports` {kind,format,filters} → 202 · `GET /v1/reports` · `GET /v1/reports/:id` · `GET /v1/reports/:id/download` (returns complete `download_url` consumed verbatim by client — never double-prefixed)
**Alerts/Webhooks:** `GET/POST /v1/webhook-endpoints` (auth) · `POST /v1/webhooks/inbound/:provider` (public, signature-verified)
**Audit (auth, admin+):** `GET /v1/audit`
**Dashboard (auth):** `GET /v1/dashboard/summary`
**Health (public):** `GET /api/health` → {status, services{db,redis,queue}, version}

---

## §4 Security Requirements (compliance-tagged)

- **SEC-1** (compliance:OWASP-A01, SOC2 CC6.1) Tenant isolation via RLS + `TenantScopedRepository`; cross-tenant read/write → E.NOT_FOUND. INV-1.
- **SEC-2** (compliance:OWASP-A01) Object-level authz `assertResourceAccess` on user-owned resources + sub-resources. INV-13.
- **SEC-3** (compliance:NIST-800-63B) Password verifier: bcrypt cost 12, min length 8 (accept ≤64, no composition rules, no forced rotation), **breached-password screening (HIBP k-anonymity)** on register + change; no knowledge-based recovery.
- **SEC-4** (compliance:OWASP-A07) Anti-enumeration on register/login/reset — uniform response shape + timing (padded or rate-limited).
- **SEC-5** (compliance:OWASP-A04, R6) Rate limiting: Redis-backed throttler on every public endpoint; auth limiter fires before hash compare, keyed (ip,email). INV-6.
- **SEC-6** (compliance:OWASP-A10, R2) SSRF guard on all scan targets: resolve→validate allowlist→block private/reserved→pin IP. INV-2.
- **SEC-7** (R1) Authorization scope deny-by-default: verified scope required; re-checked before every network action. INV-3.
- **SEC-8** (compliance:SOC2 CC7, R5) Append-only hash-chained audit for every security action (user/agent, target scope, timestamp, action, outcome). INV-5.
- **SEC-9** (has_webhooks) Inbound webhook: verify signature before DB; idempotency UNIQUE(provider,event_id). INV-4/INV-8.
- **SEC-10** (compliance:OWASP-A02, GDPR/SOC2, R8) PII/findings encrypted at rest (pgcrypto/app AES-256-GCM for evidence + sensitive columns; at-rest volume encryption delegated to prod infra — ⚠ delegated in dev, documented); redaction on export; secure retention/deletion (audit 2y).
- **SEC-11** (R7) LLM agent safety: target-derived text is untrusted data; allowlisted sandboxed tools; scope re-check + human approval before state-changing actions.
- **SEC-12** (compliance:OWASP-A03) Supply chain: pinned deps + lockfile; `npm audit`/Trivy in CI; scanner binaries pinned by version+checksum.
- **SEC-13** CSRF/Origin guard on state-mutating public routes; WebSocket Origin check + JWT + per-conn rate budget.
- **SEC-14** Secrets: AWS Secrets Manager (prod) / env + gitignored .env (dev); no hardcoded secrets. INV-7.

---

## §5 Key Workflows (must be UI-completable end-to-end, incl. terminal step)

**W1 AI-Powered Automated Penetration Test:** select verified asset + scope → create pentest run → planner runs recon→analysis (phases visible live) → validation actions requiring approval surface an **approval request** the user approves/denies in the UI → attack paths + risk-scored findings + evidence produced → user reviews and **exports** a pentest report (PDF). Terminal: report export.
**W2 Asset Discovery & Vulnerability Assessment:** add asset → verify authorization scope → launch scan (recon/vuln/full) → **live progress** via WS → discovered hosts/services + findings (CVSS/severity) correlated to asset + threat intel → prioritized list. Terminal: findings triaged in UI.
**W3 Threat & OSINT Investigation:** submit indicator (domain/ip/url/hash/email) → platform gathers authorized public intel + correlates with assets/findings/feeds (STIX) → investigation summary with evidence + risk context shown; alert can be sent (email/webhook). Terminal: investigation reviewed/exported.
**W4 Compliance & Security Posture Assessment:** select framework (SOC2/PCI DSS v4/OWASP) → findings/configs mapped to controls → gaps + letter grade + posture shown → **export** audit-ready evidence report. Terminal: compliance report export.
**W5 Finding Remediation & Continuous Retesting:** select finding → remediation guidance + status tracking → schedule/initiate **retest** → scanner/pentest revalidates → result recorded, risk score updated, audit history maintained. Terminal: retest completed + status resolved in UI.

Each workflow has exactly one Playwright e2e test named `test('W<n> …')` driving it through the rendered UI to its terminal step (INV-12 ui-coverage).

---

## §6 State Transitions & Concurrency (comprehensive)

- **scan_runs:** queued→running→(succeeded|failed|cancelled). Cancel only from queued/running. Idempotency: UNIQUE(tenant_id,idempotency_key) — duplicate POST returns the existing run (no double scan). Worker claims via `UPDATE … SET status='running' WHERE id=? AND status='queued'` (optimistic, single-claim). Race: two workers → only one row-update succeeds.
- **findings.validation_state:** unvalidated→validating→(validated|failed). Only one active validation per finding (guard). **findings.status:** open→triaged→remediating→resolved; open→accepted|false_positive. Retest of a resolved finding that recurs reopens (new finding linked via correlation_key).
- **pentest_runs:** queued→planning→running→(awaiting_approval↔running)→(succeeded|failed). awaiting_approval blocks until approval_requests decided; deny → skips the gated action, continues.
- **outbox_events:** created→claimed(`UPDATE … SET claimed_at=now() WHERE id=? AND claimed_at IS NULL`)→processed. Idempotent claim prevents double-send.
- **pending_emails/pending_webhooks:** pending→(sent/delivered|failed→retry w/ capped backoff|dlq after max attempts). Signature computed at INSERT for webhooks (not on retry).
- **authorization_scopes:** pending→verified→revoked. Scan submission requires a currently-`verified` scope covering the resolved target IP/host; revocation mid-scan aborts remaining network actions.

---

## §7 Error Handling & Contract

Typed envelope: `{ error: { code, message, details?: [{ field, issue }] } }`. Codes: `E.VALIDATION` (400, `details` field-level from Zod), `E.UNAUTHENTICATED` (401), `E.FORBIDDEN` (403), `E.NOT_FOUND` (404, also for cross-tenant), `E.CONFLICT` (409), `E.RATE_LIMITED` (429), `E.SERVICE_UNAVAILABLE` (503, dependency name), `E.INTERNAL` (500). Frontend maps `details` onto the offending input; never shows a bare top-level message for a field error, stack trace, or `[object Object]`. Unprovisioned-external state returns 503 with `{error:{code:'E.SERVICE_UNAVAILABLE', dependency}}` (contract-correct).

---

## §8 UI Surface (screen inventory)

Design tokens = template `:root` (ADR-004). Every screen: empty/loading/error/success states designed; `:focus-visible`, ≥44px hit targets, WCAG 2.2 AA; TanStack Query bindings.

| Screen | Route | Binds (endpoints) | Workflow | States | RBAC |
|--------|-------|-------------------|----------|--------|------|
| Landing (marketing) | `/` | — | — | static | public |
| Login | `/login` | POST /auth/login | all | form+error | public |
| Register | `/register` | POST /auth/register | all | form+error | public |
| Password reset | `/reset`, `/reset/confirm` | reset request/confirm | all | form+error | public |
| Dashboard | `/dashboard` | GET /dashboard/summary | all | empty/loading/error/data | viewer+ |
| Assets list | `/assets` | GET/POST /assets | W2 | empty/loading/error/data | analyst+ |
| Asset detail + scope | `/assets/:id` | asset, hosts, scope verify | W2 | all | analyst+ |
| Scans list + new | `/scans`, `/scans/new` | GET/POST /scans | W2 | all | analyst+ |
| Scan detail (live) | `/scans/:id` | scan, results, WS | W2 | queued/running(live)/done/error | analyst+ |
| Findings list | `/findings` | GET /findings | W2,W5 | empty/loading/error/data+filters | viewer+ |
| Finding detail | `/findings/:id` | finding, timeline, validate, retest | W2,W5 | all | analyst+ |
| Pentest list + new | `/pentests`, `/pentests/new` | GET/POST /pentests | W1 | all | analyst+ |
| Pentest detail | `/pentests/:id` | run, attack-paths, approvals decide | W1 | planning/running/awaiting_approval/done | analyst+ |
| OSINT investigate | `/osint` | investigate, investigations, feeds | W3 | all | analyst+ |
| Investigation detail | `/osint/:id` | investigation | W3 | all | analyst+ |
| Compliance | `/compliance` | frameworks, posture, controls | W4 | all | viewer+ |
| Framework detail | `/compliance/:key` | posture, controls, reassess | W4 | all | analyst+ |
| Reports | `/reports` | create/list/download | W1–W4 | queued/ready/failed | analyst+ |
| Audit log | `/audit` | GET /audit | all | all | admin+ |
| Settings/Account | `/settings` | me, webhook-endpoints | — | all | admin+ (webhooks), self (account) |

Every non-internal endpoint above is reachable from a screen; every workflow terminal step (approve, export, triage, retest) is UI-completable.

---

## §9 Performance & Non-Functional (acceptance criteria)

- Dashboard load < 3s (indexed aggregates, cached summary); API p95 < 200ms (excl. long-running scans, which are async); sustained 50k concurrent (stateless app, Redis-backed throttler + WS adapter, pooling). Comprehensive: representative load test (autocannon) against the deploy.
- Structured JSON logs (pino); Sentry error capture; `/api/health` + `/metrics`.
- Backups: automated daily + PITR (prod RDS; docker volume snapshot note for local).

---

## §10 Environment & Configuration (mandatory)

All config via env (12-factor). Every var appears in `.env.example` (INV via completeness gate).

| Var | Req | Used by | Note |
|-----|-----|---------|------|
| `NODE_ENV` | yes | all | development/production |
| `API_PORT` | yes | api | default 3001 |
| `WEB_PORT` | yes | web | default 3000 |
| `DATABASE_URL` | yes | api/worker | postgres:// |
| `REDIS_URL` | yes | api/worker | redis:// (noeviction) |
| `JWT_SECRET` | yes | api | ≥32 bytes; Secrets Manager in prod |
| `COOKIE_SECRET` | yes | api | session cookie signing |
| `SESSION_TTL_HOURS` | no | api | default 168 |
| `CORS_ORIGIN` | yes | api | web origin allowlist (also WS Origin) |
| `RESEND_API_KEY` | no | worker | email disabled if absent (queue holds pending) |
| `EMAIL_FROM` | no | worker | default noreply@cyberops.local |
| `SENTRY_DSN` | no | api/web | error reporting disabled if absent |
| `LLM_API_KEY` | no | api/worker | Claude; falls back to rule-based planner if absent |
| `LLM_MODEL` | no | api/worker | default claude-sonnet-5 |
| `NUCLEI_BIN` | no | worker | path to nuclei; adapter skipped if absent |
| `TRIVY_BIN` | no | worker | path to trivy; adapter skipped if absent |
| `AWS_REGION`/`AWS_SECRETS_PREFIX` | no | api | Secrets Manager in prod |
| `SCAN_EGRESS_ALLOW_PRIVATE` | no | worker | default false (SSRF guard); never true in prod |
| `NEXT_PUBLIC_API_BASE` | yes | web | e.g. http://localhost:3001/v1 |
| `NEXT_PUBLIC_WS_URL` | yes | web | ws base |
| `NODE_OPTIONS` | yes | api/web | `--max-http-header-size=32768` (INV-14) |

Protocol (RESEARCH `HTTPS only`): prod forces HTTPS + HSTS + HTTP→HTTPS redirect (managed cert/Certbot via proxy); local dev uses HTTP on localhost (documented, HTTPS-later path in QUICKSTART).

---

## §11 Testing Plan (TDD)

Per-feature: unit (services, guards, SSRF resolver, password verifier, hash chain), integration (controllers via supertest against a test DB, concurrency race tests for scan claim + outbox claim), security (tenant isolation, IDOR on sub-resources, anti-enumeration, webhook verify-before-DB, rate-limit-before-hash). Web: component + Playwright e2e (one per W1–W5). Invariant lint runner (`scripts/invariant-lint.mjs`) implements all check types incl. ui-coverage, wired into the Verification Gate. Smoke: `smoke-test.sh` (health, auth flow, CRUD, capability-generalization, negative paths, large-header) + `SMOKE-TEST.md`.

## §12 Non-Goals (not specced)
Live exploit-payload execution against arbitrary targets (ADR-005); mobile/native apps (web only); billing/payments (pricing page is marketing only); customer-managed on-prem deployment; real-time collaborative editing.
