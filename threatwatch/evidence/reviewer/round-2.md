# Reviewer Round 2 — ThreatWatch (narrow re-verify)

- **Reviewed commit:** a9ee654 ("RB-1: Review Round 1 fixes + Security pass-1 remediation"), diff-scoped against d700300, which round 1 reviewed. The live stack reports `commit: a9ee654` on `/api/health` and web `/healthz`. Uncommitted changes to BUILD-STATE.md and GOVERNANCE-DECISIONS.jsonl, and the untracked `.claude/` and smoke-test.log, were not reviewed.
- **Mechanism:** independent reviewer in a fresh Task-tool context, read-only on source. Every round-1 item was re-derived from the code at HEAD, not from the RB-1 commit message.
- **Inputs:**
  - `git show a9ee654` for the server, web, e2e, SPEC, DECISIONS (ADR-027, ADR-028), prisma and migration diffs
  - ARCHITECTURE.md
  - `node scripts/invariant-lint.mjs`
  - `artifacts/web-baseline/FRONTEND-AUDIT.partial.summary.json`
- **Re-run by reviewer:**
  - `npx vitest run` over the 19 affected server files gave 180/180 passing. The files were: cursors, notifications, auth, rules, ws, admin/team, ingest, indicators, reports, alerts, all of `test/jobs/*`, unit pagination and unit signing.
  - Invariant lint: 41 of 42 machine-checked invariants pass. The one failure is INV-2, because `FRONTEND-AUDIT.json` is missing. That is expected until the wide-close `--full` run.
  - A standalone tsx probe of `idempotent()` against an in-memory key store reproduced R2-1. Script: scratchpad `idem-probe.ts`.
  - A node probe confirmed that aborting a pinned `http.request` after the response has started does not crash the process.
- **Live probe limits:** authenticated live probes were not possible. `SEED_DEMO=false`, so the demo login does not exist, and `/auth/register` answered 429 from the register limiter. Cursor and endpoint behaviour was verified through the integration suite instead. This round created no live data.
- **Accepted deviations (per caller, not re-raised):**
  - An invalid cursor is `VALIDATION_FAILED` with `errors[{field:'cursor',code:'INVALID_CURSOR'}]`.
  - Transition accepts `{to}`, with `{status}` as a deprecated alias.
  - The ingestion-health re-run reports `processed=4` but changes nothing.
  - There is no `?health=` deep link on integrations.

## Round-1 findings re-verified

| R1 | Status | Evidence at a9ee654 |
|---|---|---|
| R1-1 INV-11 T1–T5 | RESOLVED | Each T1–T5 effect test now uses `recordJobRun(ctx, '<job>', 'schedule', c => runXForOrg({...c, orgId}))`. Each asserts the returned row and `db.jobRun.findUniqueOrThrow` `{status:'succeeded', processed, skipped, failed}`, then re-runs and asserts processed 0. See detection-sweep.test.ts:68-84, email-drain.test.ts:73-79, delivery-drain.test.ts:89-94, escalation-check.test.ts:32-37 and ingestion-health.test.ts:91-105 (re-run processed=4 with no effect change is accepted). All pass. |
| R1-2 malformed cursor 500 | RESOLVED | `decodeCursor` now requires a UUID id (server/src/lib/pagination.ts:22,28). `pageCursor` throws `invalidCursor()` (:33-44), and the alert risk cursor is checked at alerts/queries.ts:45,68. test/integration/cursors.test.ts covers 10 lists × 3 crafted cursors with 400 `VALIDATION_FAILED` / `INVALID_CURSOR`, and a well-formed cursor still returns 200. No list can reach a `::uuid` cast with a bad id any more. One residual inconsistency is tracked as R2-3. |
| R1-3 alert list live insertion | RESOLVED | `useLiveRefresh(['alert.created','alert.updated'], checkFresh, {fallbackMs: POLL_MS})` at web/src/app/(app)/alerts/AlertsView.tsx:26,168. Polling now runs only while the socket is offline (LiveContext.tsx:190-191). LiveProvider is mounted app-wide in SessionGate.tsx:55. The e2e at web/e2e/w02-detection.spec.ts:17-23 opens /alerts, waits for the WS hello, ingests a burst and expects the banner within 3 s. |
| R1-4 unread env vars | RESOLVED | ADR-027 removes `API_INTERNAL_URL` and `NEXT_PUBLIC_APP_NAME` from SPEC §8. No reader or reference remains in web/, compose, .env.example or SPEC (grep). The new `SMTP_REQUIRE_TLS` is documented and read. |
| R1-5 `delivery_endpoints.enabled` unreachable | RESOLVED | Writer: `PATCH /api/v1/delivery-endpoints/:id` (routes/notifications.ts:209-213) calls `setEndpointEnabled`. That is an org-scoped `updateMany`, returns 404 on count 0, and is audited `endpoint.enable/disable` (services/deliveries/endpoints.ts:146-163). Readers: enqueue (notify/enqueue.ts:147,187) and the drain claim (deliveries/drain.ts:55). UI switch: NotificationsView.tsx:516,616. Tests: notifications.test.ts (PATCH, audit, 400 body, analyst 403, cross-org 404, no enqueue for a disabled endpoint) and delivery-drain.test.ts:152 (deliveries for a disabled endpoint stay pending and unattempted, then resume). SPEC §3 and §6 were amended. |
| R1-6 POST /rules Idempotency-Key | RESOLVED | routes/rules.ts:80 `idempotent(deps)`. rules.test.ts checks "honours Idempotency-Key on create": the replay returns the same id with `Idempotent-Replayed`, a different body under the same key gets 409 `IDEMPOTENCY_CONFLICT`, and no duplicate rule is created. |
| R1-7 error codes outside §7 | RESOLVED | ADR-027 lists `LIMIT_REACHED`, `SELF_RESET` (400) and `VERSION_CONFLICT`, `INDICATOR_EXISTS`, `ALERT_CLOSED`, `NOT_RETRYABLE`, `ENDPOINT_IN_USE` (409) in SPEC §7. `INVALID_CURSOR` moved under `VALIDATION_FAILED` (events/service.ts:124 and indicators/service.ts:67 now use `pageCursor`). The new in-flight answer uses `CONFLICT`, which is already in §7. |
| R1-8 idempotency race | PARTIAL | The concurrent-duplicate race is closed: a placeholder is claimed under UNIQUE(org_id,key) before the handler runs (services/idempotency/store.ts:23-43, migration 0002 with down.sql), and auth.test.ts:372 shows one 201 and one 409 `CONFLICT`. However, the same fix added a release-on-`close` path that loses the stored response when a client disconnects mid-request. See R2-1. |
| R1-9 concurrent delete 500 | RESOLVED | Org-scoped `deleteMany`/`updateMany` return 404 on count 0 at notify/policies.ts:179,200, deliveries/endpoints.ts:127, indicators/service.ts:218, reports/schedules.ts:127 and team/service.ts:278. Race tests in indicators, notifications and reports give [204, 404] with a single audit row. |
| R1-10 WS ignores must-change-password | RESOLVED | api/ws/hub.ts:152 `if (session.user.mustChangePassword) return refuse(socket, 428)`. ws.test.ts checks "refuses a session that must still change its password with 428". SPEC §7 428 row amended. |
| R1-11 transition `{to}` | RESOLVED (accepted deviation) | routes/alerts.ts:56-66,155-156 accept `to`, with `status` as an alias. Sending both is 400 on `status`, and sending neither is 400 on `to`. The web client sends `to` (alerts/[id]/AlertDetail.tsx:541), and alerts.test.ts covers to/both/neither/bad. |
| R1-12 pending after a no-insert verified batch | RESOLVED | ingest/ingest.ts:235-243 activates when `inserted + filtered + duplicates > 0`. ingest.test.ts covers activation on an all-filtered or all-duplicate batch and no activation on an all-rejected one. |
| R1-13 job_runs `finishedAt` clock | RESOLVED | services/jobs/runs.ts:27-28 derives `finishedAt` from injected `now` + measured elapsed time, on both the success and failure paths. |
| R1-14 change-note error not shown | RESOLVED | rules/[id]/RuleDetail.tsx:202-217 renders `fields.changeNote` with `aria-invalid` and `aria-describedby` and an error border. |
| R1-15 two dry-run panels / duplicate id | RESOLVED | RuleDetail no longer renders its own `DryRunPanel`. There is a single panel inside `RuleForm` (rules/_parts.tsx:437) with `useId` ids, and `dry-h` no longer appears anywhere in web/src. |
| R1-16 `?version=n` ignored | RESOLVED | RuleDetail.tsx:32 reads `version`, and :252-263 highlights the "version that raised the alert". |
| R1-17 settings gates | RESOLVED | Early-return access-denied EmptyState with fetches gated on `canRead`: TeamView.tsx:92,148, JobsView.tsx:74,101 and AuditView.tsx:128,132,156. |
| R1-18 alerts from/to + asset | RESOLVED | AlertsView.tsx:27,65-71,115-117 adds from/to with an ordering check, and AlertsView imports the new AssetFilter.tsx, which sets `assetId`. These UI controls have no tests yet (R2-5). |
| R1-19 W2 45 s bound | RESOLVED | web/e2e/detection-helpers.ts:54 `timeout: 30_000`. |
| R1-20 reset temp-password email | RESOLVED | team/service.ts:291-309 queues `temp_password` with `user:<id>:reset:<nonce>` when `EMAIL_TEMP_PASSWORDS`. team.test.ts checks two resets give two distinct rows and that nothing is queued by default. |
| R1-21 unreachable 360 min step | RESOLVED | drain.ts:15 `DELIVERY_MAX_ATTEMPTS = 6`. SPEC WS-OUT-4 amended (ADR-027). delivery-drain.test.ts:132-138 asserts backoffs [1,5,30,120,360] and dlq at attempts 6. ARCHITECTURE.md still says 5 (R2-6). |
| R1-22 writes after an earlier org check | RESOLVED | rules/service.ts:146 adds `orgId` to the `updateMany`. detection/engine.ts:162 changes `findUnique` to `findFirst({id, orgId})`. |
| R1-23 stale web-baseline evidence | PARTIAL | `FRONTEND-AUDIT.partial.summary.json` now records gitSha a9ee654, but the definitive `--full` `FRONTEND-AUDIT.json` (INV-2) is still absent. BUILD.md schedules that run for the wide close, so this is expected for now, but it remains open until then. The partial run also shows regressions (R2-2). |
| R1-24 assignee options need write | RESOLVED | AlertsView.tsx:174-180 fetches `/api/v1/alerts/assignees` under `alerts:read`, which is also the server permission (routes/alerts.ts:89). |

## Reviewer Gate checklist (round 2)

| # | Item | Verdict | Rationale |
|---|---|---|---|
| 1 | RESEARCH requirements implemented, not stubbed | PASS | RB-1 did not touch the engine, DSL, correlation, risk or search. Round 1's trace stands. |
| 2 | Every SPEC feature has code and behaviour tests | PASS | F-5 live insertion and POST /rules idempotency now have code and tests. The new UI controls lack tests (R2-5, NOTE). |
| 3 | Implementation matches ARCHITECTURE | PASS | There are no new layers. The pinned `http.request` replaces `fetch` inside the same single dispatch module. The doc wording lags (R2-6). |
| 4 | No invented features | PASS | The PATCH endpoint and the ingest per-IP limiter are recorded in ADR-027 and ADR-028. |
| 5 | Security invariants hold | PASS | Cursor 500s are gone. The new PATCH route is org-scoped with 404 cross-org (tested). SSRF egress is pinned to the validated address. The WS 428 gate holds. The production pinned path is untested (R2-4, WARN). |
| 6 | Cold read of primary flows | FAIL | R2-1: RB-1's idempotency middleware now duplicates the side effect when a client that timed out retries with the same key, which is the main case Idempotency-Key exists for. |
| 7 | Runnable | PASS | The stack is healthy at a9ee654, and the affected suites reproduce green. |
| 8 | Every `manual` invariant has a verdict | PASS | INV-11 is met for T1–T7 (table below). |
| 9 | Nothing declared is unreachable | PASS | `enabled` has a writer, a reader and tests. The dead env vars were removed from SPEC. |

### INV-11 (manual) verdict: PASS

| Trigger | Test file:line | Per-org `runXForOrg` via `recordJobRun`, injected `now` | Effect + job_runs row + re-run | Verdict |
|---|---|---|---|---|
| T1 detection-sweep | server/test/jobs/detection-sweep.test.ts:68-84 | yes | yes | PASS |
| T2 email-drain | server/test/jobs/email-drain.test.ts:73-79 | yes | yes | PASS |
| T3 delivery-drain | server/test/jobs/delivery-drain.test.ts:89-94 | yes | yes | PASS |
| T4 escalation-check | server/test/jobs/escalation-check.test.ts:32-37 | yes | yes | PASS |
| T5 ingestion-health | server/test/jobs/ingestion-health.test.ts:91-105 | yes | yes (re-run processed=4, no effect change; accepted) | PASS |
| T6 retention-sweep | server/test/jobs/retention-sweep.test.ts:39-61 | yes | yes | PASS |
| T7 report-generate | server/test/jobs/report-generate.test.ts:63-64, 87-89 | yes | yes | PASS |

## New findings (introduced or exposed by RB-1)

**R2-1 · FAIL · Idempotency-Key drops the stored result when a client disconnects, so a retry runs the side effect again (regression from the R1-8 fix)**
- **Where:** server/src/api/authz/idempotency.ts:57-62 (`res.once('close', …)` calls `releaseKey`). The interaction is with the `settled` guard at :43 and :52.
- **What's wrong:**
  - When the client's connection closes before the handler responds (for example, the client times out), `close` fires, deletes the placeholder and sets `settled = true`.
  - The handler keeps running and commits its side effect. When it then calls `res.json`, the wrapper sees `settled` and skips `finalizeKey`, so nothing is stored.
  - The client's retry with the same key finds no row, claims it again and runs the handler a second time.
  - Before RB-1, `storeResponse` ran whenever the handler responded, so this retry got a replay. This is a regression in the main Idempotency-Key use case (SPEC §3) on POST /alerts/*, /reports, /indicators and /rules.
- **Reproduced:** a scratchpad probe calls the real `idempotent()` with a UNIQUE-honouring key store and a 400 ms handler. The first request aborts at 100 ms. A retry after the handler finished gets `201 replayed=no {"calls":2}`, and the handler ran 2 times.
- **Fix:** revert and re-fix with the smaller change.
  - Delete the `res.once('close')` release at :57-62. A placeholder whose request truly died (process crash) is already recovered by `IDEMPOTENCY_IN_FLIGHT_STALE_MS` (store.ts:11,32).
  - Keep `releaseKey` only on a non-2xx response.
  - Add a test to auth.test.ts against `/slow-effect`: abort the first request mid-handler, wait for the handler to finish, then retry with the same key and expect `Idempotent-Replayed: true` and `calls === 1`.

**R2-2 · WARN · Web-baseline regressions on the RB-1 tree (target-size on 22 screens, correlations overflow at 375)**
- **Where:**
  - `artifacts/web-baseline/FRONTEND-AUDIT.partial.summary.json` (gitSha a9ee654) lists 48 regressions.
  - Root cause 1 is axe `target-size` (serious) on the sidebar `Wordmark` link in 41 cells (all app screens @1280, light and dark). Its selector is `.pl-2.mb-3 > [aria-label="ThreatWatch home"]`. RB-1's compact sidebar change (web/src/components/AppShell.tsx:304-308, `mb-6`→`mb-3`, nav rows `min-h-[44px]`→`min-h-8`) is the only change on that path.
  - Root cause 2 is `correlations@375` overflow (scrollWidth 390 > 375) after the CorrelationsView.tsx rework.
  - LCP is over budget on the alert, correlation and integration detail pages.
- **Caveat:** a baseline run held `.web-baseline.lock` while I read the summary, and several `load` timeouts in it look environmental. Re-read the summary when that run finishes.
- **Fix:**
  - Restore enough spacing or size around the wordmark target in the compact sidebar, for example keep `mb-3` but give the nav's first group `mt-2`, or keep `mb-6` and recover the height elsewhere. Then re-check the 32 px nav rows against target-size spacing.
  - Fix the 15 px overflow on /correlations at 375.
  - Re-run `web:baseline -- --failing` before the wide close. A REGRESSIONS block belongs to this round.

**R2-3 · WARN · Five cursor-paged lists still silently ignore a malformed cursor (SPEC §7 amendment not applied everywhere)**
- **Where:**
  - server/src/services/ingest/batches.ts:75 (`GET /integrations/:id/batches`) and :90 (`/integrations/:id/errors`)
  - server/src/services/notify/log.ts:52-53 (`/notifications/log`)
  - server/src/services/assets/service.ts:126-131 (`/assets`, both sorts; `decodeRiskCursor` returns undefined) and :335 (`/assets/:id/activity`)
- **What's wrong:** these still call `decodeCursor`, or check the UUID ad hoc, and fall back to the first page with 200. The other lists, and SPEC §7 as amended by ADR-027 ("a malformed list cursor is `VALIDATION_FAILED` with `errors[{field:'cursor', code:'INVALID_CURSOR'}]`"), return 400. No 500 is possible any more, but a tampered or stale cursor silently restarts paging, so "Load more" can duplicate rows.
- **Fix:**
  - Use `pageCursor(q.cursor)` in batches.ts and log.ts.
  - In assets, throw `invalidCursor()` when `q.cursor` is set and the matching decoder returns undefined.
  - Add these five paths to the `LISTS` array in test/integration/cursors.test.ts, seeding one integration and one asset for the `:id` paths.

**R2-4 · WARN · The production SSRF-pinned dispatch path (S1-1 fix) is never exercised by a test**
- **Where:** server/src/services/deliveries/dispatch.ts:50-64 (`pinnedPost`) and :86-91.
- **What's wrong:** every dispatch test injects `fetchImpl` (delivery-drain.test.ts:54, signing.test.ts:47-83), which takes the `if (opts.fetchImpl)` branch. Only `pinnedLookup` and `failureClass` are unit-tested. The code that actually performs production egress, including the lookup pin, `Content-Length`, the abort signal and the status mapping, has no test. A broken pin or a regression there would pass CI.
- **Fix:** add a dispatch test without `fetchImpl`.
  - Start a local `http.createServer` on 127.0.0.1.
  - Call `dispatchDelivery` with `allowHttp: true`, a `targetUrl` whose hostname does not resolve (for example `http://receiver.invalid:<port>/hook`), and a `resolver` that returns `{address:'127.0.0.1', family:4}`.
  - Assert that the request arrives with the signed headers and body, which proves the pin, and that a 503 maps to `{ok:false, statusCode:503}`. Add a slow receiver for the `timeout` class.
  - If `assertSafeUrl` refuses loopback even with `allowHttp`, inject the safe address through a test-only resolver seam.

**R2-5 · NOTE · New UI controls from RB-1 have no test**
- **Where:**
  - web/src/app/(app)/alerts/AlertsView.tsx:65-71 (from/to range and its ordering error)
  - web/src/app/(app)/alerts/AssetFilter.tsx (asset filter)
  - notifications/NotificationsView.tsx:516,616 (endpoint enable switch)
  - rules/[id]/RuleDetail.tsx:252-263 (`?version` highlight)
  - settings Team/Jobs/Audit access-denied states
- **What's wrong:** no e2e or component test drives these controls. The server side of each is covered.
- **Fix:**
  - Extend W4 to set a from/to range and an asset and assert the request query, and to assert the "To must be after from" error.
  - Extend W11 to toggle an endpoint and assert the "Disabled" badge.
  - Add a component test for the analyst access-denied EmptyState on /settings/team.

**R2-6 · NOTE · Documentation drift from RB-1**
- **Where:**
  - ARCHITECTURE.md:137 still says T3 goes to "`dlq` after 5", but ADR-027 and WS-OUT-4 say after the 6th.
  - ARCHITECTURE.md:30 and SPEC WS-OUT-5 (SPEC.md:289) still describe `fetch` with `redirect:'error'` selecting only `pending` rows. The code uses a pinned `http.request` that never follows redirects, and claims `pending|failed` rows of enabled endpoints only.
  - SPEC §5 S4 (rate limits) omits the new 600/min/IP ingest limiter (api/middleware/rate-limit.ts:139, ADR-028 S1-5).
  - deliveries/drain.ts:12 cites WS-OUT-6 (receiver guidance) for the disabled-endpoint rule.
- **Fix:** update those lines through ADR-027 or ADR-028 (no behaviour change), and cite SPEC §3 PATCH delivery-endpoints in drain.ts:12.

**R2-7 · NOTE · A failed `finalizeKey` leaves a placeholder that blocks the key for 5 minutes, then allows a re-run**
- **Where:** server/src/api/authz/idempotency.ts:44-45 (`finalizeKey(...).catch(warn)`) and services/idempotency/store.ts:32.
- **What's wrong:** if the finalize write fails after a 2xx (for example, a transient DB error), retries get 409 `CONFLICT` "still being processed" for `IDEMPOTENCY_IN_FLIGHT_STALE_MS`. After that the stale takeover runs the handler again.
- **Fix:** retry `finalizeKey` once before warning, or log at error with the key id so the duplicate window is visible. It is acceptable as-is if documented.

## Counts

- FAIL: 1 (R2-1)
- WARN: 3 (R2-2, R2-3, R2-4)
- NOTE: 3 (R2-5, R2-6, R2-7)
- Round-1 items: 22 RESOLVED, 2 PARTIAL (R1-8 → R2-1; R1-23 waits on the wide-close `--full`), 0 OPEN.
- Reviewer Gate: item 6 does not pass (R2-1). R2-1 is a regression the RB-1 batch introduced, so under BUILD.md's Review Round rule it is handled by revert-and-refix inside round 2. The smaller re-fix is above.

Verdict: FAIL (1F/3W/3N)
