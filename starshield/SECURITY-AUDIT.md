# SECURITY-AUDIT.md — STΛRSHΞILD

Security Audit Gate narrative (`build_depth: comprehensive`), measured against SPEC.md §4 and ARCHITECTURE.md §9–§10.

| Item | Value |
|---|---|
| Pass 1 (initial scan) | commit 631d000, `artifacts/security/pass-1.md` |
| Remediation batch 1 | ADR-059, VERSION 0.1.0 → 0.1.1 |
| Pass 2 (re-scan) | commit 3697f5b, `artifacts/security/pass-2.md` |
| Remediation batch 2 | ADR-060, VERSION 0.1.1 → 0.1.2 |
| Final check | Every evidence pointer below was re-read against the current tree. |
| Verdict | **PASS WITH ADVISORY.** There are 0 open CRITICAL, HIGH or MEDIUM findings. SAST, secret detection and licence compliance were reviewed by hand because no tool was available for them. |

---

## 1. Trust boundaries

| # | Boundary | Who is on each side | Control at the crossing | Evidence |
|---|---|---|---|---|
| B1 | Internet → Caddy edge | Anonymous browsers and sensors → the TLS edge | TLS (`tls internal` locally, ACME in prod). HSTS is sent for hosts other than localhost. `x-middleware-subrequest` is stripped. `/metrics` returns 404. Static security headers are set on every response. | deploy/Caddyfile:8-18, :21, :26, :29-30, :33-34 |
| B2 | Caddy → web (Next SSR) | Edge → the Next server | The middleware mints a per-request CSP nonce and deletes `x-middleware-subrequest` again. | apps/web/src/middleware.ts:14-22, :46 |
| B3 | Caddy → api (`/api/*`, `/ingest/*`) | Edge → Express | `trust proxy` = 1 hop. The global per-IP limiter runs, then session, then CSRF, then the password-change guard, then `authorize`. | apps/api/src/app.ts:27, :65, :66; compose.yaml:16 |
| B4 | web → api (server-side fetch) | The Next server, acting for a user → Express | The base URL is fixed (`API_INTERNAL_URL`) and the paths are literal. Only `ss_session`, `x-request-id` and the client IP written by the edge are forwarded. Responses are `no-store` with a 5 s timeout. | apps/web/src/lib/server-fetch.ts:6-31 |
| B5 | Sensor → ingest | A machine source → Express | HMAC over the raw body. The raw-body route is mounted before the JSON parser and before session/CSRF. | apps/api/src/app.ts:62-63; apps/api/src/routes/ingest.ts:21-28 |
| B6 | api/worker → Postgres (via pgbouncer) | App → the system of record | One DB role, which owns the schema (see SEC-6). Queries are tagged `$queryRaw` / Prisma, so there is no `*Unsafe` raw SQL. | compose.yaml:8-9 |
| B7 | api/worker → Redis | App → the cache, limiter, nonce and queue store | `requirepass` is set, and the password is carried in `REDIS_URL`. Redis has no host port. | compose.yaml:10, :65, :67 |
| B8 | api → HIBP (the only outbound call) | App → a fixed third-party host | k-anonymity prefix only, `Add-Padding`, 2 s timeout. It fails open with a metric (ADR-023). | SPEC §4 "memorized-secret verifier"; apps/api/src/lib/hibp.ts |
| B9 | Tenant A ↔ tenant B | Two orgs on the same database | `scoped(actor, …)` forces `orgId`, and `found()` returns 404 for a foreign row (see §4). | apps/api/src/lib/actor.ts:17, :22 |

Only Caddy publishes app ports. Prometheus and Grafana bind to 127.0.0.1. Postgres, pgbouncer, Redis, api and web have no host ports (compose.yaml:1, :173-175, :192, :203).

## 2. Data-flow risks

| Flow | Risk | Mitigation | Residual |
|---|---|---|---|
| Credentials submitted before hydration | A GET request would put the password in the URL, logs and Referer (SEC-1). | Every credential and profile form has `method="post"`. | None |
| Request logging | Signatures, CSRF tokens or emails written to logs (SEC-2). | pino `REDACT_PATHS` covers every SPEC I-2 field. | `*.email` matches one level deep only. No call site logs a deeper shape. |
| SSR → API | All users pooled into one per-IP limiter bucket, which makes a DoS amplifier (SEC-20). | The client IP is forwarded, so the API keys each user separately. | The edge (Caddy) rate limit is advisory only (see §8). |
| Geo telemetry | Precise location is PII. | Coordinates are truncated to 2 dp at ingest, before storage, and bounded by CHECK constraints plus Zod (INV-20). | None |
| CSV export | Formula injection. | Cells starting with `= + - @ \t \r` get a `'` prefix. Filenames are generated server-side. The export is audited before the first byte is sent. | None |
| Source secrets at rest | A DB dump would leak HMAC keys. | AES-256-GCM with the source id bound as AAD. The secret is shown once, at create or rotate. The sources `SELECT` excludes `secretEnc`. | The key is in env (ADR-008); there is no vault in v1. |
| Browser client state | A cached store leaking across users who share a tab. | The store is created per mount inside the (app) layout. Public pages ship no store. | None |
| Post-login redirect | Open redirect. | `safeNext` allows only same-origin paths. | None |

Evidence: apps/api/src/lib/logger.ts:4-18, apps/api/src/lib/csv.ts:5, apps/api/src/lib/crypto.ts:15-16, apps/web/src/lib/server-fetch.ts:8-13.

## 3. Authentication and authorization

| Control | Implementation | Evidence |
|---|---|---|
| Password hashing | Argon2id m=19456 KiB, t=2, p=1, NFKC-normalized, 12–128 characters, never truncated. Unknown emails run a dummy verify. | apps/api/src/lib/password.ts:4, :6 |
| Breach screening | HIBP range API on register, password change and first-login change. Rejected with 422 `PASSWORD_BREACHED`. | SPEC §4 baseline item 1 |
| Session token | 256-bit random. Only its SHA-256 is stored. A fresh token at login prevents fixation. | apps/api/src/services/auth.ts:20 |
| Session lifetime | Idle 30 min, absolute 12 h. A password change revokes every session. The Redis cache TTL of 30 s bounds how long a revoked session lingers. | apps/api/src/services/auth.ts:14-15, :165 |
| Cookie | `ss_session`: `HttpOnly; SameSite=Lax; Path=/`, plus `Secure` when `COOKIE_SECURE` (default true). | apps/api/src/lib/cookies.ts:24-25 |
| CSRF | `HMAC-SHA256(SESSION_SECRET, "csrf.v1.<sessionId>.<seed>")`, recomputed per request and compared in constant time with a length check. A database read alone does not give a valid token. | apps/api/src/services/auth.ts:192-197; apps/api/src/middleware/auth.ts:41-43; test profile-csrf.test.ts:62 |
| Login throttling | A failure limiter on ip+email (5) and on ip (20 per 15 min) runs before the hash compare (INV-11). Register is limited to 10/h/IP. | apps/api/src/routes/auth.ts:39-51, :63 |
| RBAC | Every v1 route mounts `authorize(permission)` (`requireSession` for self-scoped routes). A denial returns 403 and writes an `authz.denied` audit row. | apps/api/src/middleware/auth.ts:57, :62 |
| Two-person rule | `assertNotProposer` returns 403 `SELF_APPROVAL`. unique(proposal, approver) is the backstop. High impact needs 2 approvers. | apps/api/src/services/approvals.ts:25 |
| Action tokens | HMAC-signed, exp ≤ 15 min, claims checked against the proposal. Consumed exactly once by a conditional `updateMany` under an advisory lock, and never leaves the process (SEC-17). | apps/api/src/services/action-tokens.ts:110 |
| Fail-closed config | An unset `NODE_ENV` counts as production. Secrets are required, `RATE_LIMIT_SCALE` must be 1, `HIBP_MODE=off` is rejected and the origin must be https. Dev fallbacks apply only when NODE_ENV is explicitly development or test (SEC-9). | apps/api/src/config/env.ts:24, :57-63, :69-80 |

**Edge cases accepted as residuals:**

| Finding | Edge case | Why it is accepted |
|---|---|---|
| SEC-4 | Register returns 409 "already exists", which reveals whether an email is registered. | Limited to 10/h/IP, and demo self-registration is a product feature. |
| SEC-5 | Nothing limits login attempts per account across many IPs. `/auth/password` has only the 120/min mutation limiter. | The ip+email and ip failure limiters and breach screening remain. |
| SEC-14 | The admin invite flow returns 409 for an existing email. | The flow sits behind `users:write`. |

## 4. Tenant isolation

| Mechanism | Evidence |
|---|---|
| Every service takes `actor`. `scoped(actor, where)` adds `orgId`, and `found()` returns the same 404 for a foreign id and a missing id. | apps/api/src/lib/actor.ts:17, :22 |
| Writes use the tenant from the session. Asset create spreads the input first, so the body can never override `orgId` (SEC-24). | apps/api/src/services/assets.ts:128-129 |
| Adversarial tests: every list, aggregate and export comes back empty of foreign data, and every foreign id returns 404 on reads and mutations with nothing changed. | apps/api/test/integration/tenant-isolation.test.ts ("INV-M2 …" describe blocks) |
| Ingest workers use `batch.orgId`. System jobs (retention, SLA, scheduler) are cross-tenant by design, and their per-org queries filter on `orgId`. | pass-1 SAST review |
| The advisory lock on a foreign proposal id is taken before the scoped lookup. That only briefly locks a hash key, then returns 404 (SEC-18). | apps/api/src/services/execution.ts |

## 5. Ingest signing (HMAC, replay)

| Step (in order) | Behaviour | Evidence |
|---|---|---|
| 1. Per-IP limit | 600/min per IP, as the guard for unauthenticated traffic. | apps/api/src/routes/ingest.ts:23 |
| 2. Raw body | `express.raw`, 1 MB, before any parsing. | apps/api/src/routes/ingest.ts:24 |
| 3. Header shape | The source must be a UUID and the nonce must match `^[A-Za-z0-9_-]{16,64}$`. | apps/api/src/lib/signature.ts:13-14 |
| 4. Source lookup | Only the source row is read before verification (INV-8). An unknown or disabled source looks the same as a bad signature. | apps/api/src/services/ingest.ts:46-50 |
| 5. Signature | `v1=HMAC-SHA256(secret, ts + "." + nonce + "." + rawBody)`, compared in constant time. | apps/api/src/lib/signature.ts:16-25; apps/api/src/lib/crypto.ts:37 |
| 6. Skew | Checked after the signature: \|now − ts\| ≤ `INGEST_MAX_SKEW_SEC` (default 300, bounded 30–3600). | apps/api/src/lib/signature.ts:28-31; apps/api/src/services/ingest.ts:60 |
| 7. Replay | Redis `SET NX EX 600` is the fast path. A unique `(source_id, nonce)` row is the durable backstop, so replay is still blocked when Redis is down (INV-13). | apps/api/src/services/ingest.ts:62-75 |
| 8. Per-source limit | 120 batches/min, keyed on the **verified** source id (SEC-3). An over-limit request still uses up its nonce. | apps/api/src/routes/ingest.ts:15, :28; test ingest.test.ts:161 |
| 9. Idempotency | Events are unique on (org, source, external_id). | SPEC §2 |

The per-source limit runs after the nonce check. That order is deliberate: a captured signed request cannot be replayed within the skew window to spend a source's budget.

## 6. Secrets handling

| Secret | Storage and validation | Notes |
|---|---|---|
| SESSION_SECRET, ACTION_TOKEN_SECRET | env, zod `min(32)`, required in production | Rotating SESSION_SECRET invalidates live CSRF tokens. The client re-fetches the token. |
| SOURCE_SECRET_KEY | env; must decode to exactly 32 bytes | AES-256-GCM key for source secrets |
| POSTGRES_PASSWORD, REDIS_PASSWORD | env; compose `:?` refuses an empty value | Hex only, because the password is embedded in a URL (documented in QUICKSTART and .env.example) |
| Per-source HMAC secret | AES-256-GCM sealed in the DB, source id as AAD | Shown once |
| Logs | pino redact list | SEC-2 test: logger-redact.test.ts "censors signature, CSRF, cookie headers and e-mail fields" |
| Repository | `.env` is git-ignored, and `.env.example` holds only `CHANGE_ME` placeholders. 85 commits contain no committed key or credential file. | Manual secret scan (ADVISORY ONLY) |

Accepted INFO: the Redis password is visible on the `redis-server` argv through `docker inspect` (SEC-23). That is the same exposure as any env secret, and it needs Docker-host access.

## 7. Evidence ledger

| Property | Implementation | Evidence |
|---|---|---|
| Chain | `hash = SHA-256(prevHash ∥ canonicalJSON(envelope))`. The genesis `prev_hash` is 64 zeros. The envelope binds seq, type, actor, subject, payload and createdAt (ADR-044). | apps/api/src/services/ledger.ts:2, :35-38 |
| Ordering | `pg_advisory_xact_lock(hashtext(org_id))` inside the business transaction, taken last (ADR-053), plus unique(org, seq). | apps/api/src/services/ledger.ts:44; test ledger.test.ts F-11.4 |
| Immutability | A `BEFORE UPDATE OR DELETE … FOR EACH ROW` trigger raises. | apps/api/prisma/migrations/20261107000000_init/migration.sql:659-665; test ledger.test.ts F-11.2 |
| Verification | `verifyChain` recomputes each hash and reports the first broken seq. An hourly `ledger-integrity` job runs it. | apps/api/src/services/ledger.ts:155, :185; apps/api/src/services/scheduler.ts:42; test ledger.test.ts F-11.3 |

**What the chain does and does not prove (SEC-6, corrected wording per ADR-060).** The trigger blocks UPDATE and DELETE through the app's normal paths. `verifyChain` catches a row that was changed without its hash and every later hash being recomputed. That covers an app bug, or a trigger-bypassing edit that does not recompute (test F-11.3).

The chain is **not anchored outside the database**. The API and worker connect as the schema owner. An attacker with those credentials can disable or drop the trigger, `TRUNCATE` the table (the trigger does not cover TRUNCATE), rewrite the entries and recompute the whole chain. `verifyChain` would then still pass. **The ledger does not detect that class of change.**

The residual is accepted as LOW. Hardening for v2: run the app as a least-privilege role (INSERT and SELECT only on `evidence_entries`), add a `BEFORE TRUNCATE` trigger, and periodically anchor the head hash outside the DB (ARCHITECTURE §11, "external anchoring is a v2 option").

## 8. Rate limiting

| Limiter | Limit | Key | Evidence |
|---|---|---|---|
| Global | 600/min (`RATE_LIMIT_GLOBAL_PER_MIN`, minimum 10, default 600) | client IP (`req.ip`, 1 trusted hop) | apps/api/src/app.ts:65; apps/api/src/config/env.ts:48 |
| Mutations | 120/min | user | apps/api/src/app.ts:69 |
| Login | 5 failures per ip+email, 20 per 15 min per ip | ip+email, ip | apps/api/src/routes/auth.ts:40-51 |
| Register | 10/h | ip | apps/api/src/routes/auth.ts:39 |
| Ingest | 600/min per IP and 120/min per verified source | ip, source id | apps/api/src/routes/ingest.ts:15, :23, :28 |
| Export | 5/h | user | SPEC §3 |
| Geo | 60/min | user | SPEC §4 has_geo item 2 |
| Scale knob | `RATE_LIMIT_SCALE` 1–1000, dev/test only. Forced to 1 in production, and `Math.max(1, …)` means it can never lower a limit. | n/a | apps/api/src/config/env.ts:53, :63, :80; test rate-limit-scale.test.ts |

The counters live in Redis, so the limits hold across replicas, and a 429 carries `Retry-After`.

**SEC-20 (fixed).** Server-side fetches used to reach the API from the web container's IP, so all users shared one global bucket, and anyone holding a junk cookie could drain it. Now `forwardedClient` forwards only the last X-Forwarded-For entry, which is the one Caddy writes, and only when it looks like an IP. The API trusts exactly one hop. Evidence: apps/web/src/lib/server-fetch.ts:8-13, :27-28; tests in apps/web/test/server-fetch.test.ts ("forwardedClient keeps only the edge-written (last) entry and rejects non-IP values", "without an edge header nothing is forwarded").

**Residual: the edge (Caddy) rate limit is advisory only.** The Caddyfile has no `rate_limit` directive, so only the app-layer limiters above protect the service. A volumetric flood is absorbed by Node before a 429 is returned. This is accepted per ADR-060. Recommendation: add an edge limiter (the Caddy `rate_limit` module or an upstream L4/LB limit) in production.

## 9. CSP and headers

| Surface | Policy | Evidence |
|---|---|---|
| Web CSP | `script-src 'self' 'nonce-…' 'strict-dynamic'`, with `'unsafe-eval'` only when NODE_ENV=development. `style-src 'self' 'nonce-…'`, `object-src 'none'`, `frame-ancestors 'none'`. | apps/web/src/middleware.ts:14-22 |
| Eval-free zod | zod runs jitless in the browser. `@starshield/shared` declares `src/zod.ts` and `dist/zod.js` as its only side-effectful modules, so tree-shaking cannot drop that config (SEC-21, ADR-058). | packages/shared/package.json:6-9; packages/shared/src/zod.ts:8; test apps/web/test/shared-side-effects.test.ts |
| Edge headers | XCTO nosniff, XFO DENY, Referrer-Policy, Permissions-Policy, COOP same-origin, `-Server`, `-X-Powered-By`. HSTS for hosts other than localhost. | deploy/Caddyfile:8-18, :29-30 |
| API headers | helmet CSP `default-src 'none'; frame-ancestors 'none'`, CORP same-origin. HSTS is left to the edge. | apps/api/src/app.ts:47-51 |
| Prefetch | The middleware matcher skips `next-router-prefetch` requests. Those responses are RSC payloads, not documents, and still carry the Caddy headers (SEC-11, accepted). | apps/web/src/middleware.ts:62-65 |
| Runtime check | The web baseline site-headers and cookies checks pass. | FRONTEND-AUDIT.summary.json |

**COOP versus Lighthouse (ADR-059).** COOP is kept even though it makes Lighthouse runs intermittently return `NO_NAVSTART`. The failure is in the measurement, so the header is not weakened to pass a gate.

`experimental.inlineCss` was reverted because its `<style>` tag carries no nonce.

## 10. Infrastructure (compose, Caddy, Redis)

| Component | Posture | Gaps |
|---|---|---|
| Dockerfile | Multi-stage, `USER node`, `NODE_ENV=production`, no secrets baked in | None |
| compose.yaml | Only Caddy publishes HTTP/HTTPS ports. Monitoring binds to 127.0.0.1. `NODE_ENV: production` (:5, :149). `TRUST_PROXY: '1'` (:16). | SEC-6: the app uses the DB owner role (:8-9). SEC-10: `SEED_DEMO` defaults to 1 (:83). |
| Caddy | TLS, HSTS, strips `x-middleware-subrequest` (CVE-2025-29927, INV-18), blocks `/metrics`, JSON access log, admin API on 127.0.0.1:2019 | No edge rate limit (advisory) |
| Redis | `--requirepass ${REDIS_PASSWORD:?}`. Healthcheck uses `REDISCLI_AUTH`. AOF on, `noeviction`. No host port. | SEC-23: password visible on argv (INFO) |
| Postgres / pgbouncer | Internal only. Transaction pooling. | SEC-6 |
| Metrics | `/metrics` has no auth on :4000 and is blocked at Caddy. | SEC-16: exposed if :4000 is ever published directly (INFO) |
| CI | `permissions: contents: read`, no `pull_request_target`, no secrets. Gate: `npm audit --omit=dev --audit-level=high`. | SEC-19: actions are pinned to major tags, not SHAs (INFO) |
| Dependencies | `npm audit --omit=dev`: 0 advisories | SEC-13: 2 critical and 10 high in dev and test tooling only (vitest, lighthouse chain) |

## 11. Scan coverage

| Category | Tooling | Result |
|---|---|---|
| Dependency vulnerability | `npm audit`, `npm audit --omit=dev` | prod 0; dev-only advisories tracked as SEC-13 |
| SAST | ADVISORY ONLY — no external tooling available. Agent self-review is not a substitute for automated scanning. | Manual review. Recommended tools: Semgrep (`p/owasp-top-ten`, `p/nodejs`), CodeQL, `eslint-plugin-security`. |
| Secret detection | ADVISORY ONLY — no external tooling available. | Manual `git grep` and history scan were clean. Recommended tools: gitleaks or trufflehog in CI. |
| Configuration and infrastructure | Manual check against compose, Caddyfile, Dockerfile and CI | SEC-6, 7, 10, 16, 19, 23 |
| Environment-variable audit | Manual check of env.ts against .env.example | SEC-9 |
| Licence compliance | ADVISORY ONLY — manual tally of package-lock | No AGPL, GPL or SSPL. The LGPL libvips exception is documented (SEC-12). Recommended: `license-checker --failOn 'AGPL;GPL;SSPL'`. |
| Web headers and CSP | FRONTEND-AUDIT `site.headers` and `site.cookies` | Pass. SEC-1, SEC-11, SEC-21. |

## 12. Findings

| ID | Severity | Pass | Title | Final status | Evidence |
|---|---|---|---|---|---|
| SEC-1 | MEDIUM | 1 | Credential forms without `method="post"` | FIXED | apps/web/src/app/(auth)/login/LoginForm.tsx:41; (auth)/register/RegisterForm.tsx:52; (app)/settings/SettingsView.tsx:66, :235; (app)/assets/AssetsView.tsx:209 |
| SEC-2 | MEDIUM | 1 | Logger redact list missed the signature, CSRF and email fields | FIXED | apps/api/src/lib/logger.ts:4-18; test logger-redact.test.ts |
| SEC-3 | MEDIUM | 1 | No per-source ingest limit (D-1) | FIXED | apps/api/src/routes/ingest.ts:15, :28; test ingest.test.ts:161 |
| SEC-4 | LOW | 1 | Register 409 reveals that an email exists | ACCEPTED-RESIDUAL | apps/api/src/services/auth.ts:86; ADR-059 |
| SEC-5 | LOW | 1 | No per-account login limiter across IPs | ACCEPTED-RESIDUAL | apps/api/src/routes/auth.ts:40-51; ADR-059 |
| SEC-6 | LOW | 1 | App uses the DB owner role; TRUNCATE not trapped; chain not anchored outside the DB, so an owner-level attacker can disable the trigger and recompute it | ACCEPTED-RESIDUAL | compose.yaml:8-9; migration.sql:664; ADR-060 wording |
| SEC-7 | LOW | 1 | Redis without auth | FIXED | compose.yaml:10, :65, :67 |
| SEC-8 | LOW | 1 | Public health shows version, commit and queue depth | ACCEPTED-RESIDUAL | apps/api/src/routes/health.ts:59-60; ADR-059 |
| SEC-9 | LOW | 1 | Unset NODE_ENV fell back to dev secrets | FIXED | apps/api/src/config/env.ts:24, :57-63, :69-80 |
| SEC-10 | LOW | 1 | `SEED_DEMO=1` default with a shared demo password | ACCEPTED-RESIDUAL | compose.yaml:83; ADR-059 (QUICKSTART: set 0) |
| SEC-11 | LOW | 1 | Prefetch requests skip the middleware headers | ACCEPTED-RESIDUAL | apps/web/src/middleware.ts:62-65; deploy/Caddyfile:8-18 |
| SEC-12 | LOW | 1 | RESEARCH licence claim was inaccurate (LGPL libvips) | FIXED | RESEARCH.md:327 |
| SEC-13 | LOW | 1 | Dev-only dependency advisories (2 critical, 10 high) | ACCEPTED-RESIDUAL | `npm audit --omit=dev` = 0; ADR-059 |
| SEC-14 | INFO | 1 | Admin invite 409 on an existing email | INFO | apps/api/src/services/users.ts; behind `users:write` |
| SEC-15 | INFO | 1 | Public `/api/openapi.json` | INFO | openapi router; describes the API, holds no secrets |
| SEC-16 | INFO | 1 | `/metrics` has no auth (blocked at the edge) | INFO | deploy/Caddyfile:33-34 |
| SEC-17 | INFO | 1 | Action token stays in the server process (conforms to E-2) | INFO | apps/api/src/services/action-tokens.ts:110 |
| SEC-18 | INFO | 1 | Advisory lock taken before the scoped lookup | INFO | apps/api/src/services/execution.ts |
| SEC-19 | INFO | 1 | CI actions pinned to major tags | INFO | .github/workflows/ci.yml |
| SEC-20 | MEDIUM | 2 | SSR fetches pooled every user into one per-IP bucket (DoS) | FIXED | apps/web/src/lib/server-fetch.ts:8-13, :27-28; test apps/web/test/server-fetch.test.ts |
| SEC-21 | LOW | 2 | `sideEffects:false` dropped zod jitless config (CSP regression) | FIXED | packages/shared/package.json:6-9; test apps/web/test/shared-side-effects.test.ts |
| SEC-22 | INFO | 2 | `updateProfile` wrote its audit row outside a transaction | FIXED | apps/api/src/services/auth.ts:244-258 (`$transaction` at :246); test profile-csrf.test.ts:19 |
| SEC-23 | INFO | 2 | Redis password on the argv; hex rule not validated | INFO | compose.yaml:65; ADR-060 |
| SEC-24 | INFO | 2 | Asset create could let the input override `orgId` | FIXED | apps/api/src/services/assets.ts:129; test assets-register.test.ts |
| — | Residual | 2 | No edge (Caddy) rate limit; the app limiters stand | ACCEPTED-RESIDUAL | deploy/Caddyfile (no `rate_limit`); ADR-060 |

No CRITICAL finding was raised in either pass, so the CRITICAL itemization rule did not apply.

## 13. Gate totals

| Metric | Value |
|---|---|
| **Pass 1 findings** | **19** (0 CRITICAL / 0 HIGH / 3 MEDIUM / 10 LOW / 6 INFO). The re-scan raised 5 more (SEC-20 MEDIUM, SEC-21 LOW, SEC-22..24 INFO), so 24 were catalogued in total. |
| **Auto-remediated** | **10**: SEC-1, 2, 3, 7, 9, 12 (ADR-059) and SEC-20, 21, 22, 24 (ADR-060). That includes all 4 MEDIUMs. |
| **Pass 2 residual (CRITICAL / HIGH / MEDIUM open)** | **0 / 0 / 0** |
| **LOW accepted** | **7**: SEC-4, 5, 6, 8, 10, 11, 13. The advisory edge rate limit is also recorded as an accepted residual. |
| **INFO accepted** | **7**: SEC-14, 15, 16, 17, 18, 19, 23 |
