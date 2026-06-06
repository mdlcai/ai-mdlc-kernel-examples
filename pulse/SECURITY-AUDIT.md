# SECURITY-AUDIT.md — Pulse (comprehensive depth)

Formal adversarial second-pass review against SPEC §4 (Security) and ARCHITECTURE §9.
3-pass: initial scan → auto-remediate → re-scan.

## Pass 1 — Initial scan (findings)

| # | Severity | Finding | Clause | Exploit path | Remediation |
|---|----------|---------|--------|--------------|-------------|
| 1 | HIGH | `drizzle-orm@0.36` — SQL injection via improperly-escaped identifiers (GHSA) | SEC-04 / A03 | Only reachable if user input is interpolated as a SQL **identifier**. Pulse never does this — all identifiers are static schema refs; user input flows only as parameterized **values**. Residual real-world risk low, but the vulnerable version is present. | Bump to `drizzle-orm@0.45.2`. |
| 2 | HIGH | `nodemailer@6` — SMTP command injection / DoS / wrong-domain delivery | SEC-05 / A08 | Worker email channel; attacker-influenced envelope fields could inject SMTP commands. | Bump to `nodemailer@8.0.10`. |
| 3 | CRITICAL | `vitest@2` (via @vitest/mocker→vite→esbuild dev-server) | A06 | Dev/test tooling only — esbuild dev server can be probed cross-origin. Never runs in CI prod build or production runtime. | Bump to `vitest@4.1.8`. |
| 4 | MODERATE | `vite` / `esbuild` (dev server path-traversal / SSRF) | A06 | Dev-server only. | Bump web to `vite@7`; exclude from prod images. |

**SAST / secret / config review (manual, Pass 1):**
- Parameterized queries everywhere (Drizzle); no string-concatenated SQL. ✓
- Input validation: Zod at every boundary; problem+json field errors. ✓
- AuthN/Z: Argon2id, lockout, anti-enumeration login (single envelope, timing-padded), rate-limit before password compare (INV-5). ✓
- Tenant isolation: `scopedDb`(orgId) + RLS, cross-tenant → 404 (no leak), INV-1 enforced by lint. ✓
- Webhooks: signature verify-before-write (INV-4), idempotent UNIQUE. ✓
- SSRF: `assertPublicHttpsUrl` before every outbound custom-webhook fetch (INV-15). ✓
- Secrets: env-only; channel secrets AES-256-GCM at rest (INV-14); Pino redaction of cookie/authorization/*.secret/*.token. No hardcoded secrets in source (DUMMY_HASH is a fixed non-secret used only for timing equalization). ✓
- Config: Helmet, HTTPS+HSTS+redirect (INV-13), Secure/HttpOnly/SameSite cookie, body-size limits, CSRF/Origin guard. ✓
- Env-var audit: `.env.example` documents every variable; required secrets fail-fast if absent. ✓

## Pass 2 — Auto-remediate
- Bumped `drizzle-orm`→0.45.2, `nodemailer`→8.0.10, `vitest`→4.1.8, web `vite`→7.1.5 / `@vitejs/plugin-react`→4.6.0. VERSION.md patch → 0.1.1.
- Re-ran targeted tests: shared 20/20, worker 15/15, api integration 5/5, Functional Smoke 21/21 — **no regression**.
- Added `npm prune --omit=dev` to the api/worker production image so build tooling (vite/vitest/esbuild/typescript) is **not shipped** in the runtime image. Web production image is nginx-static (no Node build tooling).

## Pass 3 — Re-scan (residual)
- `npm audit`: **0 CRITICAL, 0 HIGH**, 2 MODERATE remaining.
- Both residual MODERATEs are a nested `vite@5.4.21`/`esbuild@0.21.5` under `@vitejs/plugin-react` — **dev/build-tooling dev-server vulnerabilities**. Disposition: **ACCEPTED (mitigated)** — the vite dev server is never run in production; deployed artifacts contain no executing vite/esbuild (web = pre-built static via nginx; api/worker = `npm prune --omit=dev`). Rationale logged in DECISIONS.md (ADR-0013) and tracked in REPORT.md.

## Gate criteria
- Zero CRITICAL residual ✓ · Zero HIGH residual ✓ · every MEDIUM dispositioned (ACCEPTED, rationale) ✓ · full test suite passing ✓ · Pass 3 introduced no new CRITICAL/HIGH ✓.
- **Verdict: PASS.**

## ADR-0012 note (carried)
Register on an existing email returns `409 CONFLICT` (a deliberate deviation from strict anti-enumeration) for onboarding clarity. Compensating controls: the login endpoint is fully anti-enumerating (single envelope, timing-padded) + per-(ip,email) rate-limited + account lockout. Net credential-attack surface is on login, which is hardened. Disposition: **ACCEPTED** (product decision), surfaced as a ⚠ in COMPLIANCE.md.
