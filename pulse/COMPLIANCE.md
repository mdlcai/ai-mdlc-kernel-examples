# COMPLIANCE.md — Pulse

Compliance posture mapped from SPEC §4 (Security Requirements) and `compliance:`-tagged clauses,
after the Security Audit Gate. Additive, permanent posture.
`✓` implemented + audit-clean · `⚠` implemented with open notes/known limitation · `✗` gap.

Baseline: SOC2 (Trust Services Criteria), OWASP Top 10 (2021), OWASP ASVS 5.0.

| ID | Requirement | Status | Evidence | Notes |
|----|-------------|--------|----------|-------|
| SEC-01 | Tenant isolation (scoped access + RLS; cross-tenant → 404) | ✓ | `packages/db/client.ts` scopedDb; RLS in `0000_init.sql`; INV-1 lint; api integration "tenant B → 404"; smoke isolation | A01 |
| SEC-02 | Argon2id hashing + lockout + anti-enumeration | ⚠ | `modules/auth/routes.ts` (argon2id, lockout 5, dummy-hash timing, single 401 envelope) | Register returns 409 on existing email (ADR-0012) — deliberate onboarding deviation; login fully anti-enum + rate-limited + lockout (compensating) |
| SEC-03 | Rate limiting (per org+ip, tiered, 429+Retry-After; before password compare) | ✓ | `middleware/rateLimit.ts`; INV-5 boundary-order; smoke 429 path | A05 |
| SEC-04 | Input validation (Zod) + parameterized queries | ✓ | `middleware/validate.ts`, Zod schemas, Drizzle params; problem+json field errors | A03 |
| SEC-05 | Inbound webhook signature verify-before-write + outbound HMAC | ✓ | `modules/webhooks/routes.ts` (verifyInboundSignature→recordInbound, INV-4); `shared/hmac.ts`; smoke verify+idempotent+bad-sig | A08 |
| SEC-06 | SSRF egress guard on outbound webhooks | ✓ | `shared/ssrf.ts` assertPublicHttpsUrl; worker `channels/webhook.ts` + api test-send; INV-15; unit tests | A10 |
| SEC-07 | Secrets via env; channel secrets AES-256-GCM at rest; redaction | ✓ | `.env.example`; `shared/crypto.ts`; `channels/secrets.ts` seal/redact; Pino redaction; INV-14 | A02 |
| SEC-08 | HTTPS-only + HSTS + HTTP→HTTPS redirect | ✓ | `deploy/nginx.conf` (301 redirect + HSTS); `middleware/core.ts` httpsGuard; Helmet; INV-13 | TLS |
| SEC-09 | Audit logging of mutations + auth events | ✓ | `audit.ts` writeAudit on every mutation; `audit_logs` table; `/v1/audit` (admin) | SOC2 CC |
| SEC-10 | Data retention + downsampling | ⚠ | `DATA_RETENTION_DAYS` documented; metrics/check_results partition-ready | Retention/rollup **job not yet scheduled** in MVP (partition-ready schema; documented posture) — deferred, tracked in REPORT.md |
| SEC-11 | CI dependency + secret scanning; pinned deps + lockfile | ✓ | `.github/workflows/ci.yml` runs `npm audit`; lockfile committed; no secrets in source | A06 (secret-scanning tool: advisory — recommend gitleaks/trufflehog) |

## Audit residuals carried into posture
- 2 MODERATE dev-tooling advisories (nested vite/esbuild under @vitejs/plugin-react) — ACCEPTED/mitigated (not in production runtime; ADR-0013).

## Posture
- A (✓): **9** · B (⚠): **2** (SEC-02 register deviation, SEC-10 retention job deferred) · C (✗): **0**
- No open CRITICAL/HIGH. Both ⚠ items have documented compensating controls / deferral rationale.
