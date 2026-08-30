# SECURITY-AUDIT.md — CyberOps

**Depth:** comprehensive · **Gate:** Security Audit Gate (3-pass) · **Date:** 2026-08-15
Formal adversarial second-pass review against `SPEC.md` §4 and `ARCHITECTURE.md` §9. The per-feature
security pass and the independent Reviewer Gate ran first; this records what the audit found on top.

## Scan coverage

| Category | Tool | Result |
|----------|------|--------|
| Dependency vulnerability | `npm audit` | 3 HIGH — all in **Next-15-vendored** build-time deps (see below) |
| SAST / insecure patterns | manual review + eslint + invariant-lint | No injection/insecure-pattern findings; parameterized TypeORM queries throughout |
| Secret detection | grep (INV-7 patterns) + `invariant-lint` INV-7 | Clean — no hardcoded secrets in source (`.env` gitignored) |
| Configuration & infra | manual review of docker-compose, main.ts, nginx guidance | helmet on; CORS allowlist; cookies httpOnly+SameSite; `--max-http-header-size` set; SSRF egress guard |
| Environment variable audit | `.env.example` vs code | Every referenced var documented; secrets flagged for Secrets Manager in prod |
| License compliance | manual | Apache-2.0 project; deps MIT/Apache/ISC; AGPL reference repos studied only, never linked (RESEARCH §3.2) |

> Tooling advisory: Semgrep/Gitleaks/Trivy were not available in the build sandbox for automated
> SAST/secret/container scanning. Manual best-effort review + the invariant linter were used; the
> recommendation is to add Semgrep + Gitleaks + Trivy to CI (documented for future runs). Verdict
> degraded to **PASS WITH ADVISORY** on tooling coverage only.

## Three-pass results

- **Pass 1 (initial scan):** 3 HIGH dependency findings (below). 0 CRITICAL. No SAST/secret/config findings.
- **Pass 2 (auto-remediate):** Applied a top-level `overrides: { postcss: ^8.5.26 }` (patches the
  Tailwind/PostCSS pipeline the app itself uses). Attempted to override Next's vendored `sharp`/`postcss`
  — **not overridable** because Next 15 vendors them in its own nested `node_modules` subtree; the only
  npm-level fix is `next@16`, a breaking change deferred by ADR-006.
- **Pass 3 (re-scan):** Residual = the same 3 HIGH, all confined to `node_modules/next/node_modules/*`.

## Findings & disposition

| # | Finding | Base severity | Disposition |
|---|---------|---------------|-------------|
| A | `sharp <0.35.0` libvips CVEs (CVE-2026-33327/33328/35590/35591) via `next` | HIGH | **ACCEPTED (contextual LOW) + tracked.** `sharp` is Next's *optional* image-optimization dep. This app processes **no user-supplied images** (no uploads, no `next/image` remote optimization — icons are glyph/emoji), so the vulnerable code path is unreachable at runtime. Fully remediated by the tracked `next@16` upgrade (ADR-006). |
| B | `postcss <=8.5.22` source-map path-traversal/XSS (GHSA-qx2v/6g55/fxqj/r28c) via `next` (vendored) | HIGH | **ACCEPTED (contextual LOW) + tracked.** Exploit requires processing *attacker-controlled CSS with malicious sourceMappingURL* — this occurs only at **build time** in trusted CI, never on untrusted runtime input. The app's own PostCSS pipeline is pinned to 8.5.26 via override. Remediated by `next@16` (ADR-006). |

**Why not force `next@16` now:** ADR-006 pinned Next 15 for build-environment stability; the Next 16
bump is a documented, tracked follow-up (a one-line dependency change + re-run of this gate). Neither
residual is exploitable in the deployed application's attack surface, so shipping is safe; the operator
is notified in the Pipeline Complete handoff.

## Trust boundaries & data-flow risk assessment (comprehensive)

1. **Browser ↔ API** — JWT in httpOnly/SameSite cookie; ZodValidationPipe at every input; CSRF mitigated
   by SameSite + Origin/CORS allowlist; typed error envelope never leaks stack traces (SPEC §7).
2. **API ↔ Postgres** — all scoped access via `TenantScopedRepository` (INV-1); parameterized queries
   (no string-built SQL); RLS policies as DB backstop; least-privilege role recommended for prod.
3. **Worker ↔ target network** — the highest-risk boundary. `TargetResolver.resolveAndValidate` blocks
   RFC1918/link-local/CGNAT/reserved + IPv6 ULA/link-local and **pins the resolved IP** (`pinnedHttpProbe`
   connects to the IP with Host/SNI, closing DNS-rebinding TOCTOU). Deny-by-default authorization scope is
   re-checked before dispatch (INV-2/INV-3). No live exploitation (ADR-005).
4. **API ↔ LLM / target-derived text** — banners/HTTP bodies/OSINT treated as untrusted data, never as
   instructions (prompt-injection mitigation R7); rule-based planner needs no external AI.
5. **API ↔ third-party webhooks** — HMAC verified over the **raw body** before any DB access (INV-4);
   idempotency UNIQUE(provider,event_id).

## authN / authZ edge cases

- **Anti-enumeration** on register/login/reset: uniform response shape + timing (login always runs a hash
  compare even for unknown users; rate-limit fires before the hash — SEC-4/SEC-5).
- **Object-level authorization (INV-13/SEC-2):** `assertResourceAccess` enforced on owner-bearing
  resources and their sub-resources (assets/pentests/reports/investigations), with admin/owner override;
  same-tenant cross-user access returns NOT_FOUND (no existence leak). Machine-checked by INV-15 + the
  IDOR spec. *(This was the Reviewer Gate FAIL, now remediated — ADR-013.)*
- **Password verifier (NIST 800-63B-4):** scrypt (memory-hard KDF, not bcryptjs — ADR-011); length 8–64,
  no composition rules; HaveIBeenPwned k-anonymity breach screening with offline floor.
- **Audit non-repudiation:** append-only hash chain, advisory-lock serialized, `verifyChain()` verifier.

## Gate verdict

**PASS WITH ADVISORY** — 0 CRITICAL, 0 exploitable-in-context HIGH residual; the 2 base-HIGH dependency
findings are accepted with documented non-exploitability + a tracked `next@16` remediation; SAST/secret
tooling coverage is an advisory (add Semgrep/Gitleaks/Trivy to CI). Full test suite passes after the
override change.
