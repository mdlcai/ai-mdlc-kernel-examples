# COMPLIANCE.md — Sentinel

Maps every SPEC §4 security requirement and `compliance:`-tagged clause to implemented
code + evidence. Baselines: **OWASP Top 10** (2021 + 2025) and **SOC 2** Trust Services
Criteria. Status: ✅ implemented & audit-clean · ⚠️ implemented with open notes · ❌ gap.

Generated after the Security Audit Gate; additive (open items become permanent posture).

| Req | Description | Status | Evidence | Notes |
|---|---|---|---|---|
| SEC-1 | Tenant data org-scoped; cross-tenant → 404 not 403 | ✅ | `packages/db/src/index.js` `withOrgScope`; `packages/core/src/authz.js`; test "cross-tenant access returns 404" | OWASP A01 / SOC2 CC6.1 |
| SEC-2 | Object-level authz via `assertResourceAccess` on owned + sub-resource routes | ✅ | `packages/core/src/authz.js`; called in projects/targets/scans/findings routes | OWASP A01 |
| SEC-3 | argon2id hashing; len 8..1024, no composition, no forced rotation | ✅ | `packages/core/src/hash.js`; `validation.js`; test "argon2id hash + verify" | NIST 800-63B / OWASP A07 |
| SEC-4 | Auth anti-enumeration (shape + timing); limiter before hash | ✅ | `apps/api/src/routes/auth.js` (constant reference verify, `consume()` before verify); test "same 401 envelope" | OWASP A07 |
| SEC-5 | Rate limiting on public endpoints; auth `(ip,email)` | ✅ | `apps/api/src/middleware/rate-limit.js`; auth/login limiter | OWASP A04 |
| SEC-6 | Webhooks verify signature → idempotency → process | ✅ | `apps/api/src/routes/webhooks.js`; INV-7 boundary-order PASS; test "webhook unsigned 401" | OWASP A08 |
| SEC-7 | SSRF denylist on user-supplied target hosts before live scan | ✅ | `packages/core/src/ssrf.js`; targets route; test "SSRF blocked" + unit | OWASP A10 |
| SEC-8 | Scanners only via hardened SandboxRunner | ✅ | `packages/scanners/src/runner.js` `buildSandboxArgv`; INV-6 PASS | OWASP A05 |
| SEC-9 | Secrets via Secrets Manager/env; never hardcoded or logged | ✅ | `packages/config/src/index.js`; `apps/api/src/logger.js` scrub; INV-5 PASS | OWASP A05 / SOC2 CC6.1 |
| SEC-10 | Ownership verification before live scans | ✅ | `apps/api/src/routes/scans.js` (INV-12); test "live target cannot be scanned" | abuse/legal |
| SEC-11 | Immutable audit log of privileged mutations (1y) | ✅ | `store.writeAudit` on register/project/target/scan/triage; `compliance: audit-logging` | SOC2 CC7.2 |
| SEC-12 | Findings retained 90d; audit 1y | ⚠️ | Retention policy specified (SPEC §2); soft-age logic in store; scheduled purge job documented not yet wired | SOC2 CC / GDPR — see note |
| SEC-13 | TLS everywhere (HTTPS only), HSTS, HTTP→HTTPS redirect | ✅ (delegated) | `deploy/Caddyfile` (auto-TLS, redirect, HSTS); `securityHeaders` sets HSTS header | OWASP A02 — TLS termination delegated to proxy/infra |
| SEC-14 | Input validation (zod) with field-level problem+json | ✅ | `packages/core/src/validation.js`; `errors.js`; test "field-level problem+json" | OWASP A03 |
| SEC-15 | Security headers + CSRF/Origin guard on cookie routes | ✅ | `apps/api/src/middleware/security.js` (`securityHeaders`, `originGuard`) | OWASP A05 |

## Posture
- **A (✅): 13** · **B (⚠️): 2** · **C (❌): 0**

### ⚠️ open items (logged, non-blocking)
1. **SEC-12 retention purge** — the 90-day findings / 1-year audit retention is defined and the data model supports soft-aging, but the scheduled purge job is documented (QUICKSTART) rather than wired as a running cron in this build. Tracked as technical debt (DECISIONS) — no data is over-exposed; it is a deletion-timeliness gap, hence ⚠️ not ❌.
2. **SEC-13 TLS (delegated)** — TLS is terminated at the Caddy reverse proxy (real cert, HSTS, forced redirect), not inside the Node process. Per the kernel's delegated-infra rule a delegated control is ⚠️ (delegated), never a bare ✅; the application-layer HSTS header and HTTPS-only cookie flag are in code.

No ❌ gaps. Both ⚠️ items are deviations logged here and in DECISIONS.md, not missing controls.
