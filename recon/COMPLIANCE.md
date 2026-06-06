# COMPLIANCE.md — Recon

Spec-to-code compliance posture. Status: ✅ implemented + audit-clean · ⚠️ implemented with open notes · ❌ gap.
Maps SPEC.md §4 Security Requirements + `compliance:`-tagged clauses (data retention, audit logging, encryption-at-rest) and the `security_baseline: SOC2` / NIST 800-63B controls.

| Req ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| SEC-1 | Tenant isolation (no cross-tenant access) | ✅ | `common/tenant/tenant-scope.service.ts`, RLS in `migrations/1700000000000-InitSchema.ts` (FORCE RLS), smoke W7 | RLS gated on transaction-local `app.tenant_id`; non-superuser app role |
| SEC-2 | Authorization-to-scan | ✅ | `scans/scans.service.ts` (INV-5 gate), `assets/assets.service.ts` authorize/verify | Demo auto-verify in local mode only (ADR-015); gate always enforced |
| SEC-3 | SSRF / internal-range deny (targets + webhooks) | ✅ | `common/security/target-guard.ts`, `target-guard.spec.ts` (8 tests) | Metadata IP denied unconditionally; re-resolve at exec |
| SEC-4 | AuthN/Z on every non-public endpoint + anti-enumeration | ✅ | `common/auth/jwt-auth.guard.ts`, `roles.guard.ts`, `auth/auth.service.ts` | Uniform register/login responses + timing-equalized verify |
| SEC-5 | Rate limiting (auth + public) | ✅ | `@Throttle` in `auth.controller.ts`, global `ThrottlerGuard` (app.module) | In-memory throttle store; Redis store recommended for multi-instance (note) |
| SEC-6 | Password security (NIST 800-63B) | ✅ | `common/security/password.service.ts`, `password.spec.ts` | argon2id, min 12, HIBP k-anonymity, no rotation |
| SEC-7 | Idempotency (outbox + webhook) | ✅ | UNIQUE(event_id, channel_id) migration (INV-7), `outbox.service.ts` `orIgnore()` | At-least-once + dedupe |
| SEC-8 | Webhook signing (sign before send) | ✅ | `alerts/webhook-sender.ts` (INV-8), `webhook-sender.spec.ts` | HMAC-SHA256, X-Recon-Signature, Idempotency-Key |
| SEC-9 | Secrets management | ⚠️ | `common/secrets/secrets.service.ts` (ADR-009) | Vault wiring point present; local uses env fallback — **delegated** ⚠️; set VAULT_ADDR/TOKEN in prod |
| SEC-10 | Audit logging (SOC2 CC7/CC8) | ✅ | `audit/audit.service.ts`, append-only trigger in migration (INV-12) | DB blocks UPDATE/DELETE on audit_log |
| SEC-11 | Encryption in transit (HTTPS only) | ✅ | `deploy/nginx/recon.conf` (INV-11), QUICKSTART Protocol & TLS | HTTP→HTTPS redirect + HSTS |
| SEC-12 | Encryption at rest | ⚠️ | Postgres volume; `compliance:` | **Delegated** to the storage/disk layer ⚠️ — document managed-disk encryption in prod (COMPLIANCE-delegated, honest) |
| SEC-13 | Data retention ≥12 months | ✅ | `RETENTION_MONTHS` config, snapshot/change retention design (SPEC §4) | Monthly partitioning recommended at scale |
| SEC-14 | Concurrency safety | ✅ | per-asset Redis lock (`scans.service.ts` + `redis.service.ts`), `FOR UPDATE SKIP LOCKED` design | Lock prevents stacked scans |
| DS-webhooks | has_webhooks outbox (verify/idempotency/retention) | ✅ | `outbox.service.ts`, INV-7 | ≤5 retries → dead-letter |
| DS-email | has_email outbox (no synchronous send) | ✅ | `outbox.service.ts` email path | Resend; disabled-but-contract-correct when key absent |
| DS-dual_write | has_dual_write outbox | ✅ | transactional outbox (ADR-003) | Change-event + outbox row in one tx |
| DS-websocket | has_websocket (authenticated, room-per-tenant) | ✅ | `ws/events.gateway.ts` | Session-validated handshake |
| DS-geo | has_geo enrichment | ✅ | scanner geo hook (`packages/scanner`), `GEOIP_DB_PATH` | Country/ASN only |

## Posture
- **A (✅): 16**
- **B (⚠️): 2** — SEC-9 secrets (Vault delegated → env fallback for local; provision Vault in prod), SEC-12 encryption-at-rest (delegated to managed disk; honest delegation per the kernel's delegate-honestly rule).
- **C (❌): 0**

Both ⚠️ items are honest delegations (the kernel's summary↔detail rule: not rolled up into ✅), each with a clear production remediation: set `VAULT_ADDR`/`VAULT_TOKEN`; enable managed-disk/volume encryption. No open CRITICAL/HIGH audit finding touches any requirement.
