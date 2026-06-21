# SECURITY-AUDIT.md — Tradewind

Formal adversarial second-pass review (BUILD.md Security Audit Gate, comprehensive depth). Audited
against SPEC.md §4 (Security) and ARCHITECTURE.md §9 (Invariants). Date: 2026-06-21. Version: 0.2.1.

## Summary

| Pass | CRITICAL | HIGH | MEDIUM | LOW |
|------|---------:|-----:|-------:|----:|
| Pass 1 (initial scan) | 1 | 1 | 21 | 0 |
| Auto-remediated | 1 | 1 | 19 | 0 |
| Pass 2 (residual) | 0 | 0 | 2 | 0 |

**Gate verdict: PASS** — zero CRITICAL/HIGH residual; both MEDIUM residuals dispositioned as DEFERRED
(build-time-only, in Next's bundled toolchain; fixing requires a breaking Next major downgrade).

## Scan categories run

| Category | Tool | Result |
|----------|------|--------|
| Dependency vulnerability | `npm audit` (api + web) | See findings below |
| Secret detection | grep over `apps/**` for `sk_live_`/`whsec_`/private keys/inline passwords | **Clean** — no hardcoded secrets; all secrets via zod-validated env |
| SAST (manual + pattern) | grep for `eval`/`new Function`/`dangerouslySetInnerHTML`/raw-SQL-concat/`secure:false` | **Clean** — 0 eval, 0 dangerouslySetInnerHTML; only `$queryRaw` tagged-templates (parameterized) for row locks; `$executeRawUnsafe` only in test TRUNCATE with static SQL |
| Configuration | manual review of CORS, cookies, headers, TLS | CORS origin-allow-listed + credentialed; cookies httpOnly+SameSite=Lax+Secure(prod); security headers set; HSTS in prod |
| Environment var audit | `.env.example` vs code `env.*` references | Complete — every referenced var present |

## Findings & remediation

### F-A1 — Transitive CVEs via `@sentry/node@8` (OpenTelemetry core) — 19× MEDIUM — FIXED
`@opentelemetry/core <2.8.0` (unbounded memory in W3C Baggage propagation) pulled transitively by
`@sentry/node@8.55`. **Remediation:** upgraded to `@sentry/node@10.59.0` (the version verified in
RESEARCH §3.1). Re-audit: 0 OTel findings. Tests green.

### F-A2 — `vitest`/`vite`/`esbuild` dev-toolchain CVEs — 1 CRITICAL + 1 HIGH + 1 MEDIUM — FIXED
- CRITICAL: `vitest` `@vitest/mocker` — Vitest UI server arbitrary file read/exec (dev UI only).
- HIGH: `vite` — path traversal in optimized-deps `.map` handling (dev server only).
- MEDIUM: `esbuild` — dev server SSRF (dev server only).
All three are **devDependencies** of the test runner, never present in the production image (which runs
`node dist/index.js`; vitest/vite/esbuild are not invoked at runtime). **Remediation:** upgraded
`vitest@^3` + `npm audit fix` → patched vite/esbuild. Re-audit api: **0 vulnerabilities**. 29 tests
still pass on vitest 3. Exploit path neutralized regardless of dev/prod since the toolchain is patched.

### F-A3 — `postcss <8.5.10` bundled inside Next.js — 2× MEDIUM — DEFERRED (tracked)
PostCSS XSS via unescaped `</style>` in CSS stringify output. The root `postcss` was upgraded to 8.5.15,
but Next 15.5.19 vendors its **own** nested `postcss` in `node_modules/next/node_modules/postcss`. The
only `npm audit fix --force` path installs `next@9.3.3` — a catastrophic breaking downgrade that would
drop the App Router and the entire frontend. **Disposition: DEFERRED.** Rationale: (1) the vulnerability
is in a **build-time** CSS stringify path, not a runtime request path — it cannot be triggered by an end
user of the deployed app; (2) it is internal to Next's toolchain; (3) it resolves automatically when Next
ships a bumped bundled postcss. **Tracking:** re-run `npm audit --workspace apps/web` on each Next minor
upgrade; remove this entry when the nested postcss reaches ≥ 8.5.10.

## Trust-boundary assessment (comprehensive narrative)

- **Browser ↔ API.** Session auth (httpOnly+SameSite=Lax+Secure cookie) + object-level `assertResourceAccess`
  on every owned resource and sub-resource (INV-8). The server is the sole authority on amounts and state
  transitions — client-supplied amounts are never trusted; funding/release amounts derive from the DB
  milestone row. Card data **never** crosses into Tradewind (Stripe hosted Checkout) → PCI SAQ-A held.
  Anti-enumeration: identical register response shape for new/existing email; generic login failure with
  timing parity (dummy scrypt) and per-(ip,email) rate-limit before the verifier.
- **API ↔ Stripe.** All charges/transfers/refunds carry idempotency keys derived from the business id.
  Secret key is env-only, test-mode, redacted from logs. Transfers run **after** the ledger commit (no
  network inside a DB transaction); a transfer failure leaves the payout REQUESTED for retry without
  corrupting the ledger.
- **Stripe webhook ↔ API.** HMAC signature verified **before** any DB read (INV-5, enforced by the
  boundary-order invariant). Replay-safe via UNIQUE `stripe_event_id` (≥30-day retention > 3-day retry
  window) plus business-level idempotency keys. A failed signature/timestamp returns 400 (never 500), so a
  spoofed event cannot trigger infinite Stripe retries.
- **API ↔ DB.** Money mutations run under row locks (`SELECT … FOR UPDATE`) on the affected account, with
  guarded atomic state transitions. The ledger is append-only and immutable; every transaction's legs sum
  to zero (INV-7) and the closed system nets to zero (INV-12), both enforced at posting time and verified
  by the reconciliation job (SEC-16 / SOC2 processing integrity). No money value is ever a float (INV-2).

## AuthN/AuthZ edge cases reviewed

- Sub-resource IDOR (the classic miss): `GET /milestones/:id/ledger` enforces `assertResourceAccess` in
  the service, not just tenant scope — covered by `authz.test.ts`.
- Release is client-only; a freelancer party can read but not approve (404 on the action path).
- Dispute resolution is `requireRole('ADMIN')` only; no unilateral buyer/seller fund movement.
- Cross-owner reads return `NOT_FOUND` (not `FORBIDDEN`) so resource existence does not leak.

## Post-audit state
- Verification suite re-run after remediation: typecheck/lint/test(29)/build/invariant-lint all green.
- VERSION bumped 0.2.0 → 0.2.1 (audit remediation patch).
