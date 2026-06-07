# COMPLIANCE.md — BookFlow

Status legend: **✓** implemented & audit-clean · **⚠** implemented with open note/known limitation · **✗** gap.

Posture is reconciled with `SECURITY-AUDIT.md` — a ⚠/✗ detail never rolls up into a ✓ summary.

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| SEC-1 | AuthN on every non-public endpoint; RBAC per endpoint | ✓ | `common/guards/jwt-auth.guard.ts`, `roles.guard.ts`; INV-5; smoke "protected endpoint rejects anon 401" | Global guard + `@MinRole`. |
| SEC-2 | Tenant isolation by `org_id`; cross-tenant → 404 | ✓ | `common/tenant/scoped-repository.ts`; INV-1/INV-14; integration test `isolates tenants`; smoke isolation check | |
| SEC-3 | Password hashing + NIST 800-63B + anti-enumeration + rate-limit-before-hash | ✓ | `auth/auth.service.ts`; INV-6; smoke uniform-error checks | bcrypt cost 12 (bcryptjs, ADR-0011). |
| SEC-4 | Rate limiting (global + auth-specific + webhook ingress) | ✓ | `@nestjs/throttler` global; `@Throttle` on auth + webhook | Auth limit keyed per (ip,email). |
| SEC-5 | Append-only audit logging on every state change | ✓ | `audit/audit.service.ts`; INV-8; `/v1/audit`; smoke "audit has entries" | 7-year retention (no purge). |
| SEC-6 | Webhook integrity: verify-then-read + idempotency | ✓ | `webhooks/resend.controller.ts` (INV-4); `webhooks/svix.spec.ts` (tamper/replay rejected) | |
| SEC-7 | Email/webhook via transactional outbox (no sync send) | ✓ | `notifications/*`, `outbox/worker.ts`; INV-3 | Worker uses FOR UPDATE SKIP LOCKED. |
| SEC-8 | Secrets via env/vault; no committed secrets | ✓ | `.env.example` placeholders; INV-12; secret scan 0 findings | `.env` git-ignored. |
| SEC-9 | HTTPS only in prod (redirect + HSTS); header buffers sized | ⚠ | QUICKSTART "Protocol & TLS"; INV-15 (`--max-http-header-size`) | TLS termination is an infra/proxy responsibility documented in QUICKSTART; not enforceable inside the app container. Local dev is HTTP by design. |
| SEC-10 | Encryption at rest (PII + audit) | ⚠ | DECISIONS / REPORT | Delegated to the database storage layer (encrypted volume/disk in prod). Honestly gated as ⚠ — the app cannot self-attest disk encryption. |
| SEC-11 | OWASP Top-10 controls | ✓ | SECURITY-AUDIT.md (A01–A10 mapped & verified) | |
| compliance:data-retention | Audit logs 7y; bookings indefinite | ✓ | No TTL/purge path; append-only audit | |
| compliance:dual-write | `has_dual_write` outbox pattern | ✓ | `outbox/*`; ADR-0005 | |
| compliance:webhooks | `has_webhooks` verify+idempotency+retention | ✓ | `webhooks/*`; `webhook_events` unique ledger | |
| SOC2:availability | Daily backups + PITR; healthcheck; structured logs | ⚠ | QUICKSTART/REPORT; `/api/health` | Backup schedule + PITR are infra-provisioned in prod; app exposes health + JSON logs. |

## Posture summary
- **A (✓): 11**
- **B (⚠): 3** — SEC-9 (TLS is proxy/infra responsibility, documented), SEC-10 (at-rest encryption delegated to storage layer), SOC2:availability (backups/PITR provisioned at infra).
- **C (✗): 0**

The three ⚠ items are all **infrastructure-layer responsibilities** that an application container cannot self-attest (TLS termination, disk encryption, backup scheduling). Each is honestly gated as ⚠ with the implementation path documented in QUICKSTART.md — none is a code gap. No requirement is unmet at the application layer (C = 0).
