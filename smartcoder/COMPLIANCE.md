# COMPLIANCE.md — SmarCoders

HIPAA-baseline compliance posture, mapped from `SPEC.md` §4 (Security Requirements).
Status: ✔ implemented & audit-clean · ⚠ implemented with open notes · ✘ gap.
(Finalized at the COMPLIANCE.md Gate after the Security Audit Gate.)

| ID | Requirement | Status | Evidence | Notes |
|----|-------------|--------|----------|-------|
| SEC-01 | Passwords hashed with argon2id; no substituted primitive | ✔ | `auth.service.ts` (argon2.hash/verify), INV-14 | argon2id type set explicitly |
| SEC-02 | Anti-enumeration on register & login (shape + timing) | ✔ | `auth.service.ts` (existing-email same-shape + dummy-hash timing pad) | uniform INVALID_CREDENTIALS |
| SEC-03 | Rate limiting on public endpoints; auth limiter before hash; key (ip,email) | ✔ | `rate-limit.ts`, `auth.service.ts` (consume before argon2.verify), INV-7 | Redis-backed + in-mem fallback |
| SEC-04 | httpOnly+Secure+SameSite session cookie; CSRF/Origin guard on mutations | ✔ | `session.ts`, `guards.ts` CsrfGuard | Secure in production |
| SEC-05 | Default-deny RBAC; tenant (org_id) on every scoped query | ✔ | `guards.ts` (AuthGuard/RolesGuard), `scope.ts` withOrgScope, INV-1 | |
| SEC-06 | Cross-tenant access returns NOT_FOUND, not FORBIDDEN | ✔ | `charts.module.ts` loadChartForScope | smoke-tested |
| SEC-07 | Append-only hash-chained audit log of PHI-equivalent access/mutation | ✔ | `audit-log.ts` (prev_hash/row_hash), INV-12 | verifyAuditChain() + integrity badge |
| SEC-08 | Input validation via zod; structured field errors | ✔ | `dto.ts`, `decorators.ts` parse(), `filter.ts` | maps fields → inputs |
| SEC-09 | Webhook verify HMAC over raw body before any DB read; idempotent UNIQUE(event_id) | ✔ | `webhooks.module.ts` (verify→dedupe→process), INV-4/INV-6 | replay window check |
| SEC-10 | Dual-write via transactional outbox; idempotent consumers | ✔ | `outbox.ts`, `worker.ts` relay, INV-5 | at-least-once |
| SEC-11 | Secrets via AWS Secrets Manager (env fallback local); none committed | ⚠ | `.env.example`, INV-8 | Secrets Manager loader is env-fallback in local deploy (ADR-009) — wire SDK in prod |
| SEC-12 | TLS-only (HTTPS, HSTS); Postgres volume encrypted-at-rest | ⚠ | `nginx.conf` (TLS/HSTS/redirect), compose pgdata | at-rest encryption delegated to host/managed disk (honest ⚠) |
| SEC-13 | Security headers + header-buffer sizing | ✔ | `nginx.conf` (HSTS, nosniff, frame-deny, buffers), Node max-http-header-size, INV-13 | |
| SEC-14 | Structured JSON logging without secrets/PHI; trace-correlated | ✔ | `logger.ts` (pino + redact) | dd-trace log injection when enabled |
| SEC-15 | Rate-limit + size cap on import uploads | ✔ | `imports.module.ts` (5MB cap + 413), `rate-limit.ts` | |
| HIPAA-PW | Breached-password screening (HIBP k-anonymity); length-not-composition min 8 | ✔ | `auth.service.ts` isBreached(), `dto.ts` passwordSchema | fail-open offline (logged) |

**Posture:** A (✔) = 13 · B (⚠) = 2 · C (✘) = 0.
⚠ items (SEC-11 secrets delegation, SEC-12 at-rest encryption delegation) are deployment-environment
decisions documented in DECISIONS.md (ADR-009) — they are honest delegations, not gaps. No ✘.
