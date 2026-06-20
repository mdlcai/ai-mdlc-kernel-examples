# SECURITY-AUDIT.md — Sentinel (comprehensive depth)

Formal adversarial second-pass review against SPEC §4 (Security Requirements) and
ARCHITECTURE §9. Three-pass: initial scan → auto-remediate → re-scan.

## Trust boundaries reviewed
1. **Untrusted scanner engine ↔ host kernel** — scanners run user-supplied code/targets.
   Mitigated by the hardened SandboxRunner (`--network none --read-only --cap-drop ALL
   --no-new-privileges --pids-limit --memory --cpus --rm`, gVisor `runsc` when available);
   demo engine is network-free in-process. SEC-8/INV-6.
2. **User target URL ↔ egress (SSRF)** — `checkTargetHost` denylist (RFC1918, loopback,
   link-local incl. 169.254.169.254 metadata, CGNAT) before any live scan. SEC-7/INV-11.
3. **Proof-of-ownership** — live (domain/ip) scans gated on `ownershipVerified`. SEC-10/INV-12.
4. **Multi-tenant isolation** — `withOrgScope` + `assertResourceAccess`; cross-tenant → 404.
   SEC-1/2, INV-1/2.
5. **Credential secrecy** — secrets via config/Secrets Manager; logger scrubs secret keys;
   Sentry `beforeSend` scrub documented. SEC-9/INV-5.
6. **Public API surface** — signed idempotent webhooks (verify→idempotency→process), CSRF
   origin guard, rate limiting, security headers, RFC 9457 errors. SEC-5/6/14/15, INV-7/8.

## Pass 1 — Initial scan

| # | Category | Finding | Severity | Clause | Exploit path | Remediation |
|---|---|---|---|---|---|---|
| A1 | Dependency (SCA) | `postcss <8.5.10` XSS via unescaped `</style>` in CSS Stringify (GHSA-qx2v-qp2m-jg93), pulled transitively under `next` and `@tailwindcss/postcss` | MEDIUM | SEC (deps) | An attacker who can feed untrusted CSS through PostCSS's stringify could inject markup. | Pin `postcss ^8.5.10` via root `overrides`. |
| A2 | SAST (manual review) | Login timing: ensure argon2 verify runs even for unknown email | — (verified OK) | SEC-4 | Timing oracle to enumerate accounts | Already implemented: constant reference-hash verify path. |
| A3 | Secret detection | Hardcoded secrets / keys in source | — (none found) | SEC-9 | Credential leak | INV-5 secret-pattern scan PASS over 87 files. |
| A4 | Config | Session cookie flags, HSTS, CSP, header buffer | — (verified OK) | SEC-13/15 | Session theft / header DoS | `Secure`+`httpOnly`+`sameSite=lax`; HSTS; CSP; `--max-http-header-size=32768`. |
| A5 | Env audit | `.env` committed / secrets in repo | — (none) | SEC-9 | Secret leak | Only `.env.example` with `REPLACE_ME` placeholders; `.env` gitignored. |

## Pass 2 — Auto-remediate
- **A1:** Added `"overrides": { "postcss": "^8.5.10" }` to the root `package.json` and
  reinstalled. The application's own top-level PostCSS (used by `postcss.config.mjs` +
  `@tailwindcss/postcss`) is now **8.5.15** (patched). VERSION.md patch bumped.
- A2–A5 required no change (verified clean in Pass 1).

## Pass 3 — Re-scan (residual)
- **A1 residual (MEDIUM, ACCEPTED — not reachable):** `next@16.2.9` **bundles** its own
  copy of `postcss@8.4.31` inside its published tarball; npm `overrides` cannot replace a
  bundled dependency, and downgrading Next (the only `audit fix --force` path) is an
  unacceptable breaking change. **Exploitability assessment:** this advisory is an XSS in
  PostCSS's *stringify of untrusted CSS*. Next's bundled PostCSS runs only at **build
  time**, transforming our own first-party Tailwind/CSS; no untrusted input flows through
  it, and PostCSS does not execute at runtime. The finding is therefore **not reachable**
  in Sentinel's threat model. Disposition: **ACCEPTED**, tracked to upgrade when Next ships
  a release bundling PostCSS ≥ 8.5.10 (ADR-012). The reachable copy is patched.
- No new CRITICAL/HIGH/MEDIUM introduced by the remediation. Full test suite (21) green.

## Gate result
- CRITICAL residual: **0** · HIGH residual: **0** · MEDIUM residual: **1 (ACCEPTED, not
  reachable, tracked)**. Full test suite passing; Pass 3 introduced no new findings.
- Tooling note: dependency scan via `npm audit`; SAST/secrets/config via structured manual
  adversarial review against SPEC §4 (the product's own Semgrep/Trivy/gitleaks engines run
  against *user* targets, not self-applied in this gate) — `PASS WITH ADVISORY` on the
  self-SAST category (no Semgrep run against our own tree in this environment).
