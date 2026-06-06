# SECURITY-AUDIT.md — Recon (build 0.1.1)

Comprehensive-depth narrative audit against SPEC.md §4 + ARCHITECTURE.md §9. 3-pass: initial scan → auto-remediate → re-scan.

## Scope & method
- **Dependency vulnerability:** `npm audit` (npm registry advisories).
- **SAST / manual review:** review of auth, tenant isolation (RLS), SSRF guard, outbox/idempotency, secrets, audit logging against the per-feature security checklist.
- **Secret detection:** regex sweep for AWS keys, private keys, Slack/Stripe/GitHub tokens across source.
- **Config/infra:** nginx TLS, docker-compose, Postgres RLS role model.

## Pass 1 — findings
| # | Severity | Area | Finding | Clause |
|---|---|---|---|---|
| 1 | MEDIUM | deps | postcss <8.5.10 XSS-in-CSS-stringify (transitive via Next 15, build-time tooling) | — |
| 2 | MEDIUM | deps | same advisory via Next's nested postcss | — |
| 3 | MEDIUM→FIXED | secrets/SAST | dev-default JWT/webhook secrets could be used if env unset in prod | SEC-9 |

No CRITICAL/HIGH. Secret scan: clean (no committed real secrets; dev fallbacks are env-overridable).

## Pass 2 — auto-remediation
- **#3 FIXED:** added `assertProductionSecrets()` (main.ts) — production boot fails if `JWT_SECRET`/`WEBHOOK_SIGNING_SECRET` are unset or equal the dev defaults.
- **#1/#2:** added root `overrides: { postcss: ^8.5.10 }` → top-level postcss 8.5.15.

## Pass 3 — re-scan / residual
- #3: resolved (production now rejects default secrets).
- #1/#2: residual — Next 15 hard-pins a nested `postcss@8.4.31` for its internal build pipeline. The only upstream remediation is downgrading Next to v9 — a prohibited RESEARCH-deliverable framework downgrade. **Disposition: ACCEPTED-with-rationale + tracked.** Rationale: postcss is a build-time-only dev dependency (absent from the runtime/container image and request path); the advisory requires stringifying attacker-controlled CSS, which Recon never does; clears when Next bumps its bundled postcss. Tracking: re-run `npm audit` on each Next minor bump.

## Verdict
Zero CRITICAL residual · zero HIGH residual · every MEDIUM dispositioned (1 FIXED, 2 ACCEPTED-with-rationale+tracking) · full test suite (29) passing · Pass 3 introduced no new CRITICAL/HIGH. **Gate PASSES.**

## Adversarial review highlights (verified controls)
- **Tenant isolation:** Postgres RLS with FORCE + transaction-local `app.tenant_id`; app connects as a non-superuser role (superusers bypass RLS) — verified by the runtime smoke test W7 (cross-tenant read → 404).
- **SSRF:** `TargetGuard` denies RFC1918/loopback/link-local + metadata IP unconditionally; webhook URLs https-only + re-resolved; 8 unit tests.
- **Authorization-to-scan:** scans hard-gate on `authorized` status (INV-5).
- **Idempotency:** UNIQUE(event_id, channel_id) + `orIgnore()` (INV-7).
- **Webhook signing:** HMAC sign-before-send (INV-8) with unit proof.
- **Password:** argon2id, length-not-composition, HIBP k-anonymity (NIST 800-63B).
- **Audit:** append-only enforced by DB trigger (INV-12).
