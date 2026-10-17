# COMPLIANCE.md — STΛRDANCΞR

Spec-to-code traceability for SPEC §4 (Security requirements) and, for the UI, the WEB.md §2 (accessibility), §3 (SEO & metadata) and §6 (headers/CSP) contract rows. Each feature fills in its rows as it lands. The COMPLIANCE.md Gate (after the Security Audit) sets the final status from the audit and from `FRONTEND-AUDIT.summary.json`.

Status: `✓` implemented and audit-clean · `⚠` implemented with a note or ADR · `✗` gap · `…` pending (owning feature not yet built).

Posture: A ✓ 23 / B ⚠ 3 / C ✗ 0 (26 rows: SPEC §4 18, WEB 8) — 0.1.5, FRONTEND-AUDIT `--full`

## SPEC §4 — Security requirements

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| SC-1 | TLS at the edge, HTTP→HTTPS 308, HSTS only on a trusted certificate | ✓ | `deploy/Caddyfile`; `web/src/proxy.ts` (`HSTS_ENABLED`); `artifacts/logs/F-01-edge-probe.log` | Local deploy uses Caddy `tls internal`, so HSTS is off by design (kernel: never on localhost). |
| SC-2 | CSP with a per-request nonce, `strict-dynamic`, no inline script | ✓ | `web/src/proxy.ts` `buildCsp`; `server/src/middleware/security-headers.ts`; INV-14 | |
| SC-3 | `__Host-` session cookie, CSRF double-submit | ✓ | `server/src/middleware/csrf.ts` (Origin + `X-CSRF-Token` safeEqual); `server/src/routes/auth.ts` cookie flags; `web/src/lib/api.ts`; `auth.session.test.ts`; `artifacts/logs/F-03-edge-probe.log` | Cookie `__Host-sd_session; HttpOnly; Secure; SameSite=Lax; Path=/`. |
| SC-4 | `trust proxy` 1; body limits 100 KB / 1 MB ingest / raw webhook | ✓ | `server/src/app.ts` | |
| SC-5 | Redis rate limits (global, login, signup, ingest…) | ✓ | `server/src/middleware/rate-limit.ts`; `platform.middleware.test.ts` | Global limit in F-01. Login (5/15m per account via HMAC key, 20/15m per IP, checked before the hash compare: INV-12), signup 5/h per IP and reset 10/15m per IP are done in F-03 (`server/src/services/auth.ts`). Evidence verify 12/min and export 10/h per org (F-15). Ingest 120/min per key and key issue 20/h per user (F-05); agent API per agent and agent key issue 20/h per user (F-13). Traversal: one per-user `traversal` bucket, 30 points/min weighted by depth, shared by graph, neighbors, impact, attack paths and exposure (F-07, F-11). Demo generate/remove 10/h per org (F-06); compliance evaluate 10/h per org (F-16); prediction runs 12/h per org (F-12); billing mutations 30/10 min per org+user plus the webhook (F-17). Stream opens 30/min per session (F-08); root cause, reconstruction (F-10) and correlations (F-18) charge the `traversal` bucket. 0.1.5: evidence export 5/h/org; ≤ 6 streams per user per replica; entity list scan budget per org (ADR-058) |
| SC-6 | Tenant isolation by `org_id` from session or key; 404 on foreign ids | ✓ | `server/src/middleware/tenant.ts`; `server/src/services/org.ts` (scoped to `ctx.orgId`); `org.test.ts` (smuggled `orgId` ignored); INV-01 | Each later feature adds its own foreign-id 404 test. |
| SC-7 | Prisma only, parameterised; no `$queryRawUnsafe` | ✓ | INV-02, INV-03; `server/src/db/client.ts` | |
| SC-8 | API keys: 32 random bytes, SHA-256 stored, constant-time compare | ✓ | `server/src/lib/apikey.ts`; `server/src/middleware/api-key.ts`, `agent-key.ts` (prefix lookup, dummy-hash compare on miss, one 401 `KEY_INVALID` for every failure); `sources.test.ts`, `agents.test.ts` | Prefixes `sd_in_` / `sd_ag_` (ADR-044). Keys shown once, `Cache-Control: no-store`. |
| SC-9 | Ingest zod validation, depth ≤ 8, ≤ 500 items, backpressure | ✓ | `shared/src/ingest.ts`; `server/src/routes/ingest.ts`; `server/src/services/ingest.ts`; `server/src/jobs/ingest.ts` (5 attempts, DLQ); `ingest.test.ts` | 413 above 1 MB; 429 + Retry-After at `INGEST_MAX_DEPTH`; 503 if the enqueue fails (claimed rows removed). |
| SC-10 | Stripe webhook verify → claim → process | ✓ | `server/src/routes/billing.ts` (`verifyWebhook(` before `claimStripeEvent(`, INV-05); `server/src/services/billing.ts` (claim + handler in one transaction); `server/src/services/stripe-gateway.ts`; `billing.test.ts` | 300 s tolerance; bad/missing/stale signatures 400 with nothing written; replay is a no-op; a handler failure rolls back so Stripe retries. Simulated mode signs Stripe-shaped events with `STRIPE_WEBHOOK_SECRET` through the same receiver (F-17, ADR-049). |
| SC-11 | Governance: agents propose within capabilities; no self-approval | ✓ | `server/src/engine/policy.ts` (`checkCapability`, `checkApprover`, `autoApprovable`); `server/src/services/proposals.ts`; `server/src/services/executor.ts` `assertExecutable`; `policies.test.ts`, `proposals.test.ts` | Out-of-capability agent proposals stored as `policy_rejected` + 403 (ADR-041). Executor re-checks the pinned policy hash and the approver's current role. |
| SC-12 | Append-only hash-chained evidence ledger | ✓ | `server/src/engine/hashchain.ts`; `server/src/services/evidence.ts` (advisory lock, no update/delete path); `evidence_records_append_only` trigger; `server/src/jobs/ledger-verify.ts`; `evidence.test.ts`, `jobs.ledger-verify.test.ts`, `hashchain.unit.test.ts`; INV-10 | Tamper → `broken` at the seq with one critical integrity event; checkpoint HMAC verifies with `EVIDENCE_HMAC_KEY` (ADR-039). |
| SC-13 | argon2id passwords, NIST 800-63B verifier | ✓ | `server/src/services/password.ts` (m=19456, t=2, p=1); `server/src/lib/password-blocklist.ts`; `auth.reset.test.ts`; INV-19 | 12–128 chars, no composition rules, blocklist + context words. |
| SC-14 | Session lifetime, rotation and revocation | ⚠ | `server/src/services/session.ts` (12h idle, 7d absolute, rotation on login, revoke on password change/reset); `auth.session.test.ts` | ADR-038: tokens stored as HMAC-SHA256 with a pepper rather than plain SHA-256. 0.1.5: a revoked session also ends its open event streams (Redis `sessions:revoked`, ADR-057). |
| SC-15 | pino JSON with `requestId`; secret redaction | ✓ | `server/src/lib/logger.ts` `REDACT_PATHS`; `deploy/Caddyfile` URI token filter (F-03) | |
| SC-16 | RFC 9457 errors; no stack traces on 5xx | ✓ | `server/src/middleware/error.ts`; `server/src/lib/problem.ts`; `platform.http.test.ts` | |
| SC-17 | Non-root, read-only containers, tini, no baked secrets | ✓ | `server/Dockerfile`, `web/Dockerfile`, `compose.yaml` hardening anchor; `.gitignore` `.env` | |
| SC-18 | `npm audit --omit=dev` shows no high/critical | ✓ | `npm audit --omit=dev`: 0 (0.1.5); `package.json` `overrides`; SECURITY-AUDIT.md S-1 | Prisma CLI is a dev dependency (migrate stage only). Full tree: dev/load tooling only |

## Web delivery (WEB.md §2 / §3 / §6)

| Contract | Status | Evidence | Notes |
|---|---|---|---|
| Skip link to `#main`, one `<main>`, `html[lang]`, zoom-safe viewport | ✓ | `web/src/app/layout.tsx`, `AppShell.tsx`; FRONTEND-AUDIT 99/99 cells pass (axe, landmarks, viewport) | |
| Nav toggle `aria-expanded` + `aria-controls`; scrollable regions focusable | ✓ | `Topbar.tsx`, `Table.tsx`; FRONTEND-AUDIT cells pass | |
| Error, not-found and loading boundaries with `role=alert` / `aria-busy` | ✓ | `app/error.tsx`, `not-found.tsx`, `loading.tsx`, `States.tsx`; `site.notFound` pass | |
| Primary actions marked `data-primary`, 44 px at 375 px | ✓ | `components/ui/Button.tsx`; FRONTEND-AUDIT target-size checks pass at 375 | |
| robots, sitemap, manifest, icon, OG image | ✓ | `web/src/app/{robots,sitemap,manifest}.ts`, `icon.tsx`, `opengraph-image.tsx`; `site.meta/robots/sitemap/icon` pass | JSON-LD not required for `saas` |
| Security headers on every HTML response | ⚠ | `web/src/proxy.ts`; `site.headers`: CSP (nonce, `strict-dynamic`), frame, referrer, permissions present | No HSTS on `localhost` (ADR-023): it would pin the dev hostname. Set by the edge on a real domain |
| Cookies `__Host-`, Secure, HttpOnly, SameSite | ✓ | `server/src/services/session.ts`, `middleware/auth.ts`; `site.cookies` pass | |
| Lighthouse (mobile): a11y / best-practices / SEO ≥ 90, performance ≥ 80, LCP ≤ 2.5 s, CLS ≤ 0.1 | ⚠ | FRONTEND-AUDIT `lighthouse`: a11y 100, BP 100, SEO 100 (overview noindex by contract); LCP 2.2 s / 1.7 s; CLS ≤ 0.003 | Performance 70 / 70: hydration TBT on a CPU-starved host (ADR-059). Bundle 175 / 213 KB of the 400 KB budget |
