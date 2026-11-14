# Reviewer Gate — round 1
Commit reviewed: 59bbee66f24280418cfbf0f10865bed24b59956c
Mechanism: Claude Code Task tool, general-purpose subagent (fresh context)
Inputs: RESEARCH.md, ARCHITECTURE.md, SPEC.md, working tree, invariants.json + lint summary, FRONTEND-AUDIT.nolh.summary.json, docs/previews/, artifacts/web-baseline/*.png, Verification Gate verdict, .mdlc/BUILD.md §Reviewer Gate
Verdict: FAIL

Note: the working tree shows many `M` files, but `git diff --stat` shows content changes only in BUILD-STATE.md and smoke-test.log. The other flags are line-ending noise, so the reviewed code is HEAD.

## Checklist
| # | Item | Verdict | Rationale (file:line) |
|---|---|---|---|
| 1 | RESEARCH requirements implemented genuinely | FAIL | Workflow E / W5.5 ("rotate assets to the hybrid profile, rewrap") is not implemented: `migrateAssets` only creates a `<name>-pq` successor key (apps/api/src/services/quantum.ts:362-430). Existing objects are never rewrapped and the source key is never retired. A plan still reaches `validated` (quantum.ts:556-577) with nothing moved, including a plan whose only vulnerable asset is the transport, which is marked `operator_action_required` and counted as done (quantum.ts:385-392). The other workflows are genuine: the PDP evaluates arbitrary documents (packages/core/src/policy/index.ts:78-113) and the APEX detectors are generic threshold/ratio logic over persisted records (apps/worker/src/apex/detectors.ts). |
| 2 | Every SPEC feature has code and behavioural tests | PASS | Every F-1..F-17 has route, service and UI code, plus supertest/vitest behaviour tests (e.g. apps/api/test/vault.test.ts:424, approvals.test.ts:97-145, apps/worker/test/billing-webhook.test.ts:64-139) and e2e W1–W6 (e2e/w1-protect.spec.ts … w6-apex.spec.ts). Test-path gaps on specific states are scored under item 9. |
| 3 | Matches ARCHITECTURE | PASS | Planes, modules and frameworks match §2.1 (apps/api, apps/trust-core, apps/worker, apps/web, packages/core). Minor variances only: agent grant scope is enforced in the api (apps/api/src/services/vault.ts:100-112), not in trust-core as §2.3 step 3 says; the "tenant-scoped repos" are inline `tenantId` where-clauses, not a repo layer (packages/core/src/db/index.ts); `crypto_operations` is written by trust-core in a separate transaction from the object row (apps/trust-core/src/service.ts:316 vs apps/api/src/services/vault.ts:277-314); the root package pins ESLint 9.39.5, not the 10.12.0 in §8. |
| 4 | No invented features | PASS | Every route in apps/api/src/routes/*.ts maps to a SPEC §3 row, and every page.tsx maps to a SPEC §6 screen. Extra columns (`users.deactivated_at`, `invites.revoked_at`) only serve SPEC features. |
| 5 | Security invariants | FAIL | Agent workloads can bypass grants entirely. `issueCredential` issues a long-lived `czw_` token to any workload, including kind `agent` (apps/api/src/services/workloads.ts:252-272). `bearerPrincipal` maps it to `type:'workload'` (apps/api/src/middleware/authenticate.ts:79-92). `grantScopeViolation` only checks `p.type==='agent'` (apps/api/src/services/vault.ts:106), and the default bundle rule allows workloads (packages/core/src/policy/index.ts:175-178). This breaks ARCHITECTURE §3 ("AI agents … act only through short-lived grants"). Everything else holds: limiter before argon2 (services/auth.ts:254 vs 274), `__Host-` cookies, CSRF (middleware/csrf.ts:27-31), nonce CSP with no `unsafe-inline`, HSTS gated, mTLS + SPIFFE (apps/trust-core/src/server.ts:29-33,67), redaction, no hardcoded secrets. |
| 6 | Cold read: no obvious bugs or broken wiring | FAIL | (a) Grant expiry never fires for used grants: authenticate sets `access_grants.claimed_at` on first use (authenticate.ts:99), and grant-expiry only claims `claimedAt: null` (apps/worker/src/jobs/grant-expiry.ts:17), so used grants stay `active` with no `grant.expire` audit or revocation publish. (b) Tenant-scoped job failures are swallowed (apps/worker/src/runner.ts:82, rethrow only for `global`), so the retry→DLQ path in ARCHITECTURE §5.4 is dead for 7 of 8 jobs. (c) `submitBundle` checks for another `pending_approval` bundle without reconciling rejected or expired approvals first (apps/api/src/services/policies.ts:329-330), so a new submit returns 409 until some GET runs `reconcileRejected`. (d) Evidence verify reads the head after streaming events (apps/api/src/services/evidence.ts:250-272), so concurrent appends can give a false `head_mismatch`. |
| 7 | Runnable | PASS | QUICKSTART.md §1–§3 commands exist and are wired: `dev:secrets`, `dev:infra`, `migrate:deploy`, `seed`, `e2e:run`, `web:baseline` (package.json scripts; packages/core/package.json). compose.yaml orders pki-init → migrate → trust-core → api×2 → worker → web → edge with health conditions. .github/workflows/ci.yml reproduces the Verification Gate sequence on Postgres/Redis services. The recorded gate is all exit 0, 271/271 tests and 80/80 e2e. |
| 8 | Manual invariants upheld | PASS | INV-11 PASS; see below. |
| 9 | Nothing declared is unreachable | FAIL | Approval kind `grant_high_privilege` is declared (apps/api/src/lib/executors.ts:9,17; approvals-client.tsx:65,72) but nothing creates it and no executor is registered. JobRun `failed` and the DLQ have no production-path test. ApexProposal `rejected` has no test at all. PolicyBundle `active`/`superseded`/`rejected` are reached only by calling the executor directly or faking the approval row. AccessGrant `expired` is reached only for never-used grants. Full table below. |

## Item 8 — manual invariants
- **INV-11 "Every scheduled trigger has an effect test": PASS.** Each job has an effect test:

  | Job | Test | Trigger |
  |---|---|---|
  | key-rotation | apps/worker/test/jobs.test.ts:147 | cron |
  | rewrap | jobs.test.ts:180 | manual; asserts retired at zero dependents |
  | grant-expiry | jobs.test.ts:268 | startup |
  | audit-checkpoint | jobs.test.ts:299 | — |
  | apex-scan | apps/worker/test/apex-scan.test.ts:63 | — |
  | quantum-inventory | apps/worker/test/quantum-inventory.test.ts:70 | manual |
  | retention-purge | jobs.test.ts:322 | — |
  | approval-expiry | apps/worker/test/approval-expiry.test.ts:55 | cron |

  Each trigger kind (cron, manual, startup) is driven through `runJob` (jobs.test.ts:152, :186, :280), and smoke-test.log shows a manual rotation and its effect.
  - Caveat (not blocking): `catchUp()` and `upsertSchedulers()` in apps/worker/src/scheduler.ts are never called by a test; the startup test passes `trigger:'startup'` directly.
  - Caveat (not blocking): `rewrap` has no schedule and is not in catch-up (packages/core/src/queue.ts:23-31; scheduler.ts:30). A failed post-rotation enqueue therefore leaves objects on the old version until someone runs rewrap manually.

## Item 9 — admitted states and their writers
| Entity | Status | Production writer | Test through that path | Verdict |
|---|---|---|---|---|
| Payment | succeeded / failed / refunded / disputed / charged_back; disputed→succeeded | apps/worker/src/jobs/billing-webhook.ts:82,85,87,89,92,93 | apps/worker/test/billing-webhook.test.ts:64,88,97,114,139 (real processor over stored webhook_events); route half at apps/api/test/billing.test.ts:54-87 | PASS (in simulated mode only Stripe can emit dispute events; the processor is still a real writer) |
| Subscription | pending / active / past_due / canceled; past_due→active | billing.ts (api):276, billing-webhook.ts:82,89,92,93, billing.ts:364 | api billing.test.ts:121,196; worker billing-webhook.test.ts:64,114,139 | PASS |
| PolicyBundle | draft | services/policies.ts:234 | policies.test.ts:94 | PASS |
| PolicyBundle | pending_approval | policies.ts:332-334 | policies.test.ts:136-140, e2e/policies.spec.ts:71 | PASS |
| PolicyBundle | active / superseded | trust-core/service.ts:543-547 via executePolicyActivation (policies.ts:423-468) | policies.test.ts:67-70,161-162 call `getExecutor('policy_activate')` directly, bypassing POST /approvals/:id/approve; no e2e or smoke test approves a policy_activate | FAIL (no test through the approve route) |
| PolicyBundle | rejected | policies.ts:147-173 (lazy `reconcileRejected`, on read only) | policies.test.ts:292-303 sets the approval to `rejected` with a direct DB update | FAIL (test fakes the transition) |
| Key | active / disabled / destroyed | keys.ts:204,312,355,404 | keys.test.ts:73,229,261; approvals.test.ts:97 via /approve | PASS |
| KeyVersion | active / deactivated / retired / destroyed | trust-core service.ts:282,399; worker rewrap.ts:101; service.ts:450 | keys.test.ts:73; jobs.test.ts:147,180,215; trust-core service.test.ts:208; e2e w3-rotation.spec.ts:36 | PASS |
| Workload | active ⇄ suspended | workloads.ts:228 | workloads.test.ts:216; e2e w4-agent.spec.ts:140 | PASS |
| AccessGrant | active / revoked | workloads.ts:392,451 | workloads.test.ts:260; e2e w4-agent.spec.ts:60,135 | PASS |
| AccessGrant | expired | grant-expiry.ts:23 | jobs.test.ts:268 seeds only `claimedAt:null` grants; a grant that was used can never reach it | FAIL (see item 6a) |
| ApprovalRequest | pending / approved / executed / rejected / expired | lib/approvals.ts; services/approvals.ts:148,191,216,110; approval-expiry.ts:22 | approvals.test.ts:97,147,163,173; approval-expiry.test.ts:55 | PASS |
| ApprovalRequest kind | grant_high_privilege | none: no creator and no `registerExecutor`; approving would return 409 at approvals.ts:136 | none | FAIL (unreachable) |
| ApexFinding | open / dismissed / resolved | worker apex/scan.ts:~110,168; api services/apex.ts:316 | apex-scan.test.ts:63,90; apex.test.ts:126; smoke.mjs:592-596 | PASS |
| ApexProposal | proposed / pending_approval / executed / validated | apex.ts:378,481,578; scan.ts:163 | apex.test.ts:143,174; apex-scan.test.ts:90; smoke.mjs:589-592 | PASS (e2e W6 stops at pending_approval) |
| ApexProposal | rejected | apex.ts:135 (on read), scan.ts:206 | none | FAIL (no test) |
| MigrationPlan | draft / testing / migrating / validated | quantum.ts:~480,510,531,577 | quantum.test.ts:121; e2e w5-quantum.spec.ts:16 | PASS for reachability (the semantics of `migrating` fail item 1) |
| JobRun | running / succeeded | runner.ts:47,65 | jobs.test.ts:162 | PASS |
| JobRun | failed (+ DLQ) | runner.ts:76-82; scheduler.ts:51-57 | ops.test.ts:77 inserts a `failed` row directly; ops.test.ts:24-37 adds to the DLQ directly; tenant failures never rethrow (runner.ts:82) | FAIL |

## Focus items
- **GD-0007 F-6 policy bundles & simulation:**
  - The PDP is a real fail-closed evaluator: digest plus both signatures (policy/index.ts:118-152), and POLICY_UNAVAILABLE when no verified bundle exists (trust-core service.ts:172-186).
  - The simulator takes any document.
  - Gaps: activation is never tested through the approve route; `rejected` is lazy and its test fakes the row; `submitBundle` does not reconcile first.
- **GD-0008 F-7 workloads & agent grants:**
  - TTL ≤ 900 s (routes/workloads.ts:50), issuer MFA (:127), and 401 on revoke are all correct.
  - FAIL: agent workloads can hold `czw_` credentials that bypass grant scope.
  - FAIL: used grants are never expired by the job.
- **GD-0009 F-8 key lifecycle:** create, rotate, disable/enable and destroy-via-approval are sound and tested through the route (approvals.test.ts:97-145). Minor: rotation's `next_rotation_at` uses `Date.now()` rather than the injected clock (trust-core service.ts:407).
- **GD-0010 F-9 vault:**
  - Fresh DEK per object, AAD bound, 404 for a foreign object before any key operation (vault.ts:330), purpose bound, `no-store`, INTEGRITY_FAILURE on tamper.
  - Issue: the agent `czw_` bypass (item 5).
  - Minor: `crypto_operations` is not written in the same transaction as the object row.
- **GD-0011 F-10 evidence:**
  - Head row lock, RFC 9162 Merkle, and verify recomputes the chain, checkpoints and hybrid signatures; tamper tests exist (evidence.test.ts:143-222).
  - Race: a concurrent append during verify can give a false `head_mismatch` (evidence.ts:250-272).
- **GD-0013 F-16 dashboard:**
  - Persisted data only, on `dbRead`; the 24h/7d range drives the buckets (services/dashboard.ts:17-91); tested (dashboard.test.ts:76-148).
  - Screenshots show honest zero states.
- **GD-0014 F-12 scheduled operations:**
  - All 8 jobs have effect tests (INV-11).
  - Defect: tenant job failures never retry or dead-letter (runner.ts:82).
  - Defect: rewrap is unscheduled.
  - Ops routes are owner/admin and tenant-scoped (ops.ts:176,194-221), but an owner or admin can trigger the global `retention-purge` job.
- **GD-0015 F-13 quantum:**
  - Inventory and usage-weighted exposure are genuine (quantum.ts:131-156).
  - The interop test is a real round-trip.
  - The `migrating` phase does not rewrap or retire anything; FAIL item 1.
- **GD-0016 F-14 APEX:**
  - 7 deterministic detectors over persisted records; scores 0–100.
  - INV-4 holds (apex imports, apex.ts:1-13).
  - Execution happens only via the `apex_remediation` approval executor (apex.ts:518).
  - Gap: `rejected` proposals are untested, and e2e W6 does not reach the W6.7 terminal.
- **GD-0017 F-11 approvals inbox:**
  - Not-requester (approvals.ts:96), role-per-kind (:98), MFA (:134) and the DB CHECK (migration.sql:643) are all enforced and tested.
  - The `grant_high_privilege` kind is unreachable.

## FAIL items (verbatim, actionable)
- [1] The W5.5 quantum migration does not rotate or rewrap assets; a plan validates while every ciphertext stays on the classical key — apps/api/src/services/quantum.ts:362-430, 556-577 — At `migrating`, rewrap each source key's protected_objects onto the hybrid successor (trust-core `Rewrap`, or an enqueued `rewrap` targeting the successor) and disable or retire the source key. Block `migrating → validated` until zero objects remain on source keys and no step is `operator_action_required`, or exclude transport assets from plans. Assert this in apps/api/test/quantum.test.ts:121.
- [5] An agent-kind workload can be issued a long-lived `czw_` credential that skips grant TTL, key and purpose scope — apps/api/src/services/workloads.ts:252-272; apps/api/src/middleware/authenticate.ts:79-92; apps/api/src/services/vault.ts:106 — Refuse `czw_` issuance when `workload.kind==='agent'` (409), and also reject `czw_` bearers whose workload kind is agent in authenticate. Add a test that an agent cannot obtain a `czw_` credential or use one to encrypt or decrypt.
- [6] Used agent grants never transition to `expired`, because first use sets `claimed_at` and grant-expiry requires `claimedAt: null` — apps/api/src/middleware/authenticate.ts:99; apps/worker/src/jobs/grant-expiry.ts:17 — Record first or last use in a separate column (e.g. `first_used_at`), keep `claimed_at` for the job (or drop the `claimedAt:null` filter, since the guarded `status:'active'` update already serializes), and update the APEX unused-grant detector (detectors.ts:256). Add a test: issue → use token → advance clock → run grant-expiry → `expired` plus a `grant.expire` audit event.
- [6/9] Tenant-scoped job failures are recorded `failed` and then swallowed, so nothing retries or reaches `cz-dead-letter`; JobRun `failed` and the DLQ have no production-path test — apps/worker/src/runner.ts:76-82; apps/worker/src/scheduler.ts:51-57; apps/api/test/ops.test.ts:24-37,77 — Rethrow after recording when any tenant run failed, and make retries re-claim the window. Add a worker test where `def.run` throws, asserting the `failed` job_runs row and, via the worker, the DLQ entry.
- [6] `submitBundle` returns 409 after the previous bundle's approval was rejected or expired, until a GET reconciles it — apps/api/src/services/policies.ts:329-330, 147-173 — Write PolicyBundle `rejected` eagerly when a `policy_activate` approval is rejected (services/approvals.ts:211-230) or expired (approval-expiry job), or at minimum call `reconcileRejected` at the top of `submitBundle`.
- [6] Evidence verify can report a false `head_mismatch` under concurrent appends — apps/api/src/services/evidence.ts:250-272 — Read `audit_heads` first and bound the event scan to `seq <= head.seq`, or run both reads in one REPEATABLE READ transaction.
- [9] Approval kind `grant_high_privilege` is declared but nothing creates it or executes it — apps/api/src/lib/executors.ts:9,17; apps/web/src/app/(console)/approvals/approvals-client.tsx:65,72 — Remove it from APPROVAL_KINDS, DECIDER_ROLES and the UI label maps. Otherwise implement the creator in `issueGrant`, register an executor, and add a route-level test.
- [9] PolicyBundle `active`/`superseded` are never reached through `POST /v1/approvals/:id/approve` in any test — apps/api/test/policies.test.ts:67-70,161-162 — Add an api test where a second user with fresh MFA approves the `policy_activate` request through the route, then asserts the new bundle is `active` and the previous one `superseded`.
- [9] PolicyBundle `rejected` is tested only by faking the approval row — apps/api/test/policies.test.ts:292-303 — Drive it through `POST /v1/approvals/:id/reject` (and an expiry case) and assert the bundle is `rejected`.
- [9] ApexProposal `rejected` has no test — apps/api/src/services/apex.ts:135; apps/worker/src/apex/scan.ts:206 — Add a test: propose → submit → a second person rejects via `POST /v1/approvals/:id/reject` → GET finding or run apex-scan → assert the proposal is `rejected`. Add a twin case for an expired approval.

## Narrative assessment
This is a well-structured, unusually disciplined codebase.
- **Separation of concerns is strong.** The crypto plane really is isolated: the root key exists only in trust-core and is deleted from `process.env` after load; mTLS uses a SPIFFE allow-list.
- **Correctness guards are consistent.** State changes use guarded `updateMany` with explicit from-states, audit appends are serialized by a row lock, and the error contract and tenancy 404s are uniform.
- **Delivery evidence is strong.** 271 unit tests, 80 e2e tests and all 120 web-baseline cells pass.

The defects are at the seams, not in the core primitives:
- An identity type the grant model didn't anticipate: an agent holding a workload credential.
- An overloaded column: `claimed_at` used both as the job claim and as a first-use marker.
- An error path nobody exercised: tenant job failures are swallowed.
- A workflow phase implemented as bookkeeping rather than effect: quantum `migrating`.
- Lazily reconciled states whose tests fake the transition.

The main risk areas are:
1. Agent privilege containment, which is the product's headline AI-safety claim.
2. Job reliability. Silent failure with no DLQ undermines rotation and rewrap guarantees, and rewrap has no schedule as a fallback.
3. The migration story in QUANTUM, which today overstates what a `validated` plan means.

Recommended follow-ups beyond the FAILs:
- Schedule `rewrap` and add it to catch-up.
- Test `catchUp()` directly.
- Drive e2e W6 through approval to `validated`.
- Use the injected clock in trust-core rotation.
- Consider per-IP-only login throttling in addition to IP+email.
- Restrict `retention-purge` manual runs to a platform operator rather than any tenant owner.
