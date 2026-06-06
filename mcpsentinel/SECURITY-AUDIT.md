# SECURITY-AUDIT.md — MCP Sentinel (comprehensive build)

Formal adversarial second-pass review vs SPEC §4 (Security Requirements) +
ARCHITECTURE §9/§10. 3-pass procedure (initial scan → auto-remediate → re-scan).

## Pass 1 — initial scan

### Automated — dependency vulnerability scan (`npm audit`)
| # | Severity | Package | Issue | Surface |
|---|----------|---------|-------|---------|
| D1 | HIGH | drizzle-orm <0.45.2 | SQL injection via improperly escaped SQL identifiers (GHSA-gpj5-g38j-94v9) | runtime |
| D2 | MODERATE | drizzle-kit → @esbuild-kit/* | transitive esbuild dev-server request vuln | build-time (unused) |
| D3 | MODERATE×4 | vite / esbuild / vite-node / @vitest/mocker | dev-server request/path-traversal | test/build-time |
| D4 | CRITICAL | vitest <4.1.0 | Vitest UI server arbitrary file read/exec (only with `--ui`) | test-time |

### Automated — secret scan (manual grep; no Gitleaks/TruffleHog available)
- No production secrets committed. Test/fixture files contain intentional dummy
  tokens used as detector inputs (e.g. `sk-ABCD…` in the known-bad fixture);
  these are non-functional and required for the no-false-negative test.

### SAST / config scan — ADVISORY
> ADVISORY ONLY — no external tooling available for SAST (Semgrep/Bandit), secret
> detection (Gitleaks/TruffleHog), or config scanning (Checkov/Trivy). Agent
> self-review is not a substitute for automated scanning. Recommended for CI:
> add Semgrep + Gitleaks + OSV-Scanner (or Trivy) GitHub Actions.

Best-effort manual SAST review (clean):
- **SQL injection:** all queries use the Drizzle parameterized query builder.
  The only raw `client.query(sql)` is the migration runner executing static,
  trusted `.sql` files (the migration name is parameterized). The `sql` template
  in endpoint-service interpolates only Drizzle column refs (identifier-safe),
  no user input. ✓
- **XSS:** React auto-escapes; `dangerouslySetInnerHTML` is forbidden (INV-8,
  lint PASS); CSP `default-src 'self'` set. All MCP-origin strings render as
  escaped text. ✓
- **SSRF (the headline risk, T1):** every user-host egress goes through
  SafeConnector (INV-1/INV-2): scheme allow-list, resolve-all-IPs deny-list, IP
  pinning, redirect re-validation, IPv4-mapped-IPv6 normalization, no npm `ip`.
  Unit matrix + integration `SSRF_BLOCKED` test pass. ✓
- **Auth:** scrypt KDF, httpOnly+Secure+SameSite session cookie, CSRF double-
  submit + Origin check, per-IP rate-limit before hash, HIBP breach screen,
  anti-enumeration (uniform envelopes + timing parity). ✓
- **Secrets at rest / tokens:** MCP auth token request-scoped, never persisted
  (INV-3) or logged (INV-4 redaction); detected secrets masked before write (T6). ✓
- **ReDoS (T5):** detectors use the linear-time matcher on 32 KB-capped input;
  fixture test asserts <200 ms on a 50 K adversarial string (INV-9). ✓

## Pass 2 — auto-remediation (applied)
- **D1 (HIGH):** upgraded `drizzle-orm` → ^0.45.2 (patched). Code unchanged;
  typecheck + 37 tests + build re-run green.
- **D2 (MODERATE):** removed `drizzle-kit` entirely — it was unused (migrations
  are hand-authored SQL run by a custom `migrate.ts`). Eliminates the
  @esbuild-kit chain.
- **D3 (MODERATE):** upgraded `vite` → ^6.3.6 and `vitest` → patched line
  (pulls patched esbuild). 
- **D4 (CRITICAL):** upgraded `vitest` → ^4.1.8 (patched; the vuln required the
  opt-in `--ui` server, which this project never runs).
- VERSION.md patch bumped to 0.1.1.

## Pass 3 — re-scan
- `npm audit`: **found 0 vulnerabilities** (was 1 critical / 1 high / 7 moderate).
- Re-ran the full suite post-remediation: typecheck ✓, eslint ✓, 37/37 tests ✓,
  build ✓, invariant lint 10/10 machine + 4 manual ✓.

## Gate result
- **Zero CRITICAL residuals · Zero HIGH residuals · 0 MEDIUM residual** (all
  remediated, none merely dispositioned). Full suite passing. Pass 3 introduced
  no new findings.
- SAST/secret-tooling degraded verdict: **PASS WITH ADVISORY** (no external SAST
  scanner available; manual review clean; CI tooling recommended above).
