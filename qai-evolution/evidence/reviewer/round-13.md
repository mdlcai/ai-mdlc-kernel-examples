Commits reviewed: e74317a8c02c81d920bbd9f0173d2d6c0b41a67a (RB-14); re-checked as amended: e86426f976a40ddd994186be94cc02aaca0a3e58 (F-25), 4b4855c1bf303bca4a86ead8c27b5867f7a2c82f (F-26)
Previous round reviewed: 4b4855c (round 12, C-21, F-24, C-22, F-25, F-26)

# Reviewer Gate: Review Round 13 (RB-14 round 12 follow-ups, ADR-47; F-25 and F-26 re-checked as amended)

- **Mechanism:** a fresh-context subagent reviewed `git show e74317a` finding by finding against artifacts/reviewer/round-12.md (M-R12-1, M-R12-2, M-R12-3, N-R12-1) and the W10 selector change. It then re-checked F-25 and F-26 as amended by RB-14. The checks were made against SPEC.md §3 `orb` (with `foundAt` and the note on `queries`, `foundAt` and `queryRatio`), §5 F-24 AC, W10.3, DECISIONS.md ADR-47 and ARCHITECTURE.md §2.9. It ran these:
  - SDK tests: `pytest -k orb` gave **31 passed, 303 deselected in 24.4 s**.
  - API tests: `tests/test_jobs_orb.py` against qai-test-pg and qai-test-valkey gave **5 passed in 42.4 s**.
  - Web checks: `npx vitest run components/results/orb-panels.test.tsx` gave **1 file, 9 tests passed**. `npx tsc --noEmit` exits 0.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (31 pass, **2 fail**), 1 manual. INV-1 fails only on c473132 (F-24, carried forward as M-R12-0). INV-2 fails, which is pre-existing (N-R8-6). RB-14 has its governance decision (GD-0116).
  - One throwaway probe in the session scratchpad (review13/probe1.py), outside the repo, run with `python -I`:
    - it ran `run_orb(seed=7)` twice and compared foundAt, the whole result without timings, and `evidence_parts`;
    - it swept `durr_hoyer` over 300 seeds × budgets {1, 2, 5, 16, 60, 128} on random 64-item objectives to check 1 ≤ foundAt ≤ queries ≤ budget and foundAt = the last improving round's queries (or 1);
    - it ran seeds 1, 2, 3 and 11 at the defaults and seed 7 with budget 4.

  It did not run Playwright or a browser (W10 was reviewed statically against the markup, with the gitignored apps/web/test-results error context from an earlier run used as evidence of the selector bug). It did not touch docker, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-14 | e74317a Round 12 follow-ups (ADR-47) | **PASS** (1 minor, 3 notes) | M-R12-1, M-R12-2, M-R12-3 and N-R12-1 are all resolved. The docs were changed first (SPEC §3 and §5, W10.3, ADR-47, ARCHITECTURE §2.9). The W10 selector fix is correct and fixes a real strict-mode failure. The Optimize cell says "optimum at query N" even when the QAI best is not the optimum (N-R13-1). |
| F-25 | e86426f API kind `orb` (as amended) | **PASS** (1 note) | The runner passes `foundAt` through unchanged into the result and the evidence outputs, and the evidence stays deterministic. The API test does not assert `foundAt` (N-R13-3). |
| F-26 | 4b4855c Orb Lab web (as amended) | **PASS** (shares N-R13-1) | The panels now parse and show `foundAt`. The τ0 hint matches the SDK. The eyebrows run Stage 1 to 6 in SPEC order. The W10 selectors match the markup. |

Carried forward (human action): M-R12-0. F-24 (c473132) has no governance decision, so INV-1 still fails. Only a human can record it (`node scripts/govern.mjs decide --change F-24 --commit c473132`). It is outside RB-14, F-25 and F-26.

## RB-14 (e74317a): finding by finding

| Check | Result | Evidence |
|---|---|---|
| M-R12-1: `foundAt` semantics in `durr_hoyer` | PASS | `found_at = 1` at the start. It is set to `queries` (after `queries += k + 1`, so it includes the read of the measured candidate) only when a round improves (orb.py:362, 385). It is returned as `foundAt` (orb.py:388). So it is 1 when the initial guess stays best, and otherwise the queries count at the last improving round. A candidate measured earlier cannot be the final best without improving, unless values tie, so this is also "the query at which the final best was first measured". The sweep found 0 violations of 1 ≤ foundAt ≤ queries ≤ budget or of the last-improving-round rule over 1,800 runs, 529 of which kept the initial guess. |
| M-R12-1: result and SPEC shape | PASS | `optimize.qai.foundAt` (orb.py:501) matches SPEC.md:188. `queries` stays as the budget spent, as the note at SPEC.md:192-193 says. |
| M-R12-1: queryRatio | PASS | `search["foundAt"] / n_items` (orb.py:551), where n_items = 2^qubits = settings. For seed 7 it is 61/256 = 0.23828125, and the probe confirmed it is exactly equal. The SDK test asserts the equality (test_orb.py:189). |
| M-R12-1: reason text | PASS | "…QAI found the classical optimum at query 61 of a 128-query budget, against 256 classical evaluations." (orb.py:652-656, probe output). This text appears only on a pass, where `optimum_agreement` guarantees that the QAI best is the classical optimum. The "no fewer queries than checking every setting" suffix still tests `queries` (the budget) ≥ n_items, which is correct for "the search used". |
| M-R12-1: evidence outputs | PASS | `evidence_parts` adds `foundAt` to `qai` (orb.py:714). The probe got `{'best': 107, 'qExt': 19173.50…, 'value': 0.47532…, 'queries': 128, 'foundAt': 61, 'budget': 128}`, and the outputs and summary were identical across two runs. |
| M-R12-1: SPEC AC and W10.3 | PASS (N-R13-3) | SPEC.md:533 and SPEC.md:630 now state foundAt ≤ budget 128. The SDK test asserts `1 <= foundAt <= queries`, the queryRatio equality and the reason text (test_orb.py:188-190), plus the rounds rule (test_orb.py:163-165). |
| M-R12-1: web | PARTIAL (N-R13-1) | `parseOrbResult` reads `foundAt` with `num()`, which accepts only finite numbers (orb-panels.tsx:132, 267). The QAI cell carries `data-found-at` and reads "optimum at query 61; 128 of 128 queries spent" (orb-panels.tsx:640-642). When `foundAt` is missing (an older result), the attribute is omitted and only the budget spent is shown, as ADR-47 says. The Compare hint reads "Queries to find the optimum / classical evaluations", as ADR-47 says. On an inconclusive run the cell still says "optimum" (N-R13-1). |
| M-R12-1: smoke | PASS | scripts/smoke-test.mjs:453 reports "optimum at QAI query foundAt/queries". smoke-test.mjs is not one of the pinned runners (invariant-lint, web-baseline, govern, governance-policy). |
| M-R12-2: τ0 hint | PASS | `hint="τ0 = Q0/ω"` (orb-panels.tsx:471) matches `tau0 = q0 / omega` (orb.py:446) and the comment at orb.py:50. |
| M-R12-3: stage order | PASS | ARCHITECTURE.md:128-129 now reads 2 Simulate, 3 Optimize. The eyebrows run Report "Stage 6" (430), Model "Stage 1" (466), Simulate "Stage 2" (528), Optimize "Stage 3" (598), Validate "Stage 4" (816) and Compare "Stage 5" (735). This matches SPEC.md:520-525, the W10 header and ADR-47. The orb.py stage comments already used this order; "# 3. Optimize (before simulate…)" explains why it is computed first. |
| N-R12-1: runtime and size | PASS | ARCHITECTURE.md:133 reads "2–3 s single-threaded at the defaults (up to ~15 s and ~260 KB at the limits)". This matches the round 12 measurements (15.3 s and 261,344 B). ADR-46's "about 90 KB" (DECISIONS.md:383) is not qualified, but it describes the default result, which is accurate. |
| W10 selector change | PASS (N-R13-4) | Kpi renders `<dd … data-alignment={value}>` (orb-panels.tsx:354, 742). The alignment table rows are `<tr data-alignment={iterations}>` (orb-panels.tsx:767). The hint `dd` has no data attribute, so `dd[data-alignment]` matches exactly one element. The earlier run's error context shows that the bare `[data-alignment]` "resolved to 28 elements", which is a strict-mode violation. The KPI text is `sci(divergence, 2)`, and `Number()` parses it. |
| W10 foundAt and queryRatio assertions | PASS | `tr[data-search="qai"] [data-found-at]` matches the QAI cost `td`. `(foundAt / 256).toFixed(3)` matches `queryRatio.toFixed(3)`, the same float computed the same way ("0.238" for seed 7). `tr[data-search] td:nth-child(3)` is still the Q_ext column, because the `th` is child 1. |
| Docs first, ADR | PASS | SPEC, ARCHITECTURE and the new ADR-47 change in the same commit as the code, and they describe the code exactly. |
| Governance | PASS | GD-0116 records RB-14 (e74317a). |

## F-25 (e86426f) as amended

| Check | Result | Evidence |
|---|---|---|
| Result and evidence pass-through | PASS | The runner stores the SDK result unchanged and builds the evidence from `evidence_parts`, so `foundAt` reaches both without any API change. test_jobs_orb.py passes (5 tests). |
| Determinism | PASS | ORB-0 seed 7 gives **foundAt = 61** (queries 128, best 107, 55 rounds). Two runs give the same result without timings and identical evidence outputs and summary. Other seeds: 1 gives 19, 2 gives 54, 3 gives 35 and 11 gives 28, all pass. |
| Test coverage | NOTE (N-R13-3) | test_jobs_orb.py:137 still asserts only the always-true `queries <= 128` and does not check `foundAt` in the stored result or the evidence outputs. |

## F-26 (4b4855c) as amended

| Check | Result | Evidence |
|---|---|---|
| foundAt parse and cell | PASS (N-R13-1) | See the RB-14 table. The test asserts the cell text "optimum at query 61" (orb-panels.test.tsx:230), with a fixture queryRatio of 0.2383. |
| τ0 hint and eyebrows | PASS | See M-R12-2 and M-R12-3 above. |
| Typecheck and tests | PASS | tsc exits 0. 9 of 9 panel tests pass. |
| W10 static | PASS | Every selector the RB-14 diff changed or added matches the markup. The rest are unchanged since round 12. |

## Findings

- **N-R13-1 (minor, RB-14 / F-26): the Optimize cell says "optimum at query N" even when QAI did not find the optimum.**
  - **Evidence:** `run_orb(seed=7, budget=4)` is inconclusive. QAI best is 57, the classical best is 107, foundAt is 4 and queryRatio is 0.016. The panel still renders "optimum at query 4; 4 of 4 queries spent" (orb-panels.tsx:641). The Compare KPI shows 0.016 under the hint "Queries to find the optimum / classical evaluations" (orb-panels.tsx:745-749). By SPEC §3, `foundAt` is where the *final best* was measured, which is the optimum only when `optimum_agreement` passes. This is the same kind of labelling-honesty issue that M-R12-1 raised.
  - **Fix:** label the cell "best at query N", or say "optimum" only when `qai.best === classical.best`. Qualify or blank the Compare KPI when the two disagree. Add an inconclusive-fixture assertion to the panel test.
- **N-R13-2 (note, RB-14): `queryRatio` sets a hindsight QAI figure against the full classical cost.** Dürr–Høyer cannot stop at `foundAt`, because it does not know it has found the optimum, so it always spends the budget. The ratio foundAt / 2^qubits (0.238 for seed 7) is therefore a best-case, after-the-fact number compared with an exhaustive 256 evaluations. SPEC.md:192-193 and ADR-47 define and disclose it, so it meets SPEC. The hint could say "(in hindsight)", or the panel could show `queries / 2^qubits` next to it.
- **N-R13-3 (note, RB-14 / F-25): a few assertions are still weak or missing.**
  - test_jobs_orb.py:137 asserts `queries <= 128`, which is always true, and does not check `foundAt` in the stored result or the evidence outputs.
  - W10 `queries <= 128` (w10-orb.spec.ts:93) and `foundAt <= queries` (:97) hold by construction. The checks of W10.3 that can actually fail are `chosen[0] === chosen[2]` and the queryRatio equality.
  - No panel test covers a result without `foundAt`, which is the fallback for older results in ADR-47's consequences.
- **N-R13-4 (note, process): the round 12 static W10 review missed a strict-mode violation.** Round 12 passed `[data-alignment]` as "exists in the markup". At run time it matched 28 elements (the earlier run's error context in apps/web/test-results, which is gitignored). RB-14 fixes it. A static review checks that a selector exists but not that it matches exactly one element. Playwright was not re-run this round (docker is busy with a rebuild), so W10 with RB-14 has not been run to green yet.
- Carried forward: N-R8-6 / INV-2 (the web baseline is stale: 3 unaccepted failures, 20 of 27 screens, source changed since d7df5cb) is still open and pre-existing. The round 12 notes N-R12-2 to N-R12-6 were outside RB-14's scope and remain open.

Counts: 0 blocker, 0 major, 1 minor, 3 notes (plus M-R12-0 carried forward for a human).

## Carried forward (human action)

- **M-R12-0:** F-24 (c473132) has no governance decision. `npm run lint:invariants` reports `FAIL INV-1 c473132 "F-24: SDK Orb Lab (qai.orb, ORB-0)" has no governance decision`. A human must record it. This reviewer did not run govern.mjs. The release should not ship while INV-1 fails.

## Overall

RB-14 does what round 12 asked:
- `foundAt` is correct, deterministic (61 for ORB-0 seed 7), never more than `queries`, and carried in the result, the evidence and the reason.
- `queryRatio` is now informative.
- The τ0 hint and the stage order match SPEC.
- The runtime note is qualified.
- The W10 selector fix removes a real strict-mode failure.

F-25 and F-26, as amended, meet SPEC §3, §5 and W10.3. The tests pass (31 SDK, 5 API, 9 web) and tsc is clean. The one minor (N-R13-1) is a labelling fix worth a small follow-up before release.

Verdict: PASS (RB-14, F-25, F-26). The release still waits on the human governance action M-R12-0.
