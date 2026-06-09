# SECURITY-AUDIT.md — ServiceHub

Formal adversarial second-pass audit vs SPEC §4 (Security Requirements) + ARCHITECTURE §9 (Invariants). build_depth: comprehensive. Date: 2026-06-08.

## Method
3-pass: (1) initial scan, (2) auto-remediate CRITICAL/HIGH/MEDIUM, (3) re-scan. Scan categories: dependency vulnerability (`npm audit`), SAST (manual review of query construction, auth, crypto, error handling), secret detection (regex sweep of `apps/`), configuration & infrastructure (compose/Caddy/Helmet headers), env-var audit (vs `.env.example`).

## Pass 1 — Findings

| # | Severity | Category | Finding | Clause |
|---|----------|----------|---------|--------|
| F1 | MEDIUM | Dependency | `file-type@16` — ASF parser infinite loop on malformed input (GHSA-5v7r-6r5c-r473). Reachable via the attachment upload path before the allowlist check. | SEC-7 / R3 |
| F2 | MEDIUM | Dependency | `postcss <8.5.10` (XSS via unescaped `</style>` in CSS stringify) bundled transitively under Next.js. | — |
| F3 | LOW | Configuration | Dev secrets present in `docker-compose.yml` (JWT dev values, `servicehub_app` DB password). | SEC-13 |

### SAST review (no findings)
- **SQL injection:** all user-value queries are parameterized (`$1`/`:param` via TypeORM QueryBuilder); raw template literals only interpolate compile-time constants (table names in migrations). The RLS migration interpolates an env-supplied admin password with `'`-escaping. PASS.
- **Crypto:** argon2id for password hashing (INV-13 — no bcryptjs/argon2-browser); HMAC-SHA256 webhook verification with `crypto.timingSafeEqual` (constant time). PASS.
- **Error handling:** single RFC 9457 filter (INV-10); 500s never leak a stack trace (`server.error` generic body). PASS.
- **AuthZ:** object-level chokepoint `assertResourceAccess` (404 to non-owner) on every ticket route + sub-resource; verified at runtime by the smoke test (W3 same-tenant IDOR, sub-resource IDOR, cross-tenant all 404; attachment non-owner download 404). PASS.
- **AuthN anti-enumeration:** login returns identical `auth.invalid_credentials` envelope for unknown-user vs bad-password (smoke `auth.anti_enum` PASS); rate-limit fires before argon2 verify (INV-9). PASS.
- **Headers:** Helmet (CSP frame-ancestors none, X-Content-Type-Options nosniff, X-Frame-Options DENY) + Caddy HSTS confirmed on live responses. PASS.

### Secret detection
Regex sweep of `apps/` for inline credentials → **no matches**. The only literal hash is `DUMMY_HASH` (a deliberate constant for login timing equalization, not a credential). Dev passwords live only in compose/seed (documented dev defaults).

## Pass 2 — Auto-remediation
- **F1 → FIXED.** Removed the `file-type` dependency entirely; replaced with a dependency-free, allowlist-scoped magic-byte sniffer (`apps/api/src/storage/magic-bytes.ts`) for the 6 permitted types. Re-`npm audit`: file-type advisory gone. Covering test: smoke `W1.attach_upload` (image/png detected + accepted), `W1.attach_download` (200), `W1.attach_idor` (404). VERSION patch-bumped 0.1.0 → 0.1.1.
- **F2 → ACCEPTED (with rationale + tracking).** `postcss` is a build-time transitive of Next.js; it processes only trusted, build-authored CSS — ServiceHub never runs postcss over untrusted input, so the `</style>` stringify XSS has no exploit path in this build. No forward fix exists without downgrading Next to v9 (a regression). Tracked: clears when Next bumps its bundled postcss. Logged in COMPLIANCE.md + DECISIONS (ADR-12).
- **F3 → ACCEPTED (dev-only).** Secrets are read only from env (SEC-13); compose values are clearly-marked dev defaults and `.env.example` documents production replacement. Not a production exposure.

## Pass 3 — Re-scan (residual)
- CRITICAL: **0** · HIGH: **0** · MEDIUM: **1** (`postcss`, dispositioned ACCEPTED-with-rationale + tracking) · LOW: **1** (dev secrets, ACCEPTED).
- Full test suite + 27-check functional smoke pass after remediation.

**Gate result: PASS.** Zero CRITICAL/HIGH residual; the single MEDIUM is dispositioned with rationale and a tracking note; no new findings introduced by remediation.
