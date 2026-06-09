# COMPLIANCE.md — ServiceHub

Maps every SPEC §4 security requirement and `compliance:`-tagged clause to implemented, evidenced code. Status: `✅` implemented + audit-clean · `⚠️` implemented with open notes · `❌` gap. Security baseline: **SOC 2** (Security/Confidentiality/Availability) + **OWASP Top 10**.

## SPEC §4 Security Requirements

| ID | Requirement | Status | Evidence | Notes |
|----|-------------|--------|----------|-------|
| SEC-1 | Tenant isolation (RLS `SET LOCAL` per tx) | ✅ | `common/tenant/tenant-tx.service.ts`, migration `EnableRls`; smoke `W3.cross_tenant` 404 | RLS + app role `servicehub_app` (non-owner) + INV-1/3 |
| SEC-2 | Object-level authz (404 to non-owner) | ✅ | `common/authz/assert-resource-access.ts`; smoke `W3.same_tenant_idor`, `W3.sub_resource_idor`, `W1.attach_idor` all 404 | INV-2 |
| SEC-3 | AuthN on protected endpoints; CSRF/Origin guard | ✅ | global `JwtAuthGuard`; Helmet; CORS allowlist; WS Origin check | smoke unauth → 401 |
| SEC-4 | Auth anti-enumeration (shape + timing) | ✅ | `auth.service.ts` (uniform `auth.invalid_credentials`, dummy-hash timing); rate-limit before hash | smoke `auth.anti_enum` PASS; INV-9 |
| SEC-5 | NIST 800-63B password policy + argon2 + HIBP | ✅ | `auth/password.service.ts` (min 8 / ≤64 / no composition / HIBP k-anonymity / argon2id) | INV-13; HIBP graceful-skip offline |
| SEC-6 | Rate limiting (auth `(ip,email)`, upload cap) | ✅ | `@nestjs/throttler` global; `auth/rate-limit.service.ts` Redis `(ip,email)`; `MAX_UPLOAD_BYTES` | — |
| SEC-7 | Attachment safety (magic-byte, quota, randomized key, gated download) | ✅ | `storage/magic-bytes.ts`, `tickets/attachments.service.ts` (size cap, randomized key, scan_status, ownership-gated `download`) | INV-14; smoke `W1.attach_*` PASS |
| SEC-8 | Webhook verify-before-read, replay, dedup | ✅ | `webhooks/` (HMAC-SHA256 over raw body, `timingSafeEqual`, 5-min staleness, `webhook_events` UNIQUE) | INV-6/8; smoke `webhook.invalid_signature` 401 |
| SEC-9 | Guaranteed delivery via transactional outbox | ✅ | `outbox/outbox.service.ts` (`pending_emails` UNIQUE, ON CONFLICT DO NOTHING, capped backoff → DLQ); `worker.ts` drain | INV-5; never synchronous send |
| SEC-10 | Optimistic locking on ticket edits | ✅ | `tickets.service.ts` `@VersionColumn` + 409 `resource.version_conflict` | — |
| SEC-11 | Append-only audit + server-authoritative SLA | ✅ | `audit/audit.service.ts`, `sla/sla.service.ts`; ValidationPipe `whitelist` strips client SLA fields | ADR-05 |
| SEC-12 | WebSocket JWT handshake + per-room authz | ✅ | `events/events.gateway.ts` (JWT verify, Origin check, tenant-room gate) | R9 |
| SEC-13 | Secrets only from env (validated) | ✅ | `config/env.ts` (zod fail-fast); secret-detection sweep clean | dev defaults in compose only |
| SEC-14 | Security headers (HSTS/CSP/nosniff/frame) | ✅ | Helmet + Caddy; verified on live `/api/health` response | HSTS in production / Caddy |

## Compliance-tagged clauses

| Clause | Status | Evidence | Notes |
|--------|--------|----------|-------|
| `compliance:data_retention` (retain ticket data indefinitely) | ✅ | No hard-delete of tickets/`ticket_events`/`audit_logs`; soft state only | ADR per constraint |
| `compliance:audit_logging` (all mutations + auth events) | ✅ | `AuditService.record` on auth, ticket create/update/status/assign, comment, attachment | append-only |
| `compliance:encryption_at_rest` | ⚠️ | Delegated to infrastructure (managed Postgres + volume encryption) | **Delegated — never claimed as fully implemented in-app.** Document the chosen KMS/volume encryption at deploy time. SOC2 Confidentiality. |
| `compliance:soc2` (Security/Confidentiality/Availability) | ✅ | Controls above map to TSC; daily backups + PITR documented (QUICKSTART); health/metrics for Availability | Availability also via autoscaling + multi-instance |

## Open dependency note (from Security Audit)
- ⚠️ `postcss` (Next.js build-time transitive) — XSS-via-stringify advisory; no untrusted-CSS path in ServiceHub. Tracked until Next bumps bundled postcss (ADR-12).

## Posture
- **A (✅) = 17** · **B (⚠️) = 2** (`encryption_at_rest` delegated; `postcss` tracked) · **C (❌) = 0**
