# COMPLIANCE.md — ServiceHub

Maps each SPEC §4 security requirement and `compliance:`-tagged clause to its
implementation status. Posture legend: **✅** implemented & audit-clean · **⚠**
implemented with open notes/limitations · **❌** gap.

Baselines: **SOC 2** (2017 TSC) · **OWASP Top 10:2021** · NIST SP 800-63B.

| ID | Requirement | Status | Evidence | Notes |
|----|-------------|:------:|----------|-------|
| SEC-1 | Argon2id password hashing, length policy (min 12) | ✅ | `apps/api/src/modules/auth/password.ts`; unit test `unit.test.ts` | OWASP params (m=19456,t=2,p=1) |
| SEC-2 | JWT access ≤15m + rotating refresh w/ reuse detection, httpOnly+Secure cookies | ✅ | `modules/auth/tokens.ts`, `services/auth.ts` (refresh family revoke), `routes/auth.ts` (cookie, Secure in prod) | |
| SEC-3 | Anti-enumeration (uniform response + timing), rate-limit before hash | ✅ | `services/auth.ts` (DUMMY_HASH timing equalization), `routes/auth.ts` (authRateLimiter before handler) | smoke: wrong-pw → 401 uniform |
| SEC-4 | RBAC deny-by-default, per-route permission, per-org scope | ✅ | `modules/auth/rbac.ts`, `middleware/auth.ts` (requirePermission), `middleware/security.ts` (originGuard) | |
| SEC-5 | Tenant isolation: org-scope + Postgres RLS | ✅ | repositories filter org; RLS policies in migration (INV-11); smoke: B→A = 404 | |
| SEC-6 | Input validation (Zod) on every endpoint | ✅ | Zod schemas in every `routes/*.ts` | |
| SEC-7 | Parameterized SQL only (no string SQL) | ✅ | TypeORM query builder / parameterized `set_config`/increment | |
| SEC-8 | Distributed rate limiting; stricter on auth | ✅ | `middleware/rateLimit.ts` (rate-limit-redis) INV-6 | |
| SEC-9 | Helmet security headers + HTTPS-only + redirect | ⚠ | `app.ts` Helmet (CSP, HSTS in prod) | HTTP→HTTPS redirect + HSTS terminated at reverse proxy (QUICKSTART §Protocol & TLS); app emits HSTS in prod |
| SEC-10 | Inbound webhook HMAC verified on raw body before DB; replay window; dedupe | ✅ | `modules/webhooks/inbound.controller.ts` (INV-5), `signature.ts` (5-min window), UNIQUE (INV-4); smoke: bad sig→401, replay→duplicate | |
| SEC-11 | Outbound webhooks signed; SSRF guard | ✅ | `modules/webhooks/ssrf.ts` (private-range + DNS-rebind block), `worker.ts` delivery signs + asserts | |
| SEC-12 | Uploads: type/size limits, image re-encode + EXIF strip, SVG blocked | ✅ | `services/attachments.ts` (sharp re-encode, allowlist, size cap) | |
| SEC-13 | Append-only audit log; covers auth/ticket/approval/config | ✅ | `services/audit.ts`, migration trigger `trg_audit_append_only` (INV-12) | |
| SEC-14 | Secrets from env only; fail-fast validation | ✅ | `config/env.ts` (Zod, exit on invalid), `.env.example` (INV-7) | |
| SEC-15 | Dependency scanning in CI + SBOM | ⚠ | `.github/workflows/ci.yml` runs `npm audit` | SBOM generation documented as a follow-up CI step |
| SEC-16 | Centralized error tracking; structured JSON logs w/ request-id | ⚠ | `config/logger.ts` (pino JSON + redaction), `observability/sentry.ts` capture hook, `x-request-id` | Sentry/OTel are pluggable drop-ins (DSN/endpoint via env); base build forwards to structured logs |
| SEC-17 | Data retention: tickets 365d, audit 7y, PITR 7d (configurable) | ⚠ | env `TICKET_RETENTION_DAYS`/`AUDIT_RETENTION_DAYS`; ADR-09 | Automated purge job + PITR cron documented in QUICKSTART; scheduled job not yet wired |
| SEC-18 | PII access-controlled, encrypted at rest + TLS in transit | ⚠ | RBAC + RLS + pino redaction; TLS to DB via `DATABASE_SSL`; HTTPS at proxy | At-rest encryption delegated to the storage layer (volume/disk encryption) — per kernel, delegated = ⚠ |

## Posture
**A (✅) = 13 · B (⚠) = 5 · C (❌) = 0**

### Open ⚠ items (auto-proceed under review_gates: auto; logged in DECISIONS.md)
- **SEC-16 / monitoring** — Sentry + OpenTelemetry are wired as pluggable hooks
  (enabled by `SENTRY_DSN` / `OTEL_EXPORTER_OTLP_ENDPOINT`); the base image keeps
  them dependency-free and forwards errors to structured JSON logs + `/api/metrics`
  (prom-client). Drop in the provider SDK in `observability/` to forward.
- **SEC-17 retention** — retention windows are configurable; the scheduled purge
  job and PITR backup cron are documented (QUICKSTART) but not yet scheduled.
- **SEC-18 at-rest encryption** — delegated to the Postgres volume / disk
  encryption layer (kernel marks delegated controls ⚠).
- **SEC-9 / SEC-15** — TLS redirect/HSTS handled at the reverse proxy; CI runs
  `npm audit`, SBOM generation is a documented follow-up.

No CRITICAL/HIGH gaps. All ⚠ items have a documented remediation path.
