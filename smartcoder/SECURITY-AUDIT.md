# SECURITY-AUDIT.md — SmartCoder (build 0.1.3)

Comprehensive-depth narrative audit against SPEC.md §4 (Security Requirements) + ARCHITECTURE.md §9 (Architectural Invariants). 3-pass: initial scan → auto-remediate → re-scan. Every control claimed in SPEC §4 / ARCH §9 / REPORT.md was grepped against the actual source and cited `file:line`; claimed-but-inert controls are flagged as findings, not rubber-stamped. Reconciled with the inline Security Audit Gate already recorded in REPORT.md (which fixed CRITICAL AUTH-1 + MEDIUM CFG-1 at 0.1.1 — both re-verified present here).

## Scope & method
- **Dependency vulnerability:** `npm audit` (npm registry advisories) at workspace root (covers all 3 workspaces; single lockfile).
- **SAST / manual review:** read auth (`auth.service.ts`, `auth.controller.ts`), RBAC + CSRF (`common/guards.ts`), session (`common/session.ts`), tenant isolation (`server/scope.ts`, `charts.module.ts` `loadChartForScope`), input validation (`decorators.ts` `parse`, shared `dto.ts` zod), injection (raw `pg` parameterization in `server/db/pool.ts` + every handler), rate limiting (`common/rate-limit.ts`), webhook verify-first/idempotency (`webhooks.module.ts`, shared `verifySignature`), audit logging (`server/audit-log.ts`), secrets boot guard (`main.ts`).
- **Secret detection:** regex sweep of `apps/`, `packages/`, `scripts/`, `infra/`, `db/` (excluding `node_modules`/`dist`/`.next`) for AWS keys, PEM private keys, Slack/Stripe/GitHub/Google tokens, JWT/DB passwords.
- **Config/infra:** `docker-compose.yml`, `infra/nginx.conf` (TLS/HSTS/headers/CSP), Postgres role model, env-var matrix vs `.env.example`.
- **Env-var audit:** code-referenced secrets vs `.env.example`; dev-default usability in prod.
- **HIPAA/PHI-specific:** at-rest encryption posture, PHI (chart JSONB / MRN-equivalent) handling, PHI-access audit logging, password verifier policy (NIST 800-63B).

## Pass 1 — findings
| # | Severity | Area | Finding | Clause |
|---|---|---|---|---|
| 1 | MEDIUM→ACCEPTED | deps | `postcss <8.5.10` XSS-in-CSS-stringify (GHSA-qx2v-qp2m-jg93), 2 moderate, transitive inside Next 15.5.19's bundled build toolchain | DEP-1 |
| 2 | MEDIUM→FIXED | config / SEC-13 | **Content-Security-Policy header absent** from `nginx.conf` (only HSTS/nosniff/frame-deny/referrer set) although SEC-13 enumerates CSP and COMPLIANCE.md marked SEC-13 ✔ — overclaimed control | SEC-13 |
| 3 | MEDIUM→FIXED | SAST / SEC-15 / ARCH §2 | Import upload (`imports.module.ts`) had a 5 MB size cap but **no rate limit**, while SEC-15 requires "rate-limit **and** size cap"; ARCHITECTURE §2 named a `RateLimitGuard` cross-cutting guard that **does not exist** in source (claimed-but-inert) | SEC-15 |
| 4 | LOW→DEFERRED | SAST / SEC-04 | `CsrfGuard` only rejects when an `Origin` header is present and mismatched; requests with **no Origin** pass. Mitigated by `SameSite=lax` cookie + the JWT being httpOnly | SEC-04 |
| 5 | INFO (hardened) | SEC-07 / INV-12 | Audit-log append-only was enforced only by app convention (no `UPDATE`/`DELETE` in source — verified) + the invariant-lint grep; **no DB-level guard**. Not an overclaim (COMPLIANCE cited `verifyAuditChain()`, not a trigger) but hardened proactively given HIPAA load-bearing status | SEC-07 |

**No CRITICAL/HIGH.** (Prior CRITICAL AUTH-1 register-takeover was remediated at 0.1.1 and re-verified active here — see verified controls.) **Secret scan: clean** — no AWS keys, PEM keys, or vendor tokens in source; dev fallbacks in `docker-compose.yml`/`session.ts`/`webhooks.module.ts` are env-overridable and blocked in prod by `assertProdSecrets()` (`main.ts:14`). **Injection: clean** — every query in `server/**` and handlers is parameterized (`$1..$n`); no string interpolation of user input into SQL. **PHI at-rest:** chart payload (PHI-equivalent) stored as plaintext `JSONB`; encryption-at-rest is honestly delegated to the managed/host disk (SEC-12 ⚠ in COMPLIANCE, `docker-compose.yml:11`) — consistent, not a hidden gap, and data is Synthea-synthetic.

## Pass 2 — auto-remediation
- **#2 FIXED (SEC-13 CSP):** added `Content-Security-Policy` to `infra/nginx.conf` — `default-src 'self'; base-uri 'self'; object-src 'none'; frame-ancestors 'none'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'; connect-src 'self' https://api.pwnedpasswords.com; form-action 'self'`. `frame-ancestors 'none'` backstops `X-Frame-Options`; `connect-src` whitelists the HIBP breach-screen endpoint actually called from the app path. Per the build constraint, **no nonce/`strict-dynamic`** was introduced (it breaks Next.js standalone hydration); `'unsafe-inline'` on style/script is retained and logged as a residual.
- **#3 FIXED (SEC-15 rate-limit):** `imports.module.ts` now injects `RateLimiter` and calls `consume('import:<orgId>', IMPORT_RATE_LIMIT_MAX ?? 20, RATE_LIMIT_WINDOW_MS)` before accepting an upload; module providers updated. Corrected ARCHITECTURE §2 to describe the real service-boundary `RateLimiter` (auth + import) instead of the non-existent `RateLimitGuard`/`AuditInterceptor`.
- **#5 HARDENED (SEC-07 DB enforcement):** new forward-only migration `db/migrations/003_audit_log_append_only.sql` installs `BEFORE UPDATE`/`BEFORE DELETE` triggers on `audit_log` that `RAISE EXCEPTION ... ERRCODE 'insufficient_privilege'`, making tamper-evidence enforced by Postgres independent of the app layer.
- COMPLIANCE.md SEC-07 / SEC-13 / SEC-15 evidence rows updated to reflect the now-accurate posture.
- **VERSION.md** bumped `0.1.2-functional-verified → 0.1.3`.

## Pass 3 — re-scan / residual
- **API typecheck:** `tsc --noEmit` clean after edits. **Shared unit tests:** pass (0 fail). No new CRITICAL/HIGH introduced.
- **#1 DEP-1 — postcss <8.5.10 — ACCEPTED-with-rationale + tracked.** `npm audit` still reports 2 moderate; the only upstream fix is `npm audit fix --force` → `next@9.3.3`, a prohibited framework downgrade. postcss here is a **build-time-only dev-toolchain dependency** (Next's bundled CSS pipeline) — absent from the runtime container and the request path; the advisory requires stringifying attacker-controlled CSS, which SmartCoder never does. Matches the recon disposition and the build's build-time-CVE triage rule. **Tracking:** re-run `npm audit` on each Next minor bump.
- **#2 CSP `'unsafe-inline'` — residual, ACCEPTED-with-rationale.** style/script-src retain `'unsafe-inline'` because Next.js 15 standalone emits inline runtime/bootstrap + inline styles; a nonce/`strict-dynamic` policy has broken standalone hydration in prior builds and cannot be browser-verified in this offline audit. The added policy still removes the framing/object/base-uri/form-action attack surface and is strictly better than the prior absence. **Tracking:** revisit when Next exposes a stable nonce hook for standalone output.
- **#4 CsrfGuard missing-Origin — DEFERRED (LOW).** Cookie is `SameSite=lax` + httpOnly, so cross-site form/script CSRF cannot attach the session; the residual is only non-browser clients omitting Origin. Hardening to a strict allowlist could break the documented server-to-server/curl smoke flows. Tracked for a future double-submit-token pass.
- **#3 / #5 — FIXED** (rate-limit wired; audit-log append-only DB-enforced). **AUTH-1 / CFG-1** (0.1.1) re-verified FIXED.

## Verdict
Zero CRITICAL residual · zero HIGH residual · every MEDIUM dispositioned (MED-2 FIXED, MED-3 FIXED, DEP-1 ACCEPTED-with-rationale+tracking) · 1 LOW deferred with rationale · API typecheck clean · shared tests passing · Pass 3 introduced no new CRITICAL/HIGH. **Gate PASSES.**

## Adversarial review highlights (verified controls)
- **Password hash (SEC-01 / INV-14):** `argon2.hash(..., { type: argon2.argon2id })` and `argon2.verify` — `auth.service.ts:72,97`; no substituted primitive (bcryptjs/argon2-browser) present.
- **Password policy (HIPAA-PW, NIST 800-63B):** length-not-composition `z.string().min(8).max(200)` — `packages/shared/src/dto.ts:15`; **breach-screen (SHALL)** via HIBP k-anonymity `isBreached()` — `auth.service.ts:29-42`; no composition rules (correctly **not** added).
- **Anti-enumeration (SEC-02):** existing-email register returns `{created:false}` and the controller emits a shape-identical 201 with **no session** — `auth.service.ts:60-70`, `auth.controller.ts:29-31` (AUTH-1 fix verified active); login uses a precomputed `DUMMY_HASH` so timing is uniform for unknown emails — `auth.service.ts:18,95`.
- **Rate-limit-before-hash (SEC-03 / INV-7):** `rateLimiter.consume('auth:login:<ip>:<email>', ...)` precedes any `argon2.verify` — `auth.service.ts:90` before `:97`.
- **Session cookie (SEC-04):** `httpOnly:true, secure: NODE_ENV==='production', sameSite:'lax'`, 12h JWT (jose HS256) — `common/session.ts:24-30`.
- **Default-deny RBAC + tenant scope (SEC-05):** `AuthGuard` requires a session for any non-`@Public()` route and enforces `@Roles()` — `common/guards.ts:41-49`; every scoped query funnels org_id through `withOrgScope` — `server/scope.ts:14-23`.
- **Cross-tenant → NOT_FOUND (SEC-06):** `loadChartForScope` returns `NOT_FOUND` (never `FORBIDDEN`) for other-org / non-assignee charts — `charts.module.ts:17-19` (smoke-tested cross-org → 404 per REPORT).
- **Append-only hash-chained audit log (SEC-07 / INV-12):** `prev_hash`+`row_hash` SHA-256 chain, INSERT-only, `verifyAuditChain()` — `server/audit-log.ts:15-20,29-37,53-64`; grep confirms **zero** `UPDATE`/`DELETE` on `audit_log` in source; **now also DB-enforced** by trigger — `db/migrations/003_audit_log_append_only.sql`.
- **Input validation (SEC-08):** every handler body/query runs zod via `parse()` → structured `VALIDATION_ERROR` with field map — `common/decorators.ts:21-27`, schemas in `packages/shared/src/dto.ts`.
- **Webhook verify-first + idempotency (SEC-09 / INV-4/6):** HMAC over **raw body** verified before any DB read; `verifySignature` uses `timingSafeEqual` — `webhooks.module.ts:20-25`, `packages/shared/src/index.ts:12-18`; dedupe via `processed_events` UNIQUE(event_id), duplicate → 200 no-op — `webhooks.module.ts:38-49`, `db/migrations/001_init.sql:143`; raw body retained at `main.ts:35`.
- **SQL injection:** all queries parameterized through `server/db/pool.ts` (`$1..$n`); no user input string-concatenated into SQL anywhere in handlers.
- **Production-secret assertion (SEC-11):** `assertProdSecrets()` aborts boot in production if `JWT_SECRET` is weak/<32 chars or `WEBHOOK_SIGNING_SECRET` is a dev default — `main.ts:14-23` (CFG-1 fix verified active).
- **Security headers + buffers (SEC-13 / INV-13):** HSTS, CSP (added), nosniff, frame-deny, referrer-policy — `infra/nginx.conf:31-35`; nginx `large_client_header_buffers 8 32k` + Node `--max-http-header-size=32768` — `nginx.conf:6-7`, `docker-compose.yml:30`.
- **Structured logging without secrets/PHI (SEC-14):** pino with `redact` of `password`, `password_hash`, `cookie`, `authorization`, `*.payload` — `common/logger.ts`.
- **Import DoS controls (SEC-15):** 5 MB size cap → `PAYLOAD_TOO_LARGE` and per-org rate limit (added) — `imports.module.ts`.
