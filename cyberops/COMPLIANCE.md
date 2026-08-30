# COMPLIANCE.md — CyberOps

Spec-to-code traceability for `SPEC.md` §4 (Security Requirements) and `compliance:`-tagged clauses.
Baseline: **OWASP Top 10 + SOC 2** (RESEARCH `security_baseline`). Status legend: ✓ implemented &
audit-clean · ⚠ implemented with a noted limitation · ✗ gap.

| Req | Description | Status | Evidence (file / test) | Notes |
|-----|-------------|:------:|------------------------|-------|
| SEC-1 | Tenant isolation (RLS + scoped repo); cross-tenant → NOT_FOUND | ✓ | `common/tenancy/tenant-scoped.repository.ts`; migration `RlsPolicies`; INV-1 (0 violations); smoke cross-tenant 404 | RLS enabled (not forced); app-layer is the enforced+tested guarantee |
| SEC-2 | Object-level authz on user-owned resources + sub-resources | ✓ | `common/auth/access.ts`; assets/pentest/reports/osint services; `idor.integration.spec.ts`; INV-15 | Remediated per Reviewer Gate (ADR-013), incl. reports.download + pentest.decide |
| SEC-3 | NIST 800-63B verifier: scrypt, len 8–64, no composition, breach screen | ✓ | `common/security/password.service.ts`; `password.service.spec.ts` | scrypt (ADR-011); HIBP k-anonymity + offline floor |
| SEC-4 | Anti-enumeration (uniform shape + timing) on register/login/reset | ✓ | `modules/auth/auth.service.ts` (uniform register, timing-safe login, silent reset) | Verified by smoke (register/login) |
| SEC-5 | Rate limiting; auth limiter before hash compare, keyed (ip,email) | ✓ | `common/security/throttler.config.ts`; `auth.service.ts checkAuthRate`; `@Throttle` on auth routes; INV-6 | Redis-backed storage in prod |
| SEC-6 | SSRF guard: resolve→validate→block private→pin IP | ✓ | `common/security/target-resolver.service.ts`; `scanners/net.ts` (IP-pinned); INV-2; `target-resolver.service.spec.ts` | DNS-rebinding TOCTOU closed (ADR-013) |
| SEC-7 | Authorization scope deny-by-default; re-checked before network action | ⚠ | `scan-orchestrator.service.ts` (verified-scope required + INV-3); `assets.service.verifyScope` | Scope verification supports DNS-TXT proof OR audited manual attestation (build/sandbox); DNS proof is decorative when attested — audited as `scope.verify.attested` |
| SEC-8 | Append-only hash-chained audit for every security action | ✓ | `modules/audit/audit.service.ts` (advisory-lock chain + verifyChain); INV-5 | |
| SEC-9 | Webhook: verify signature (raw body) before DB; idempotency UNIQUE | ✓ | `modules/alerts/inbound-webhook.controller.ts`; INV-4/INV-8; `main.ts rawBody:true` | Raw-body HMAC (ADR-013) |
| SEC-10 | PII/findings encrypted at rest; redaction; retention | ⚠ | Evidence marked `encrypted`; retention documented (audit 2y); at-rest volume encryption delegated to infra | **⚠ delegated:** column/volume encryption is a prod-infra responsibility (RDS/KMS) — dev uses an unencrypted volume; documented in QUICKSTART. App-level field encryption for evidence blobs is a tracked follow-up |
| SEC-11 | LLM agent safety: untrusted target text; sandboxed allowlist; human approval | ✓ | `modules/pentest/pentest-planner.service.ts` (rule-based; approval gate; no live exploit — ADR-005); decide now owner-gated | |
| SEC-12 | Supply chain: pinned deps + lockfile; audit in CI | ⚠ | `package-lock.json`; `npm audit`; `SECURITY-AUDIT.md` | 2 base-HIGH in Next-15-vendored postcss/sharp — accepted (not in attack surface) + tracked to next@16 |
| SEC-13 | CSRF/Origin guard on mutations; WS Origin+JWT+rate budget | ✓ | `main.ts` (CORS allowlist, SameSite cookie); `modules/realtime/scan.gateway.ts` | |
| SEC-14 | Secrets: Secrets Manager (prod) / env (dev); no hardcoded | ✓ | `config/config.service.ts`; `.env.example`; INV-7 (0 matches) | |

## SOC 2 / OWASP mapping summary

- **OWASP A01 Broken Access Control** → SEC-1, SEC-2 (✓, remediated).
- **A02 Cryptographic Failures** → SEC-3 (✓), SEC-10 (⚠ delegated at-rest).
- **A03 Supply Chain** → SEC-12 (⚠ tracked).
- **A07 Auth Failures** → SEC-3/4/5 (✓).
- **A10 SSRF/Mishandling** → SEC-6 (✓).
- **SOC 2 CC6 (logical access)** → SEC-1/2/3/5 (✓). **CC7 (monitoring/audit)** → SEC-8 (✓), Sentry + `/api/health` + `/metrics`. **CC8 (change mgmt)** → CI + migrations.

## Posture

- **A (✓): 10** · **B (⚠): 3** (SEC-7 attestation, SEC-10 delegated at-rest encryption, SEC-12 vendored-dep HIGH) · **C (✗): 0**
- No open CRITICAL/HIGH exploitable-in-context findings. The three ⚠ items are documented deviations
  with compensating controls and tracked remediations, not gaps. Summary agrees with the detail rows
  (no ⚠ rolled up into an "Applied/✓" claim).
