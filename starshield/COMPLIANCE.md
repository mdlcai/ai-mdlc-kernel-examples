# COMPLIANCE — STARSHIELD v0.1.2

Compliance posture summary (COMPLIANCE.md Gate, BUILD.md). Sources: SPEC.md §4 (RBAC, threat-derived controls, NIST 800-63B verifier baseline, `has_geo` checklist), the `compliance:` clauses, and WEB.md §2/§3/§6. Web-delivery evidence comes from the definitive `FRONTEND-AUDIT.summary.json` (`npm run web:baseline -- --full`, 20 screens × 5 cells = 100 cells). The audit findings referenced below are in SECURITY-AUDIT.md §12.

Status: `✓` means implemented and audit-clean. `⚠` means implemented, but with an accepted LOW residual or a deviation logged as an ADR. `✗` means a gap.

**Posture: 21 ✓ / 7 ⚠ / 0 ✗**

## SPEC §4 — RBAC and threat-derived controls

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| RBAC | The permission matrix in `packages/shared/src/rbac.ts` is enforced by `requirePermission`. A denial returns 403 `FORBIDDEN` and writes `authz.denied`. | ✓ | packages/shared/src/rbac.ts; apps/api/test/integration/audit.test.ts, users.test.ts, export.test.ts; e2e/w8.spec.ts (W8.3), e2e/audit.spec.ts | Hidden UI controls are mirrored by an API 403 (ADR-058). |
| S-1 | Ingest HMAC-SHA256 over `ts.nonce.rawBody`, constant-time compare, skew ≤ 300 s, nonce replay cache, and the source must be enabled. Verification precedes business reads. | ✓ | apps/api/src/services/ingest.ts `verifyIngestSignature`; test ingest.test.ts | INV-8 lint-checked. Per-source limit added (SEC-3, fixed). |
| S-2 | 256-bit session token, stored as SHA-256 only. `ss_session` is HttpOnly, Secure and SameSite=Lax. Rotated on login and password change. Idle 30 min, absolute 12 h. | ⚠ | apps/api/src/middleware/auth.ts, apps/api/src/lib/cookies.ts; tests auth.test.ts, auth-lib.test.ts; summary `site.cookies: pass` | SEC-5 LOW accepted: there is no per-account login limiter across IPs (ADR-059). |
| S-3 | A per-session CSRF token is required as `X-CSRF-Token` on non-GET requests and compared in constant time. | ✓ | apps/api/src/middleware/auth.ts (CSRF check); test profile-csrf.test.ts | — |
| S-4 | CSP nonce + strict-dynamic, HSTS, XCTO, Referrer-Policy, Permissions-Policy, COOP, `frame-ancestors 'none'`. Caddy strips `x-middleware-subrequest`. | ⚠ | apps/web/src/middleware.ts; deploy/Caddyfile; summary `site.headers: pass`, 0 CSP violations across 100 cells | SEC-11 LOW accepted: prefetch requests skip the middleware headers, and Caddy re-applies the static set. INV-18 holds. |
| T-1 | SHA-256 ledger chain with genesis `prev_hash` 64×"0", canonical JSON, per-org advisory lock, unique(org, seq) and an immutability trigger. | ⚠ | apps/api/src/services/ledger.ts; test ledger.test.ts; e2e/w8.spec.ts ("Chain intact.") | SEC-6 LOW accepted: the app uses the DB owner role and the chain is not anchored outside the DB (ADR-060 wording). |
| E-1 | Approver ≠ proposer (403 `SELF_APPROVAL`), distinct approvers, 2 approvers for high impact, approve/reject only in `validated`. | ✓ | apps/api/src/services/approvals.ts; test approvals.test.ts | — |
| E-2 | HMAC action token with exp ≤ 15 min, consumed atomically by a single-row UPDATE (409 `TOKEN_INVALID`), then dispatched. | ✓ | apps/api/src/services/action-tokens.ts; test execution.test.ts | INV-9. SEC-17 INFO: conforms. |
| E-3 | An action on a business_critical asset, or `disable_identity` on a criticality-5 asset, is always high impact. A policy deny fails validation. | ✓ | apps/api/src/services/validator.ts; tests validator.test.ts, policies.test.ts | `minConfidence` policy rule (ADR-059) included. |
| I-1 | Every service is scoped by `actor.orgId`. Cross-tenant `/:id` requests return 404. | ✓ | test tenant-isolation.test.ts (every `/:id` route against a second org) | SEC-24 (input orgId override) fixed. |
| I-2 | The pino redact list covers auth, cookie, CSRF, signature, password, token, secret and email fields. | ✓ | apps/api/src/lib/logger.ts; test logger-redact.test.ts | SEC-2 fixed. |
| D-1 | Redis-backed limits: 600/min/IP global, per-route limits, 120/min per ingest source, 120/min mutations per user. 429 carries `Retry-After`. | ⚠ | apps/api/src/lib/rate-limit.ts; tests rate-limit-scale.test.ts, ingest.test.ts:161; Stage 4 k6 + limit-check | Advisory residual accepted: no edge (Caddy) rate limit (ADR-060). SEC-20 SSR bucket fixed. |
| A-1 | Auth, logout, password, role/active, source, policy, export and authz-denial events are written to `audit_logs`. | ✓ | apps/api/src/services/audit.ts; tests audit.test.ts, export.test.ts; e2e/w8.spec.ts (exports audited, scoped export) | SEC-22 fixed (profile audit inside a transaction). |
| SEC-ops | Production hygiene: demo seed, public health detail, dev-only dependency advisories. | ⚠ | compose.yaml; apps/api/src/routes/health.ts; `npm audit --omit=dev` = 0 | SEC-4, SEC-8, SEC-10 and SEC-13 LOW accepted (ADR-059). QUICKSTART says to set `SEED_DEMO=0`. |

## SPEC §4 — Memorized-secret verifier (NIST 800-63B §5.1.1.2, ADR-007)

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| NIST-1 | HIBP k-anonymity breach screening with padding and a 2 s timeout. A match returns 422 `PASSWORD_BREACHED`. Fails open with a warn log and a metric. | ⚠ | apps/api/src/lib/hibp.ts; test auth-breach.test.ts | Fail-open is the documented availability trade-off (ADR-007). |
| NIST-2 | Length 12–128, no composition rules, no truncation, NFKC normalization. | ✓ | packages/shared password schema; test auth-lib.test.ts | — |
| NIST-3 | The rate limit runs before the hash compare. Unknown emails get a dummy Argon2 verify. Argon2id m=19456, t=2, p=1. | ✓ | apps/api/src/routes/auth.ts (`loginLimiter`); test auth.test.ts | INV-11, INV-47. |
| NIST-4 | No forced rotation and no security questions. Recovery is an admin reset to a temporary password that must be changed. | ✓ | apps/api/src/services/users.ts; test users.test.ts | — |

## SPEC §4 — Domain checklist `has_geo`

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| GEO-1 | Coordinate bounds are enforced by DB CHECK constraints and Zod at ingest, with per-event rejection. | ✓ | migration.sql CHECK constraints; packages/shared ingest schema; test ingest.test.ts | INV-20. |
| GEO-2 | No billable map proxy. `GET /events/geo` is limited to 60/min/user. | ✓ | apps/api/src/routes/events.ts; test events.test.ts | ADR-016. |
| GEO-3 | Coordinates are truncated to 2 decimals before storage. Raw coordinates are never stored. Synthetic data is badged. | ✓ | apps/api/src/jobs/ingest-process.ts; tests ingest.test.ts, synthetic.test.ts | — |

## WEB.md — web delivery contract (definitive FRONTEND-AUDIT.summary.json)

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| WEB-2.1 | Skip link, landmarks, one h1, labelled nav, `html[lang]`. | ✓ | summary `screens.*.failing = 0` (100 cells) | — |
| WEB-2.2 | Two-tier targets: 24×24 everywhere, 44×44 primary controls at 375 px. | ✓ | summary rollup: no target failures | — |
| WEB-2.3 | Measured contrast (axe) in light and dark. | ✓ | summary rollup: no axe/contrast failures | Dark measured at 375 and 1280. |
| WEB-2.4 | Keyboard and `:focus-visible`, reduced motion, form labels. | ✓ | summary rollup: no focus/motion failures; Lighthouse accessibility ≥ target | — |
| WEB-3.1 | Per-route metadata and social cards on public screens. | ✓ | summary `site.meta: pass` (3 public screens) | Authenticated screens are noindex by contract (Lighthouse SEO residual, not scored). |
| WEB-3.2 | Crawl surface (robots, sitemap, icons, manifest) and a designed 404. | ✓ | summary `site.robots`, `site.sitemap`, `site.icon`, `site.notFound: pass` | JSON-LD skipped: not required for archetype `saas`. |
| WEB-4 | Lighthouse performance and best-practices budget on landing and command. | ⚠ | summary `lighthouse[]` (median of 3) | Landing pass (runs 91/0/95). Command accepted per ADR-059: runs 93/0/80, where the 0 is a NO_NAVSTART run under COOP. The clean runs average 86.5, but one scored 80, so performance is marginal against 85. Best-practices is 100 on every clean run. |
| WEB-6 | Security header set, Profile N CSP with zero `securitypolicyviolation`, and session cookie flags. | ✓ | summary `site.headers: pass`, `site.cookies: pass`; 0 CSP violations | — |

## Logged

The posture is logged in DECISIONS.md (ADR-061). The ⚠ items carry into the Pipeline Complete handoff. No ✗ items.
