# Reviewer Round 1 — ThreatWatch

- **Reviewed commit:** d700300 (HEAD, "Verification Gate" chore). Uncommitted working-tree changes to BUILD-STATE.md and GOVERNANCE-DECISIONS.jsonl were not read and not reviewed.
- **Reviewer mechanism:** independent reviewer starting from a fresh Task-tool context, read-only on source. I wrote three parallel read-only sub-sweeps: (a) tenant isolation, authz, validation and error contract, plus live probes against https://localhost:9443; (b) frontend coverage of SPEC §6; (c) jobs, outboxes, ingest, reports, INV-11 and unreachable declarations. I re-checked every finding taken from a sub-sweep against the cited file:line before including it.
- **Inputs read:** SPEC.md, ARCHITECTURE.md, RESEARCH.md (via the SPEC trace), BUILD-PLAN.md, invariants.json, the server/ and web/ source and tests, the web/e2e specs, the artifacts/web-baseline summaries, README.md / QUICKSTART.md / .env.example, and the output of `node scripts/invariant-lint.mjs`.
- **Not read (independence rule):** DECISIONS.md, REPORT.md, CHANGELOG*, earlier reviewer rounds, BUILD-STATE.md.
- **Verification evidence cited:** "Verification Gate PASS — npm ci 0 · typecheck 0 · lint 0 (--max-warnings 0) · format:check 0 · server 403/403 · web 15/15 · build 0 · e2e 42/42 · coverage 93.29% stmts · web-baseline partial PASS · invariant lint 41/42 (INV-2 deferred to --full)".
- **Re-run by reviewer:**
  - Invariant lint: 43 total, of which 42 are machine-checked (41 pass; 1 fails, INV-2, because FRONTEND-AUDIT.json is missing as expected before `--full`) and 1 is manual (INV-11, assessed below).
  - Live stack: `/api/health` returns 200 with db, temporal and listener all ok. Unauthenticated `/api/v1/*` returns 401 `application/problem+json`.
  - A sub-sweep confirmed defect R1-2 live. Its probes left one org, "Probe Rev A" (`reva-probe@example.test`), with one report on the live stack. Clean these up if needed.

## Reviewer Gate checklist

| # | Item | Verdict | Rationale |
|---|---|---|---|
| 1 | RESEARCH requirements implemented, not stubbed | PASS | Everything traced end to end is real code with tests on realistic input: the ingest pipeline, the rule DSL, the risk formula, the 30-minute correlation window, the outboxes, all 7 schedules and search. |
| 2 | Every SPEC feature has code and behavior tests | FAIL | F-5 "live insertion" on the alert list is not implemented: it polls every 20 s (R1-3). POST /rules does not honour Idempotency-Key, which SPEC §3 requires (R1-6). |
| 3 | Implementation matches ARCHITECTURE | PASS | The layout is server/src/{config,lib,domain,services,api,worker}. Routes call services only (INV-13 passes). Temporal schedules T1–T7 are in worker/jobs.ts:21-27. There is no framework swap or shadow module. |
| 4 | No invented features | PASS | The only extras are small: an `alerts.group_key` column and some extra 409/400 codes (tracked in R1-7). No user-visible feature exists outside SPEC. |
| 5 | Security invariants hold | FAIL | Tenant isolation, RBAC, CSRF/Origin, secrets and SSRF guards are all sound. However, crafted pagination cursors produce 500 INTERNAL_ERROR on 9 list endpoints instead of 400 validation (R1-2), and concurrent requests with the same Idempotency-Key both run (R1-8). |
| 6 | Cold read of entrypoint and primary flows | FAIL | R1-2 is a reachable 500 in the primary list flows. R1-3 is a broken live-push wiring: `useLiveRefresh` is used only by the dashboard. |
| 7 | Runnable (README / QUICKSTART, Verification reproduces) | PASS | The QUICKSTART commands and TEST_DATABASE_URL (.env.example:12) are consistent, the prod stack answers on :9443 with healthy checks, and the Verification line reports every command at 0. Advisory note: the web-baseline evidence is at 8d09fef, not d700300 (R1-23). |
| 8 | Every `manual` invariant has a verdict | FAIL | INV-11 is only partly met. T6 and T7 assert job_runs through `recordJobRun`. T1–T5 assert the returned counts only, and they drive the effect test through the global runner rather than `runXForOrg` (R1-1). |
| 9 | Nothing declared is unreachable | FAIL | `delivery_endpoints.enabled` has no production writer that can set it to false, yet enqueue checks it (R1-5). The SPEC §8 env vars `API_INTERNAL_URL` (marked required) and `NEXT_PUBLIC_APP_NAME` have no reader anywhere (R1-4). |

### INV-11 (manual) verdict — FAIL (partial)

INV-11 requires each trigger's test to call `runXForOrg` with an injected `now`, assert the row-level effect plus the job_runs counts, and run twice to show exactly one effect.

| Trigger | Test file:line | Effect + run twice | job_runs counts | Per-org form for the effect | Verdict |
|---|---|---|---|---|---|
| T1 detection-sweep | server/test/jobs/detection-sweep.test.ts:63-75 (per-org only at :96) | yes | no (returned counts only) | no | FAIL |
| T2 email-drain | server/test/jobs/email-drain.test.ts:66-79 | yes | no | no | FAIL |
| T3 delivery-drain | server/test/jobs/delivery-drain.test.ts:80-106 | yes | no | no | FAIL |
| T4 escalation-check | server/test/jobs/escalation-check.test.ts:25-47 | yes | no | no | FAIL |
| T5 ingestion-health | server/test/jobs/ingestion-health.test.ts:77-102 | yes | no | no | FAIL |
| T6 retention-sweep | server/test/jobs/retention-sweep.test.ts:39-61 | yes | yes (`recordJobRun` + `db.jobRun` row at :55-56) | yes | PASS |
| T7 report-generate | server/test/jobs/report-generate.test.ts:63-64, 87-89 | yes | yes (`recordJobRun`) | yes | PASS |

## Focus items (governance REVIEW queue)

- **GD-0004 = F-4 (detection rules, Temporal engine, correlation): PASS WITH FINDINGS.**
  - Engine (server/src/services/detection/engine.ts): advisory lock, org-scoped FOR UPDATE batch, threshold history, suppression extend-or-create, required evidence, risk from org-scoped asset criticality, and notifications enqueued in the same transaction.
  - Rule DSL and diff: domain/rules.ts.
  - Correlation: domain/correlate.ts, with the 30-minute window and src_ip > asset > indicator > user priority.
  - All 8 default rules match SPEC (services/orgs/defaults.ts).
  - All 10 rule routes have the correct permissions (api/routes/rules.ts:58-121).
  - Correlation reads are org-scoped (services/correlations/service.ts:15-201).
  - Accept tests: w02-detection.spec.ts:10-29 covers 12 events, T1110 and the group. The second burst extending the group is tested at server/test/integration/detection/detection.test.ts:102.
  - Findings: R1-6 (FAIL), R1-7, R1-14, R1-15, R1-16, R1-19, R1-22.
- **GD-0006 = F-8 (asset inventory and profiles): PASS.**
  - Every statement in server/src/services/assets/service.ts is bound to `org_id`. PATCH validation is strict, and the update is audited inside the transaction (:286-327).
  - The asset detail returns the 30-day series, top sources, alerts, rules triggered, exposure and destination ports.
  - Criticality changes risk only for later alerts (server/test/integration/assets/assets.test.ts:260-277). The viewer is read-only on both the server (:282) and the UI (web/src/app/(app)/assets/[id]/AssetDetail.tsx:37,352). Accept is covered by web/e2e/w06-asset.spec.ts.
  - No F-8-specific defects. The shared defect R1-2 does not affect the assets lists, which already check the cursor UUID (:101,130).
- **GD-0007 = F-7 (search, threat-source profiles, event explorer, intel): PASS WITH FINDINGS.**
  - Every raw SQL statement in services/search/service.ts filters on `org_id`, and LIKE wildcards are escaped.
  - Search has a 60/min/user limit (api/routes/search.ts:33).
  - Global indicators are read-only, and deleting one returns 404 (services/indicators/service.ts:213-220). CSV import is capped at 1 MB / 10k rows with upsert.
  - Sources and events are org-scoped.
  - Accept is covered: an IP search returns the ip, alert and event groups (server/test/integration/search/search.test.ts:65), and web/e2e/w05 and w10 drive the UI.
  - Findings: R1-2 (the events and indicators lists return 500 on a crafted cursor), R1-7 (INDICATOR_EXISTS, INVALID_CURSOR), R1-9 (indicator delete race).

## Findings

### FAIL (must fix before ship)

**R1-1 · FAIL · INV-11 not met for T1–T5**
- **Where:** server/test/jobs/detection-sweep.test.ts:63-75, email-drain.test.ts:66-79, delivery-drain.test.ts:80-106, escalation-check.test.ts:25-47, ingestion-health.test.ts:77-102.
- **What's wrong:** The effect tests call the global `runX(ctx)` and assert only the returned `{processed, skipped, failed}`. No job_runs row is written or asserted, and `runXForOrg` appears only in the tenancy cases.
- **Fix:**
  - In each T1–T5 effect test, run `const run = await recordJobRun(ctx(now, orgId), '<job>', 'schedule', (c) => runXForOrg({ ...c, orgId }))`.
  - Assert the row-level effect.
  - Assert `run` and `await db.jobRun.findUniqueOrThrow({ where: { id: run.id } })` match `{status:'succeeded', processed, skipped, failed}`.
  - Re-run with the same `now` and assert there is no second effect and processed is 0. T6 (retention-sweep.test.ts:39-61) already follows this pattern.

**R1-2 · FAIL · A malformed pagination cursor returns 500 on 9 list endpoints**
- **Where:** server/src/lib/pagination.ts:18-24 and server/src/services/alerts/queries.ts:39-46.
- **What's wrong:**
  - `decodeCursor` and `decodeListCursor` validate the timestamp but never check that `id` is a UUID.
  - The id then reaches `::uuid` or a Prisma uuid comparison, which throws. The consumers are events/service.ts:126, rules/service.ts:86, correlations/service.ts:22, team/service.ts:141, team/audit-log.ts:41, indicators, reports and integrations.
  - Confirmed live: `?cursor=base64url("2024-01-01T00:00:00.000Z|notauuid")` returns 500 INTERNAL_ERROR on /alerts, /events, /indicators, /users, /audit-logs, /rules, /reports, /integrations and /correlations.
  - This violates SPEC §7 (400 for validation) and the §10 plan that every route returns a validation 400.
- **Fix:**
  - Have both decoders return `undefined` unless `id` matches the UUID regex.
  - Make every caller throw 400 `VALIDATION_FAILED` with `errors:[{field:'cursor', code:'INVALID_CURSOR', message}]` whenever `q.cursor` is set but does not decode. events and indicators already do this; the other seven lists need it.
  - Add one integration test per list.

**R1-3 · FAIL · The alert list has no live insertion**
- **Where:** web/src/app/(app)/alerts/AlertsView.tsx:14-19, 133-148.
- **What's wrong:** SPEC F-5 (SPEC.md:359, "live insertion") and W2.7 (SPEC.md:429, "a live `alert.created` push reaches the dashboard and alert list") are not implemented. The view polls every 20 s (`POLL_MS`), and the stale comment says "live push arrives with F-6". The only `useLiveRefresh` caller is web/src/features/dashboard/api.ts:94.
- **Fix:**
  - Call `useLiveRefresh(['alert.created','alert.updated'], checkFresh)` in AlertsView, feeding the existing "N new alerts" banner. Keep polling only as the fallback when the socket is offline.
  - Add a W3 or W4 e2e assertion that a newly ingested alert is announced on /alerts within 3 s.

**R1-4 · FAIL · Declared configuration has no reader (item 9)**
- **Where:** SPEC.md:664-665.
- **What's wrong:** `API_INTERNAL_URL` (marked required for web) and `NEXT_PUBLIC_APP_NAME` are read nowhere: not in web/, compose.yaml or .env.example. The app name is hard-coded.
- **Fix:** Either wire them up (read `NEXT_PUBLIC_APP_NAME` for the product name in the shell and title; use `API_INTERNAL_URL` for any server-side fetch and set it in compose), or remove them from SPEC §8 through a recorded decision. Do not leave a required variable that nothing reads.

**R1-5 · FAIL · `delivery_endpoints.enabled` false state is unreachable (item 9)**
- **Where:** server/src/services/notify/enqueue.ts:147,187 and server/src/services/deliveries/drain.ts:50-58.
- **What's wrong:**
  - No route or service ever writes `enabled=false`. deliveries/endpoints.ts only creates, rotates and deletes, and there is no PATCH route.
  - The `p.endpoint?.enabled` checks in enqueue are therefore dead branches, and the drain ignores the flag entirely.
- **Fix:** Either add an enable/disable toggle (`PATCH /api/v1/delivery-endpoints/:id {enabled}`, audited, plus a UI switch), make the drain skip or park deliveries for disabled endpoints, and test that path; or drop the column and the checks through a SPEC amendment.

**R1-6 · FAIL · POST /rules ignores Idempotency-Key**
- **Where:** server/src/api/routes/rules.ts:79.
- **What's wrong:** SPEC §3 (SPEC.md:102) requires side-effecting POSTs to accept `Idempotency-Key`. `createRule` has no `idempotent(deps)`, so a retried create makes duplicate rules. Every other non-secret-returning side-effecting POST has it: alerts, reports and indicators use `once` / `idempotent`.
- **Fix:** Add `idempotent(deps)` after `...write`, and add a replay test plus a test that a different body under the same key returns 409 `IDEMPOTENCY_CONFLICT`.

### WARN

**R1-7 · WARN · Error codes outside SPEC §7**
- **Where:**
  - `VERSION_CONFLICT`: server/src/services/rules/service.ts:146,163
  - `INDICATOR_EXISTS`: services/indicators/service.ts:100,127
  - `ALERT_CLOSED`: services/alerts/actions.ts:122
  - `LIMIT_REACHED`: services/reports/schedules.ts:89
  - `NOT_RETRYABLE`: services/notify/log.ts:100,108
  - `ENDPOINT_IN_USE`: services/deliveries/endpoints.ts:121
  - `INVALID_CURSOR`: events/service.ts:125, indicators/service.ts:68
  - `SELF_RESET`: team/service.ts:270
- **What's wrong:** SPEC §7 lists the allowed codes per status, and "`code` drives the UI copy".
- **Fix:** Map 409s to `CONFLICT` or `INVALID_TRANSITION`, and 400s to `VALIDATION_FAILED`. Keep the specific reason as `errors[].code`. Update IntelView.tsx:371, which matches on `INDICATOR_EXISTS`. Alternatively, amend SPEC §7 to list the codes.

**R1-8 · WARN · Idempotency race**
- **Where:** server/src/api/authz/idempotency.ts:154-173.
- **What's wrong:** The key row is stored only after the handler finishes, so two concurrent requests with the same key both execute the side effect.
- **Fix:** Before `next()`, insert a placeholder row under `UNIQUE(org_id,key)`. On collision with an unfinished row, return 409 `CONFLICT`. Finalize the row on 2xx and delete it otherwise.

**R1-9 · WARN · Concurrent deletes return 500**
- **Where:** services/notify/policies.ts:178,197, services/deliveries/endpoints.ts:126, services/indicators/service.ts:218, services/reports/schedules.ts:127.
- **What's wrong:** Each runs an org-scoped `findFirst` and then `delete({where:{id}})`. If a concurrent delete wins, Prisma raises P2025, which surfaces as 500.
- **Fix:** Use `deleteMany` / `updateMany({ where: { id, orgId } })` and return 404 when `count === 0`.

**R1-10 · WARN · WebSocket ignores must-change-password**
- **Where:** server/src/api/ws/hub.ts:136-146.
- **What's wrong:** A session with a pending forced password change can open /ws, while REST answers 428.
- **Fix:** Refuse the upgrade (403 or 428) when `session.user.mustChangePassword` is set, and add a test.

**R1-11 · WARN · Transition body field differs from SPEC**
- **Where:** server/src/api/routes/alerts.ts:54-57.
- **What's wrong:** SPEC §3 specifies `{to, note?}`, but the code accepts `{status, note?}`. The web client and e2e both use `status`.
- **Fix:** Accept `to` (optionally keeping `status` as an alias) or amend SPEC §3.

**R1-12 · WARN · Integration stays pending after a verified request with no inserts**
- **Where:** server/src/services/ingest/ingest.ts:238-239.
- **What's wrong:** The pending → active switch happens only when `inserted > 0`. A first verified request whose events are all filtered or duplicates leaves the integration pending, and manual activation is refused (services/integrations/service.ts:204-206). SPEC.md:92 says "first verified event".
- **Fix:** Activate when `accepted + filtered + duplicates > 0` after signature verification.

**R1-14 · WARN · Rule edit change-note error is not shown**
- **Where:** web/src/app/(app)/rules/[id]/RuleDetail.tsx:199-215.
- **What's wrong:** The thrown `changeNote` field error is never rendered on the textarea, so the user sees "Fix the highlighted fields" with nothing highlighted.
- **Fix:** Render `fields.changeNote` under the textarea with `aria-invalid` and `aria-describedby`.

**R1-15 · WARN · Two dry-run panels with a duplicate DOM id**
- **Where:** web/src/app/(app)/rules/[id]/RuleDetail.tsx:178 and web/src/app/(app)/rules/_parts.tsx:431-439.
- **What's wrong:** Writers see two dry-run panels, one for the current version and one for the draft, and both use `id="dry-h"`. This is an accessibility defect and confusing.
- **Fix:** Merge them into one panel with a "current / draft" toggle, or give them distinct headings and ids.

**R1-17 · WARN · Settings pages lack a client permission gate**
- **Where:** web/src/app/(app)/settings/team/TeamView.tsx (around line 58), settings/audit/AuditView.tsx (around line 116), settings/jobs/JobsView.tsx:48.
- **What's wrong:** A deep link opened by a role without access shows a 403 error with a Retry button that can never succeed.
- **Fix:** Early-return the access-denied EmptyState when `!can('users:read' | 'audit:read' | 'jobs:read')`, as NotificationsView does.

**R1-18 · WARN · Alerts filters are incomplete against SPEC §6**
- **Where:** web/src/app/(app)/alerts/AlertsView.tsx:21-31.
- **What's wrong:** SPEC §6 lists "from/to" and "asset" controls. Only relative windows are offered (`from` only, never `to`), and `assetId` can only be set from a pivot chip.
- **Fix:** Add from/to datetime inputs, as EventsView has, and an asset picker or text input that sets `assetId`.

### NOTE

**R1-13 · NOTE · job_runs `finishedAt` ignores the injected clock**
- **Where:** server/src/services/jobs/runs.ts:35,45.
- **What's wrong:** `finishedAt: new Date()` is used instead of the injected `now`.
- **Fix:** Use an injected clock so the timestamps are deterministic.

**R1-16 · NOTE · The alert's rule-version link is ignored**
- **Where:** web/src/app/(app)/alerts/[id]/AlertDetail.tsx:279 → rules/[id]/RuleDetail.tsx:24.
- **What's wrong:** The link carries `?version=n`, but RuleDetail never reads it.
- **Fix:** Highlight that version in the history list.

**R1-19 · NOTE · W2 e2e is looser than the acceptance bound**
- **Where:** web/e2e/detection-helpers.ts:54.
- **What's wrong:** `waitForGroup` polls for up to 45 s, but F-4 acceptance says "within 30 s".
- **Fix:** Use `timeout: 30_000` in the W2 acceptance test.

**R1-20 · NOTE · Admin password reset never emails the temporary password**
- **Where:** server/src/services/team/service.ts:265-292.
- **What's wrong:** The reset never queues a `temp_password` email, even with `EMAIL_TEMP_PASSWORDS=true`. createUser does (:194-202).
- **Fix:** Queue it when the flag is on, using businessRef `user:<id>:reset:<nonce>`, and add a test.

**R1-21 · NOTE · The last delivery backoff step can never be used**
- **Where:** server/src/services/deliveries/drain.ts:13-14.
- **What's wrong:** The 360-minute step is unreachable because dlq happens after the 5th attempt. This matches the internally inconsistent SPEC WS-OUT-4.
- **Fix:** Clarify SPEC, then either drop the step or dlq after 6 attempts.

**R1-22 · NOTE · Writes that depend on an earlier org check**
- **Where:** server/src/services/rules/service.ts:145 and server/src/services/detection/engine.ts:162.
- **What's wrong:** `updateMany({where:{id,currentVersion}})` and `correlationGroup.findUnique({where:{id}})` omit `orgId`. They are safe only because an org-scoped read happened first.
- **Fix:** Add `orgId` to both as defence in depth.

**R1-23 · NOTE · Web-baseline evidence is stale**
- **Where:** artifacts/web-baseline/FRONTEND-AUDIT.partial.summary.json and FRONTEND-AUDIT.nolh.summary.json.
- **What's wrong:** Both record gitSha 8d09fef, not d700300, and the definitive `--full` FRONTEND-AUDIT.json (INV-2) has not been produced yet.
- **Fix:** Run web-baseline `--full` at the release commit before ship.

**R1-24 · NOTE · Assignee filter options depend on write permission**
- **Where:** web/src/app/(app)/alerts/AlertsView.tsx:154.
- **What's wrong:** Assignee options are fetched only when `canWrite`, so viewers can filter only by "me" or "none".
- **Fix:** Fetch them under `alerts:read`.

## Counts

- FAIL: 6 (R1-1, R1-2, R1-3, R1-4, R1-5, R1-6)
- WARN: 10 (R1-7, R1-8, R1-9, R1-10, R1-11, R1-12, R1-14, R1-15, R1-17, R1-18)
- NOTE: 8 (R1-13, R1-16, R1-19, R1-20, R1-21, R1-22, R1-23, R1-24)

Reviewer Gate: items 2, 5, 6, 8 and 9 are FAIL.

**Verdict: FAIL** (6 FAIL findings; Reviewer Gate FAIL on items 2, 5, 6, 8, 9)
