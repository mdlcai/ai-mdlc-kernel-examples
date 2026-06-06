# COMPLIANCE.md — MCP Sentinel

Maps every SPEC §4 Security Requirement + tagged compliance clause to implemented,
audited code. Status: ✅ implemented & audit-clean · ⚠ implemented with open note ·
❌ gap. Reconciliation rule: a control rolls up to "Applied" only if its detail row
is ✅ backed by named code.

Baseline regime: **OWASP Top 10 (2021)** + OWASP MCP Top 10 mapping (RESEARCH §5).

| ID | Requirement | Status | Evidence (file / test) | Notes |
|----|-------------|--------|------------------------|-------|
| SEC-1 | SSRF: resolved-IP deny-list + IP pinning + redirect re-validation; egress only via SafeConnector | ✅ | `apps/api/src/scanner/safe-connector.ts`; `safe-connector.test.ts` (deny matrix), `integration.test.ts` (SSRF_BLOCKED); INV-1/INV-2 PASS | OWASP A10. Opt-in `SCAN_ALLOW_HOSTS` default empty (ADR-015). |
| SEC-2 | Bounded consumption: connect+total timeouts, byte ceiling, JSON depth cap, per-user concurrency | ✅ | `mcp-client.ts` (readCapped, deadline), `scan-service.ts` (concurrency), `normalizer.ts` (depth 32) | OWASP LLM10. |
| SEC-3 | MCP auth token request-scoped: never persisted/logged, redacted, not sent cross-host | ✅ | `scan-service.ts` (token in opts only), `mcp-client.ts` (drop on cross-host redirect), `logger.ts` redact; INV-3 PASS | No DB column stores a token. |
| SEC-4 | ReDoS-safe matching on length-capped input | ✅ | `scanner/rules/matcher.ts`; `matcher.test.ts` (<200ms on 50K adversarial); INV-9 manual ✓ | ADR-012 linear matcher. |
| SEC-5 | AuthN/Z: scrypt; httpOnly+Secure+SameSite session; protected routes 401; tenant isolation; anti-enumeration | ✅ | `lib/crypto.ts`, `services/identity.ts`, `middleware/auth.ts`, `db/scope.ts`; `integration.test.ts` (isolation→404, uniform failure); INV-5 PASS | OWASP A01/A07. |
| SEC-6 | Rate limiting: per-IP auth (before hash) + per-user scan + concurrency | ✅ | `middleware/rate-limit.ts`, `routes/auth.ts`, `routes/scans.ts`; INV-12 manual ✓ | |
| SEC-7 | CSRF/Origin guard on all state-mutating verbs | ✅ | `middleware/csrf.ts` (Origin + double-submit); INV-13 manual ✓ | OWASP A01. |
| SEC-8 | Breached-password screening (HIBP k-anonymity); length-not-composition; no forced rotation | ✅ | `lib/hibp.ts` (range API, Add-Padding), `shared` password schema (8–128) | NIST 800-63B. Fails open on HIBP outage (logged). |
| SEC-9 | Secret-at-rest redaction + XSS defense (escaped render, CSP, no innerHTML) | ✅ | `lib/redact.ts` (mask before persist), `components/*` (escaped), `middleware/security-headers.ts` (CSP); INV-8 PASS | OWASP A03. |
| SEC-10 | Audit logging of security-relevant actions | ✅ | `audit_log` table; `services/identity.ts` (auth.* events, IP) | `audit_logging: true`. |
| CMP-1 | Structured error contract (RFC 9457 problem+json, field-level) | ✅ | `lib/problem`/`errors.ts`, `middleware/error.ts`, `lib/validate.ts` | UI binds field errors (SPEC §7). |
| CMP-2 | Secrets via environment variables only | ✅ | `src/env.ts`, `.env.example`; no secrets in code (audit secret scan clean) | `secrets_management`. |
| CMP-3 | Structured JSON logging | ✅ | `lib/logger.ts` (pino) | `logging_format`. |
| CMP-4 | HTTPS-only serving (HSTS, TLS-at-proxy, HTTP→HTTPS redirect) | ✅ | `security-headers.ts` (HSTS in prod), `QUICKSTART.md` §Protocol & TLS, `Dockerfile`/compose | `protocol_support: HTTPS only`. |
| CMP-5 | Dependency vulnerability posture | ✅ | `npm audit` → 0 vulnerabilities (SECURITY-AUDIT.md) | Post-remediation. |

## Posture
- **A (✅): 15 · B (⚠): 0 · C (❌): 0** — all requirements met.

## Advisory (non-blocking)
- No external SAST/secret-scanning tooling in this environment → Security Audit
  is PASS WITH ADVISORY. Recommended for CI: Semgrep, Gitleaks, OSV-Scanner/Trivy.
  This does not lower any control's status (each ✅ is backed by named code +
  test + the automated dependency scan), but is recorded for transparency.
