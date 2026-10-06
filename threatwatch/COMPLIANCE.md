# COMPLIANCE.md — ThreatWatch

One row per SPEC §4 requirement plus the WEB.md delivery rows. Status: ✓ implemented and verified · ⚠ partial or delegated · ✗ missing.
Evaluated at: `5fc6924` (after RB-7, the last code change; RB-8 `114e621` adds only gate evidence). Baseline floor: OWASP Top 10, no regulatory regime (SPEC §4.4).

## Threat-derived controls (SPEC §4.1)

| # | Requirement | Status | Implementation | Evidence |
|---|---|---|---|---|
| S1 | Tenant isolation, foreign id → 404 | ✓ | `findFirst({id, orgId})` in every service; malformed ids → 404 (`api/middleware/validate.ts` parseParams) | org-B cases in every `server/test/integration/<domain>/*.test.ts`; smoke check 9; INV-13 |
| S2 | argon2id, `__Host-` session, idle/absolute expiry, rotation, revocation | ✓ | `services/auth/password.ts`, `services/auth/session.ts` | `test/integration/auth/auth.test.ts`, `test/unit/auth/password.test.ts`; INV-41 |
| S3 | CSRF Origin check on non-GET → 403 `CSRF_ORIGIN` | ✓ | `requireSameOrigin` | auth tests; smoke check 4; INV-42 |
| S4 | Rate limits (global, login, register, ingest, search, ws) with `Retry-After` | ✓ | `api/middleware/rate-limit.ts`, `api/ws/hub.ts` | auth.test.ts (register 6th → 429, login 6th → 429), search.test.ts, ws.test.ts (1008) |
| S5 | Ingest: HMAC verify first, identical 401 | ✓ | `api/routes/ingest.ts`, `lib/signing.ts` | INV-16; ingest.test.ts (tamper, skew, unknown, disabled); smoke check 4 |
| S6 | SSRF guard at create and before every dispatch | ✓ | `lib/ssrf.ts` `assertSafeUrl` | INV-10, INV-15; `test/unit/lib/ssrf.test.ts` |
| S7 | Nonce CSP, no `dangerouslySetInnerHTML`, headers, `frame-ancestors 'none'` | ✓ | `web/src/proxy.ts`, `web/next.config.ts`, helmet | INV-18; FRONTEND-AUDIT headers section |
| S8 | Secrets at rest AES-256-GCM | ✓ | `lib/crypto.ts` | `test/unit/lib/crypto.test.ts` (round-trip, tamper, nonce uniqueness) |
| S9 | Logging redaction, generic 5xx | ✓ | `lib/logger.ts`, `lib/problem.ts` | `test/unit/lib/logger.test.ts`, `test/unit/lib/problem.test.ts` |
| S10 | Audit log for every listed action | ✓ | `audit_logs` writes in each service | audit assertions across integration suites; `admin/org-audit-jobs.test.ts` |
| S11 | zod on every body/query/param; body limits | ✓ | `api/middleware/validate.ts`; `api/app.ts` limits (100 KB / 256 KB ingest / 1 MB import) | per-route 400 tests; malformed-cursor 400 on every list (RB-1) |
| S12 | Assignee is an active user of the same org | ✓ | `services/alerts/actions.ts` | alerts.test.ts |

## RBAC (SPEC §4.2)

| # | Requirement | Status | Implementation | Evidence |
|---|---|---|---|---|
| RBAC | Permission matrix owner/admin/analyst/viewer; admin cannot touch owners; last-owner guard | ✓ | `api/authz/permissions.ts` | role cases in every integration suite; `admin/team.test.ts` (`LAST_OWNER`); e2e viewer specs |

## Domain-signal checklists (SPEC §4.3)

| # | Requirement | Status | Implementation | Evidence |
|---|---|---|---|---|
| WH-1 | Signature first, DB second | ✓ | ingest route reads only `secret_enc` by PK before verify | INV-16; ingest.test.ts |
| WH-2 | Idempotent receipts + per-event dedupe | ✓ | `webhook_receipts` UNIQUE, events UNIQUE | INV-12, INV-20; smoke check 6 (replay → `duplicate:true`) |
| WH-3 | Receipts retained 7 d | ✓ | `services/jobs/retention-sweep.ts` (`RECEIPT_TTL_MS`) | retention-sweep.test.ts |
| WH-4 | Anomaly flag on failed detection | ✓ | `services/detection/batch.ts`, integration health | detection.test.ts, integrations.test.ts; INV-37, INV-38 |
| GEO-1 | `NUMERIC(9,6)` coordinates with range CHECKs | ✓ | `prisma/schema.prisma` | INV-8 |
| GEO-2 | Map provider proxy | ✓ N/A | local SVG map (ADR-010), global limit | ADR-010 |
| GEO-3 | Only attacker city centroids, no asset locations | ✓ | `domain/enrich.ts` | enrich.test.ts |
| WS-1 | Origin checked before upgrade | ✓ | `api/ws/hub.ts` | INV-17; ws.test.ts |
| WS-2 | Inbound budget 20/10 s, 4 KB frames, close 1008 | ✓ | `api/ws/hub.ts` (`maxPayload`) | ws.test.ts |
| WS-3 | Org channel only, `{kind,id}` payloads, revoked sessions closed; forced password change refused (RB-1) | ✓ | `api/ws/hub.ts` | ws.test.ts |
| SW-1 | Effect assertion per trigger T1–T7 | ✓ | `server/test/jobs/*.test.ts` | INV-11 (manual verdict in REPORT Reviewer Gate) |
| SW-2 | Every trigger exercised; exhaustive `JOBS` map | ✓ | `worker/jobs.ts` `satisfies Record<JobName,…>` | 7 job test files |
| SW-3 | Per-tenant `runXForOrg` used by tests | ✓ | each job module | job tests via `recordJobRun(… runXForOrg)` (RB-1) |
| SW-4 | Idempotent claim, run twice → one effect | ✓ | conditional updates / `SKIP LOCKED` | job tests (second run processed 0) |
| SW-5 | Injected time, UTC schedules | ✓ | `now` parameter everywhere; `job_runs.finishedAt` on the injected clock (RB-1) | job tests |
| SW-6 | `job_runs` counts; Admin › Jobs highlights failures | ✓ | `services/jobs/runs.ts`, JobsView | job tests assert the persisted row; e2e admin jobs |
| SW-7 | catchup 10 m, overlap SKIP, graceful shutdown | ✓ | `worker/schedules.ts`, `worker/main.ts` | Stage 4 shutdown check |
| SW-8 | Smoke forces detection-sweep + email-drain | ✓ | `POST /api/v1/admin/jobs/:job/run` | smoke check 7 (`smoke-test.log`) |
| DUAL | DB → Temporal outbox with sweep reconcile | ✓ | `ingest_batches` pending + `detect-<batchId>` + T1 | detection.test.ts, detection-sweep.test.ts; INV-35–37 |
| EM-1 | `pending_emails` columns + index | ✓ | `prisma/schema.prisma` | migration |
| EM-2 | Inserted in the triggering transaction (incl. admin reset temp password when enabled, RB-1) | ✓ | `services/notify/enqueue.ts`, team service | notifications.test.ts, team.test.ts |
| EM-3 | Drain LIMIT 50 SKIP LOCKED via transport | ✓ | `services/email/drain.ts` | email-drain.test.ts; INV-29, INV-30 |
| EM-4 | Backoff 1 m/10 m/1 h → dlq; manual retry | ✓ | `services/email/drain.ts`, Notifications › Log | email-drain.test.ts; INV-31 |
| EM-5 | No synchronous sends | ✓ | transport only | INV-14 |
| EM-6 | UNIQUE `(to_address, template_key, business_ref)` | ✓ | schema | INV-23 |
| WS-OUT-1 | `pending_deliveries` columns + index | ✓ | schema (ADR-011) | migration |
| WS-OUT-2 | Signature at INSERT, re-sent verbatim | ✓ | `services/notify/enqueue.ts` | `test/unit/notify/signing.test.ts`, delivery-drain.test.ts |
| WS-OUT-3 | `Idempotency-Key` + webhook headers | ✓ | `services/deliveries/drain.ts` | delivery-drain.test.ts |
| WS-OUT-4 | Backoff 1 m…6 h, dlq after the 6th failure (ADR-027) | ✓ | `services/deliveries/drain.ts` | delivery-drain.test.ts; INV-34 |
| WS-OUT-5 | SKIP LOCKED, `assertSafeUrl`, `redirect:'error'`, 5 s timeout; disabled endpoints skipped (RB-1) | ✓ | `services/deliveries/drain.ts` | delivery-drain.test.ts; INV-10 |
| WS-OUT-6 | Receiver guidance documented | ✓ | QUICKSTART "Outbound webhooks: receiver guidance" | QUICKSTART "Outbound webhooks: receiver guidance" |
| PAY | `has_payments` | ✓ N/A | ADR-006 | INV-8 |

## Compliance baseline (SPEC §4.4)

| # | Requirement | Status | Implementation | Evidence |
|---|---|---|---|---|
| PW-1 | Breached-password screening (register, change, reset) | ✓ | `services/auth/password.ts` | password.test.ts, auth.test.ts (`PASSWORD_BREACHED`) |
| PW-2 | Length 12–128, no composition rules, no truncation | ✓ | `passwordPolicyError` | password.test.ts |
| PW-3 | Login limiter before argon2 verify | ✓ | `api/routes/auth.ts` | auth.test.ts |
| PW-4 | No rotation or security questions; temp-password recovery | ✓ | team service `must_change_password` | team.test.ts |
| ENC | Volume encryption at rest | ⚠ delegated | column-level AES-256-GCM covers secrets (S8); volume encryption belongs to the deploy target | SPEC §4.4 |

## Web delivery (WEB.md §2 / §3 / §6)

Source: `FRONTEND-AUDIT.summary.json` (full report `FRONTEND-AUDIT.json`) from the definitive `--full` run at `5fc6924`, 2026-10-05 15:47–16:02Z.

| # | Requirement | Status | Evidence |
|---|---|---|---|
| WEB §2 | Accessibility, contrast, target size, overflow, CLS in light + dark at 375/768/1280 | ✓ | 29 screens × 375/768/1280 × light/dark = 145 cells: axe (0 serious/critical), contrast, target size, overflow, keyboard, focus, skip link and CWV (CLS ≤ 0.036) all pass. Loading skeletons carry `role="status"` (RB-7) |
| WEB §3 | Security headers, CSP nonce, HSTS posture (ADR-025), cookies | ✓ | Site `headers` and `cookies` pass: per-request CSP nonce with `strict-dynamic`, `X-Frame-Options: DENY`, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `__Host-tw_session` (Secure, HttpOnly, SameSite). Local HSTS `max-age=0` by ADR-025; production sets a real policy at its trusted origin |
| WEB §6 | Lighthouse performance / a11y / best-practices / SEO budgets | ✓ | landing perf 94 / a11y 100 / BP 100 / SEO 100. dashboard perf 81 (floor 80) / a11y 100 / BP 100; SEO not scored (authenticated, `noindex` by contract). First-load JS 136/166 KB of a 400 KB budget. Earlier runs at 75–79 under host load are listed in REPORT.md |

**Posture: 50 ✓ / 1 ⚠ / 0 ✗** (the ⚠ is ENC volume encryption at rest, delegated to the deploy target per SPEC §4.4)
