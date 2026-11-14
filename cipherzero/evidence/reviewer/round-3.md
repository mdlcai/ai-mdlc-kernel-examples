Commit reviewed: af52231a351c7ff3a3c182542dbf23509a8a4517
Previous round reviewed: 32b23e16167f8ae37162304f2df1276224026a59
Type: settle check (round-2 surviving items 2, 6, 9 only)
Mechanism: Claude Code Task tool, general-purpose subagent (fresh context), read-only except this file

Inputs:
- artifacts/reviewer/round-2.md (item 2, 6, 9 definitions and FAIL items 1-4)
- `git diff --ignore-cr-at-eol 32b23e1..af52231` (e2e/*, apps/api/src/services/quantum.ts, apps/api/test/quantum.test.ts, apps/web/src/app/(console)/quantum/quantum-shared.ts)
- DECISIONS.md ADR-070 (the claims; the code was judged, not the claims)
- artifacts/logs/e2e-settle.log, artifacts/logs/smoke-settle.log, artifacts/logs/wb-run8.log (tail)
- Own run: `npx vitest run test/quantum.test.ts` in apps/api, 10/10 passed. The working-tree copies of quantum.ts, quantum.test.ts, vault.ts, quantum-shared.ts and the seven e2e specs below are byte-identical to af52231 (`git diff --stat af52231 -- …` is empty). HEAD e8666d6 differs from af52231 only in GOVERNANCE-DECISIONS.jsonl.

## Item 2 — tests/e2e exercise the shipped behavior: PASS
Each round-2 call site is fixed to the new contract:
- API MFA setup sends the password: e2e/approvals.spec.ts:26-29 and e2e/keys.spec.ts:26-29 post `data: { currentPassword: GOOD_PASSWORD }`. GOOD_PASSWORD is the password registerViaUi registers (e2e/helpers/app.ts:3, :37).
- UI MFA setup fills the password field: e2e/auth.spec.ts:68, e2e/w4-agent.spec.ts:22 and e2e/w6-apex.spec.ts:22 fill `#mfa input[name=currentPassword]`. That input exists inside `section#mfa`, in the setup form (apps/web/src/app/(console)/settings/settings-client.tsx:236, :254-256).
- Agent credentials: e2e/w4-agent.spec.ts:79-81 now asserts that "Issue credential" has count 0 and that the "Agents never hold long-lived credentials" text is shown (workload-detail-client.tsx:306). The czw_ reveal-once checks moved to the service workload (w4-agent.spec.ts:149-156): the token format, "I have stored it" hides the token, and the 12-character prefix stays visible. The token is then used against the API.
- W5: e2e/w5-quantum.spec.ts:75 expects 'Migrated: data re-sealed, source key disabled', which matches quantum-shared.ts:112.
- Dashboard drawer: e2e/dashboard.spec.ts:39-47 opens the drawer when "Open navigation" is visible, asserts the Primary nav and closes it again. Both aria-labels exist in ConsoleShell.tsx:93 and :103.
- Run evidence: e2e-settle.log has every affected spec passing in both projects. That covers approvals, auth 2FA, dashboard, all keys specs, both w4-agent specs, w5-quantum and w6-apex (log lines 8-87). The total is 79/80. The only failure is e2e/billing.spec.ts:47 (desktop): the refund status was still SUCCEEDED after 30 s. billing.spec.ts was not changed by this batch, and the failure is unrelated to items 2/6/9.
- Caveats, neither a FAIL:
  - The "rerun billing.spec --repeat-each=2: 12/12 pass" line is a one-line manual note appended to the log, not raw Playwright output.
  - The working tree now holds uncommitted billing/route/e2e-helper changes made after af52231. So the e2e log cannot be tied byte-for-byte to af52231 for billing. The specs in scope here are identical to the commit.
  - Recommend that the gate's final full e2e run be stamped with the reviewed sha.

## Item 6 — cold read, `operator_action_required` dead path: PASS
- The value is gone from the code. The `MigrationStep.action` union is `'migrated' | 'successor_created' | 'already_target' | 'policy_review_required' | 'failed' | 'missing'` (apps/api/src/services/quantum.ts:79).
- `git grep "operator_action_required\|Operator action required\|Hybrid successor key provisioned" af52231` finds no hits outside docs and logs.
- The former branch now records `failed` with "Asset is not bound to a key." (quantum.ts:457-460). `failed` is not in `COMPLETE` (quantum.ts:387), so such an asset correctly blocks validation instead of silently passing.
- Observation (not a FAIL): this branch is defensive only. Transport assets are refused at plan creation (quantum.ts:43-46), and every other inventory asset carries a keyId. The value it writes has other reachable writers that are tested, so no admitted state is dead.
- The new `policy_review_required` state (SEC-9) is coherent:
  - `keyScopeBlocker` (quantum.ts:411-431) checks the active bundle's `rules[].resources.keyIds` and active, unexpired grants' `keyIds`, and blocks unless the successor is named too.
  - It is not in `COMPLETE`, so the plan stays `migrating` (409) until the scope is fixed.
  - It runs before any successor is created, so no orphan key is left (quantum.ts:473-484).

## Item 9 — nothing declared is unreachable: PASS
Each `MigrationStep.action` value below has a production writer, and a test in apps/api/test/quantum.test.ts reaches it through POST /api/v1/quantum/plans/:id/advance. All tests passed in my run (10/10).

| Value | Production writer | Test through /advance |
|---|---|---|
| `migrated` | quantum.ts:517-525 (after `retireSource`) | :204 (main flow); :283 (after a partial-move retry, source `disabled` :284, object on successor :285) |
| `successor_created` | quantum.ts:507-515 (`moved.remaining > 0`) | :240-286. The trust double fails decrypt for the source key on the testing→migrating advance (:265-270), giving `successor_created`. The source stays `active` and the object stays on the source key (:271-272). A second forced failure gives 409 with the plan still `migrating` (:275-278). The next advance gives `validated` (:280-285). This matches round-2 FAIL item 3 exactly. |
| `already_target` | quantum.ts:467-469 | :316-324. The key is already HYBRID_X25519_MLKEM768, the step is `already_target` with from = to = the key, and the plan then validates. |
| `policy_review_required` | quantum.ts:474-484 via `keyScopeBlocker` :411-431 | :346-408. A deny rule scoped to the source key gives the step plus a detail naming the rule (:388-392). The data does not move and the key stays active (:393-394). The next advance is 409 (:395). Once the rule also names the successor, the retry validates and the object moves (:400-407). |
| `failed` | quantum.ts:493-500 (successor name held by a non-target key), :528-531 (createKey error), :458-459 (defensive, see item 6) | :336-343. The `held-aes-pq` key is AES256_GCM, which gives `failed` with "held-aes-pq is taken". The source stays `active` and the next advance is 409. |
| `missing` | quantum.ts:463-465 (key destroyed), :453-455 (asset row gone) | :327-333. The key is destroyed after plan creation, giving `missing`; the next advance is 409 and the plan stays `migrating`. |

Notes (not FAILs):
- Some writers are not tested on their own; in each case another writer of the same value is tested:
  - the grant arm of `keyScopeBlocker` (:424-429);
  - the "asset left inventory" arm of `missing` (:453-455);
  - the createKey-error arm of `failed` (:528-531).
  Item 9 asks that each admitted state be reachable and tested through its path, and every value meets that.
- The SEC-9 test edits the active policy bundle directly in the DB to arrange the rule. That is an arrangement step; the state under test is still reached through /advance.

UI labels: STEP_LABEL (apps/web/src/app/(console)/quantum/quantum-shared.ts:111-118) has exactly six keys. They match the six union values one-to-one, and each is writable as the table shows. There is no `operator_action_required` key. Its one consumer, quantum-client.tsx:438, falls back to the raw action for unknown values.

Settle verdict: PASS
