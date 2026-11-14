# Reviewer Gate — round 6 settle check (FAIL-1 / F-6 only)
Commit reviewed: f5cd74b8f1a43981066a0fc7534ab01ac4a6df0a
Previous round reviewed: b76af58e0fcf6ad60b9aa9333e53e21123c7fbf6
Type: settle check scoped to round-6 FAIL-1 (F-6), per `.mdlc/BUILD.md` §Reviewer Gate (round-2 triage, settle check). The fix is commit 9abc2f8. HEAD f5cd74b differs from it only in GOVERNANCE-DECISIONS.jsonl.
Mechanism: Claude Code Agent tool, general-purpose subagent (fresh context), read-only except this file
Verdict: PASS

Inputs:
- artifacts/reviewer/round-6.md (FAIL-1 item text and the 12 focus-item verdicts). artifacts/reviewer/round-5-settle.md was used for format only.
- `.mdlc/BUILD.md` §Reviewer Gate
- `git show 9abc2f8`: ARCHITECTURE.md (+2 lines, §5.7 prose), CHANGELOG.md, DECISIONS.md (ADR-088), VERSION.md (0.4.2 → 0.4.3), apps/api/src/services/policies.ts, apps/api/test/policies.test.ts. I did not rely on the commit message or the ADR. I judged the code at HEAD.
- Files I opened: apps/api/src/services/policies.ts (createDraft :218-245, reconcileRejected :147-173, submitBundle :284-369, executePolicyActivation :425-486), apps/api/src/services/approvals.ts (:1-20, loadForDecision, approveRequest :125-205), apps/trust-core/src/service.ts (activateBundle :536-551, bootstrapTenant :554-622), apps/api/test/policies.test.ts (helpers :1-75, new test :131-169). I also grepped every writer of `policyBundle` and every caller of `activateBundle`.
- My own runs: `npx vitest run apps/api/test/policies.test.ts` (env loaded per ADR-041) passed 1 file, 12/12 tests. `npm run lint:invariants` gave 52 total / 51 machine (50 pass, 1 fail) / 1 manual. The one failure is INV-2: FRONTEND-AUDIT is stale because policies.ts and policies.test.ts changed after the baseline gitSha. Every round expects this, it is outside FAIL-1, and it is not re-raised here.

## Focus items (round 6, with F-6 re-judged by this settle check)
| ID | Verdict | Source |
|---|---|---|
| F-6 | PASS | This settle check (below) |
| F-7 | PASS | round-6.md |
| F-8 | PASS | round-6.md |
| F-9 | PASS | round-6.md |
| F-10 | PASS | round-6.md |
| F-11 | PASS | round-6.md |
| F-12 | PASS | round-6.md |
| F-13 | PASS | round-6.md |
| F-14 | PASS | round-6.md |
| F-16 | PASS | round-6.md |
| C-3 | PASS | round-6.md |
| C-4 | PASS | round-6.md |

## FAIL-1 re-derived at HEAD
| Check | Verdict | Rationale (file:line) |
|---|---|---|
| (a) A retry after trust-core committed but the API tx failed succeeds | PASS | trust-core's commit leaves the bundle `active` with `activatedAt` set (service.ts:542-548). The approval stays `approved` with `result.error` (approvals.ts:166-185). The same approver's retry re-enters at approvals.ts:139-140 and reaches the executor. `resuming` is true (policies.ts:433), so :434 no longer throws 409. The trust call is skipped (:449-455), which matters because trust-core would reject a non-pending bundle (service.ts:539). The guarded tx writes `approverId` and the audit (:457-479), and approveRequest then moves approved → executed (approvals.ts:187-203). A trust call that commits but loses its gRPC response takes the same path. |
| (b) approverId and `policy.activate` written exactly once, including under concurrent retries | PASS | The write is `updateMany where {id, tenantId, approverId: null}`, and the audit is written only when `count === 1` (policies.ts:459-463), in the same transaction. Two concurrent retries both skip trust-core and both run the tx. Postgres row locking makes the second UPDATE re-evaluate `approver_id IS NULL` after the first commits, so it gets count 0 and writes no audit. Both then return success, and `approval.execute` is itself guarded on `status: 'approved'` (approvals.ts:189-201). If two concurrent attempts both find the bundle still `pending_approval`, both call trust-core. The second activateBundle either fails the pending check (service.ts:539) or re-flips inside its own tx. In both cases at most one API tx wins the approverId guard. The new test calls the executor a third time and asserts exactly one `policy.activate` event (policies.test.ts:162-167). |
| (c) The resume path cannot bypass the two-person rule or the approved digest | PASS | The author/requester check (:435-438) and the digest check `payload.digest === b.digest` (:439-442) run before the resume branch, unchanged. Only the same approver can retry (approvals.ts:139), and loadForDecision re-checks the role and the requester and requires fresh MFA (:131-134). A bundle that is active "via some other approval" is not reachable. `policy_activate` approvals are created only by submitBundle, from a `draft` bundle (policies.ts:288, :346-353). `active` is written only by trust-core activateBundle (service.ts:547) and bootstrap v1 (:594-605). activateBundle's only caller is this executor (policies.ts:451; server.ts:58 is the RPC binding), and v1 is never a draft, so no approval targets it. A tenant has at most one pending bundle (policies.ts:330-331). A bundle leaves pending only through its own approval's executor, or through reconcileRejected when its latest approval is rejected or expired. A rejected or expired approval cannot be approved again (approvals.ts:142). So an approval that finds its bundle `active` is the one that activated it. If an earlier activation already set approverId, the guard writes nothing, so a different approver's identity can never overwrite it. Drafts start with approverId null (policies.ts:233-235), so the guard cannot block a legitimate first activation. |
| (d) supersededVersion | PASS | First-time: the bundle active before the trust call (:448), as before. Resume: the highest `superseded` version below `b.version` (:443-447). Active versions only increase: submit refuses a draft older than the active bundle (:290-293), and only one bundle can be pending at a time. So that row is the bundle trust-core just superseded, or null if there was none. The test asserts `supersededVersion: 1` on resume (:159). |
| (e) The regression test exercises the failure and fails on pre-fix code | PASS | The proxy fails only the first `$transaction` on the API db (policies.test.ts:143-155). The executor's reads are `findFirst`, so the first `$transaction` is the post-trust tx. The in-process trust-core uses the unproxied `db` (:20-25) and commits. The test asserts the half state `active` + `approverId: null` (:157-158), so the failure is real. On pre-fix code the retry would throw 409 at the old `status !== 'pending_approval'` guard, and `expect(result)` would fail. The test then asserts approverId = approver and exactly one audit event after a further call. Gap (observation, not a FAIL): it calls the executor directly, not `POST /v1/approvals/:id/approve`, so it does not assert approved → executed. That path is unchanged and passed under F-11 in round 6. |
| (f) No regression to normal first-time activation | PASS | For a `pending_approval` bundle the flow matches the old code. The only change is the `approverId: null` predicate, which always holds for a newly submitted draft. The existing first-time tests (two-person activation, the audit chain via verifyChain, self-approval refusal) pass in my run (12/12). |
| Other files in 9abc2f8 | PASS | ARCHITECTURE.md §5.7 (two lines of prose), CHANGELOG.md, DECISIONS.md (ADR-088) and VERSION.md are documentation only. Behavior changes only in policies.ts. |

## Observations (not FAILs)
- Residual window: a resume can be pre-empted by a later activation. Suppose trust-core activates v2, the API tx fails, and before anyone retries, another person submits and fully activates v3 (allowed, because v2 is `active`, not pending). Then v2 is `superseded`, the retry of v2's approval returns 409 at policies.ts:434, and v2 never gets an approverId or a `policy.activate` event. v3's own `policy.activate` event records `supersededVersion: 2`, so the chain still shows v2 was enforcing. This needs a second, independent two-person activation while a failed approval is still visible, and refusing to re-activate a superseded bundle is the right behavior. A later batch could allow an audit-only resume for a `superseded` bundle with `approverId: null` and a matching digest.
- Retry in the same window: the fix requested by round-6 item 1 asked for "reaches `executed`" via the approve route. The test proves it at executor level (see (e)).

Round verdict: PASS (FAIL-1 is closed at HEAD; F-6 settles PASS, so all 12 round-6 focus items are PASS)
