# SECURITY-AUDIT.md — BookFlow

Formal Security Audit Gate (3-pass) for build 0.1.0. Depth: comprehensive.

## Method
- **Pass 1 — initial scan:** dependency vulnerabilities (`npm audit`), secret detection (grep for live keys / private keys / inline credentials), SAST-style review of auth/tenant/webhook/rate-limit boundaries, config & env review, plus the OWASP Top-10 checklist against SPEC §4.
- **Pass 2 — auto-remediate:** applied fixes without prompting; bumped VERSION patch; re-ran tests.
- **Pass 3 — re-scan:** repeated Pass 1 against the remediated tree.

## Pass 1 findings

| # | Severity | Finding | Clause | Exploit path |
|---|---|---|---|---|
| F-1 | HIGH | `next@14.2.x` carries multiple advisories (RSC DoS, cache poisoning, SSRF on WS upgrade, middleware bypass) fixed only in the 16.x line. | SEC-9 / OWASP A06 | Remote attacker triggers DoS / cache poisoning against the web tier. |
| F-2 | MEDIUM | Transitive `postcss < 8.5.10` (XSS via unescaped `</style>` in CSS stringify). | OWASP A06 | Build-time CSS tooling only; not reachable by app runtime. |

Secret detection: **0 findings** (no live keys, private keys, or inline credentials in source; `.env` git-ignored; `.env.example` placeholders only).
SAST / boundary review: **0 findings** — verified controls below.
Config/env review: **0 findings** — secrets via env/vault; CORS pinned; helmet enabled; header buffers sized (INV-15).

## Pass 2 — auto-remediation
- **F-1 → FIXED:** upgraded `next` 14.2.x → **16.2.7** and `react`/`react-dom` → 19 (aligns with RESEARCH §3.1, which records Next 16.2.x as current stable). Web typecheck + production build pass; all 16 routes render. VERSION bumped to 0.1.1.
- **F-2 → PARTIALLY FIXED / DEFERRED:** added root `overrides.postcss: ^8.5.10` (our direct tree now resolves postcss 8.5.15). Next.js **vendors its own bundled `postcss@8.4.31`** that npm overrides cannot rewrite; the only npm-offered fix is an unstable `next` canary. Residual is **build-time only** (CSS stringify), **not runtime-exploitable** by BookFlow (we never run untrusted CSS through postcss). **Disposition: DEFERRED**, tracked here; clears when a stable Next release ships a patched bundled postcss.

## Pass 3 — re-scan (residual)
- CRITICAL: **0** · HIGH: **0** · MEDIUM: **1** (F-2, deferred build-time-only with rationale + tracking) · LOW: 0.
- Full test suite: **10/10 pass**. No new CRITICAL/HIGH introduced.

## Verified controls (OWASP Top-10 mapped)
- **A01 Broken Access Control** — global `JwtAuthGuard` (INV-5), `RolesGuard` (`@MinRole`), row-level org scoping via `ScopedRepository` (INV-1); cross-tenant → NOT_FOUND (INV-14). *Verified live:* org B sees 0 of org A's data; member → 403 on approvals; self-approval → 403.
- **A02 Crypto** — bcrypt (cost 12) for passwords; JWT access/refresh with rotation; refresh tokens stored as SHA-256 hashes. (bcryptjs is the same algorithm — ADR-0011.)
- **A03 Injection** — parameterized TypeORM queries throughout; class-validator DTOs; no string-built SQL.
- **A04 Insecure Design** — no-double-booking is a DB `EXCLUDE` constraint, not an app check (race-proof); transactional outbox for dual-write.
- **A05 Misconfig** — helmet; CORS pinned to `CORS_ORIGIN`; header buffers sized (INV-15); secrets via env/vault.
- **A07 Auth Failures** — anti-enumeration (uniform register + login envelopes, timing-padded), rate-limit **before** bcrypt compare (INV-6), NIST 800-63B password policy. *Verified live.*
- **A08 Integrity** — inbound webhooks verify Svix HMAC on the raw body **before** any read (INV-4) + idempotency ledger; *unit-tested* (tamper/replay/missing-header rejected).
- **A09 Logging** — append-only audit log on every mutation (INV-8); structured JSON logs with secret redaction.
- **A10 SSRF** — no user-controlled outbound URLs in v1; outbound webhook targets are server-configured.

## Gate result
Zero CRITICAL, zero HIGH residual; the single MEDIUM is dispositioned (DEFERRED, build-time-only, tracked); full suite green; Pass-3 introduced no new CRITICAL/HIGH. **Gate: PASS.**
