Commits reviewed: ccf7d96f1ac1df8e29cec401adb1dcece1bd9953 (RB-21, Round 20 follow-ups: N-R20-1, N-R20-2, N-R19-3, ADR-50)
Previous round reviewed: round 20 (RB-20 PASS with minors; F-31 and F-33 PASS; F-32 and F-34 PASS with minors; no FAIL)

# Reviewer Gate: Review Round 21 (RB-21, the Round 20 follow-ups)

- **Mechanism:** a fresh-context subagent read `artifacts/reviewer/round-20.md` and `git show ccf7d96` (DECISIONS.md, SPEC.md, protocol.py, test_protocol.py; the added round-20.md was not re-reviewed). It checked the diff against SPEC.md §4 :115 and :119 (the jobs and requeue rows), §3 :161 (the protocol params and slot), the result schema at :244-245, §5 F-7 :374 (the gate rule), §5 F-32 :731-736 (the governance run), the protocol AC at :756 and :765, W12.1 at :862-867, the §7 code list at :959, and ADR-49 and ADR-50 (DECISIONS.md :412, :446-463). It read the surrounding code: protocol.py `_governance` (:384-450) and `_RunOutcome.checkpoint` (:372); evolution.py `gate` (:279-304) and `decide`; services/jobs.py `requeue` (:320-355), `check_protocol_slot` (:134-146) and `PROTOCOL_RETRY_SEC` (:53); errors.py :39; apps/web/e2e/w12-protocol.spec.ts (all of it); and the governance types in apps/web/components/results/protocol-panels.tsx. It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider tests/test_protocol.py tests/test_evolution.py` **errored at collection** (exit 4): `tests/conftest.py:6` imports `hypothesis`, which is not installed in the host Python 3.13 (not in user or system site-packages; `pip list` shows pytest 8.4.2, numpy, scipy, but no hypothesis). Round 20 ran the same command and got 73 passed, so the host environment has changed since then. Neither test file uses a conftest helper, so it re-ran with `--noconftest`: **74 passed** in 254.04 s (exit 0). That is the 73 from round 20 plus the new `test_governance_rejects_a_tie_at_epsilon_zero`, and it includes `test_defaults_with_seed_2026_pass_all_six`. No package was installed.
  - Harnesses in the session scratchpad (`$TMP/r21/`), each under `python -I`:
    - `gov_seed2.py`: the round-20 seed-2 case (nodes 6, runs 2, splits 1/1/1, candidates 1, decisionSeeds 1, evaluationSeeds 2, shots 64, baseline `{p 2, maxiter 60}`) at ε 0 and 0.002; a forced no-adoption case (seed 5, ε 0, every run marked not adopted); and seeds 1-10 at ε 0.
    - `revert_check.py`: the seed-2 case at ε 0 with `GOVERNANCE_MIN_EPSILON` monkeypatched to 0, which simulates the pre-RB-21 code.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual. The one fail is INV-2 (web baseline: status fail, 2 of 29 screens, stale since 3f35306), the known carry. Nothing new fails.
  - `npm run lint:py`: "All checks passed!" and "163 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 1 (RB-21, GD-0158, "minimum for project architecture"); pending review none; in rework none; blocked none.

  It did **not** run Playwright (so W12 has not been run end to end), the smoke test, the web baseline, the API suite or the web vitest suite. Playwright, smoke and the baseline are waiting on a stack rebuild. RB-21 changes no API or web code, so the API and web suites were not needed. It did not touch Docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json or `.env`. It ran govern.mjs only as `queue`. It edited only this file, installed nothing, and staged and committed nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-21 | ccf7d96 Round 20 follow-ups | **PASS with minors** (0 blockers, 0 major, 1 minor, 3 notes) | N-R20-1 is fixed. At ε 0 with no adoption, governance now passes and the protocol is inconclusive, not fail. The test catches the old behaviour. The search gate is unchanged. N-R20-2 and N-R19-3 are fixed in SPEC. The new result field `epsilonGate` is missing from the SPEC result schema (N-R21-1). |
| F-31 | 91d8e36 SDK governance gate | **PASS** (carried) | RB-21 does not change `gate` or `decide`. The gate still accepts an equal J at ε 0, as SPEC :374 says it should (test_evolution `test_gate_rejects_equal_j_as_not_better` covers the ε > 0 side). N-R19-6 is still a note. |
| F-32 | f0da132 qai.protocol | **PASS** (given RB-21) | N-R20-1 is closed. N-R20-3 is still a note: governance gates against the last accepted checkpoint, not Ψ\*, and RB-21 did not reword SPEC :732. The seed-2026 defaults still pass all six. |
| F-33 | 7931aff protocol job kind | **PASS** (carried) | SPEC §4 now lists the requeue 429 (N-R20-2). The code is unchanged and matches it. N-R19-7 is still a note. |
| F-34 | e1fdd2e /app/protocol | **PASS** (given RB-21) | SPEC W12.1 now matches the e2e order (N-R19-3). W12 was not run. N-R20-4 (the Advanced re-open edge case) is still a note. |

## Check by check

| Check | Result | Evidence |
|---|---|---|
| N-R20-1: tie rejected at ε 0, no adoption | PASS | `gov_seed2.py` at ε 0: runs adopted `[False, False]`, governance run 1, worse **rejected** (J 0.17825147689034682 = jCheckpoint), invalid and missing rejected, governance pass, protocol **inconclusive**. Round 20 recorded `fail` for the same case. |
| N-R20-1: the new test catches the old behaviour | PASS | `revert_check.py` (margin set to 0): protocol status fail, governance fail, worse accepted, which is exactly the round-20 result. test_protocol.py:224-245 asserts governance pass, all three rejected, `epsilon` 0.0 and `epsilonGate` 1e-9 on each injection, and status not fail. So the test would fail on the pre-RB-21 code. |
| N-R20-1: ε > 0 is unchanged | PASS | The seed-2 case at ε 0.002: `epsilon` = `epsilonGate` = 0.002, the threshold is 0.176251, all rejected, inconclusive. `max(ε, 1e-9)` is ε for any ε ≥ 1e-9, and SPEC :161 allows 0-0.1. |
| N-R20-1: wider ε 0 sweep | PASS | Seeds 1-10 at ε 0: governance pass and all three rejected in every case, protocol inconclusive in every case. This includes seeds 3 and 10, which round 20 flagged as no-train-acceptance cases. The forced no-adoption case (seed 5, ε 0) also passes. |
| N-R20-1: ε_g is used only for the injections | PASS | `eps = max(prm["epsilon"], GOVERNANCE_MIN_EPSILON)` is local to `_governance` (protocol.py:391). It is passed only to the three `gate(...)` calls (:424). `_run_once` and the search still read `prm["epsilon"]`. evolution.py `gate` and `decide` are unchanged. The test_evolution suite is unchanged and passes. |
| N-R20-1: the margin is sound | PASS | The train chain is built by the gate, so J(checkpoint) ≤ J(Ψ₀) always holds, and Ψ₀ is re-scored deterministically on the same train set and seeds. So the "worse" injection can only tie or be strictly worse, and a 1e-9 margin turns the tie into a rejection. It cannot make a true gate defect (an accepted strictly-worse J) pass, because a defect would still accept or change the sha. The "invalid" and "missing" injections are rejected before the J rule, so the margin does not affect them. |
| N-R20-1: `epsilon` and `epsilonGate` are recorded | PASS in code, **gap in SPEC** | protocol.py:433-434 records `epsilon` (the configured value) and `epsilonGate` on every injection. ADR-50 Consequences (DECISIONS.md:463) says so. The SPEC result schema (SPEC.md:245) still lists `injections[{kind, decision, reason, j?, jCheckpoint, epsilon, sha256After}]`, without `epsilonGate`, and the SPEC :731 sentence does not say it is recorded. See N-R21-1. |
| N-R20-1: SPEC and ADR text match the code | PASS | SPEC :731 ("with ε_g = max(ε, 1e-9), so a candidate that only ties the checkpoint is not accepted (ADR-50)") and ADR-50 Decision match the code. The reason string is the gate's own `is not <=` reason, as ADR-50 says. |
| Side effect: the evidence hash | Accepted | Adding `epsilonGate` changes the protocol result payload, so a protocol evidence sha256 from before RB-21 differs from one after it, for the same params. Determinism holds within a version (`test_same_params_same_result_and_evidence` passes). Nothing pins a golden protocol hash. The web fixture (protocol-panels.fixtures.json) and the `injections` type (protocol-panels.tsx:97) do not carry the field, and the parser ignores unknown keys, so this is harmless. |
| N-R20-2: SPEC §4 lists the requeue 429 | PASS | SPEC.md:119 now reads "for a `protocol` job, 429 QUEUE_FULL with `Retry-After: 60` while another protocol is queued or running". This matches services/jobs.py:327-329 → `check_protocol_slot` (:139-146: `ApiError("QUEUE_FULL", 429, …, Retry-After = PROTOCOL_RETRY_SEC)`), and `PROTOCOL_RETRY_SEC = 60` (:53). QUEUE_FULL (429) is in the §7 list (:959) and in errors.py:39. "Another" is accurate, because the job being requeued is dead or failed and so is not counted. SPEC :161 still says only "a second submit". See N-R21-2. |
| N-R19-3: W12.1 matches the e2e | PASS | SPEC :865-867 has: configure Advanced; with another protocol of the same sizes queued or running, the submit shows the 429 retry time and stays on the form, with the first posted through the API; once it is done, the submit lands on the job page. w12-protocol.spec.ts follows the same order. It fills Advanced (:51-58), posts the first protocol with `SMALL` through `page.request.post` and expects 202 (:61-66), clicks Run and expects the alert with "Try again in N s." and the URL still `/app/protocol` (:67-71), polls the first job until `succeeded` (:73-81), then clicks Run and waits for `/app/jobs/{id}` (:83-84). The file's doc comment (:3-13) says the same. A narrow race is left: if the worker finishes the API-posted job before the first click, the 429 assertion fails. The click comes straight after the 202, and a protocol in a fresh interpreter takes seconds, so this is unlikely and does not need a finding. |
| Lint, invariants, queue | PASS | ruff check and format are clean. The invariants show INV-2 only (the carry). The queue holds RB-21 for human approval. |
| Regressions | None found | Only these files changed: DECISIONS.md, SPEC.md, protocol.py (+5/−2), test_protocol.py (+24) and the round-20 artifact. The SDK suites pass. |

## Findings

- **N-R21-1 (minor, RB-21 / F-32): the SPEC result schema does not list the new `epsilonGate` field.**
  - Location: SPEC.md:245 lists `injections[{kind: worse|invalid|missing, decision, reason, j?, jCheckpoint, epsilon, sha256After}]`. protocol.py:434 now also writes `epsilonGate`, and ADR-50 (DECISIONS.md:463) says "the result records both `epsilon` and `epsilonGate`".
  - Impact: a client or verifier written against SPEC §3 would not expect the field. The CLAUDE.md guardrail says the SPEC changes first. The schema also no longer says what `epsilon` means now that there are two values (the configured ε, not the one the gate used).
  - Fix: change :245 to `…, jCheckpoint, epsilon, epsilonGate, sha256After}]`. Optionally add "(the configured ε and the ε_g the gate used)" at :731. This is a doc-only change.
- **N-R21-2 (note, RB-21 / F-33): SPEC :161 still says "a second submit" only.**
  - Round 20 suggested "a submit or requeue" at :161 as well as the :119 change. Only :119 was changed. The two lines do not contradict each other, because §4 :119 now covers requeue, but :161 (and the AC at :756 and :765) read as submit-only. A one-word change would close this.
- **N-R21-3 (note, F-32): at ε 0 the governance reason string hides the margin.**
  - The gate formats `J(Psi_n) - epsilon` to 6 decimals, so the seed-2 "worse" reason reads "rejected: J 0.178251 is not <= J(Psi_n) - epsilon = 0.178251". That is true, but a reader sees two equal numbers and a rejection. The recorded `epsilonGate` explains it, but the web Governance panel does not show that field. A possible fix, if wanted: print ε_g in the reason, or show `epsilonGate` in the panel. This is cosmetic.
- **N-R21-4 (note, environment): the host Python is missing `hypothesis`, so the SDK conftest cannot load.**
  - `python -I -m pytest …` (and so `npm run test:sdk` on this host) stops at `tests/conftest.py:6` with ModuleNotFoundError. Round 20's identical command passed, so the package has been removed from the environment since then. This is not a code defect, and CI is not affected if it installs the dev deps. This round's 74 passes used `--noconftest`, which is valid because neither file imports from conftest. Before the next full SDK run, reinstall the SDK dev dependencies (`pip install -e packages/qai-sdk[dev]` or the equivalent).

Counts: 0 blockers, 0 major, 1 minor, 3 notes.

## Carried forward (human action)

- **Queue:** RB-21 (GD-0158) is waiting for human approval. This round is its review evidence. Nothing is pending review or in rework. N-R21-1 is a one-line SPEC fix that can go in a later RB commit and does not block.
- **Still open from earlier rounds:** N-R20-3 (governance gates against the last accepted checkpoint, not Ψ\*; reword SPEC :732 or change the code), N-R20-4 (the Advanced re-open edge case), and the notes N-R19-5, -6, -7 and -8. All are notes.
- **INV-2:** still fails (web baseline stale; it covers 2 of 29 screens). It needs a quiet-host baseline run or an explicit human acceptance.
- **W12, smoke and the web baseline:** not run, pending a stack rebuild. Run `npm run e2e` (W12), `./smoke-test.sh` and `npm run web:baseline` before release 0.8.0.
- **Environment:** reinstall `hypothesis` (N-R21-4) before the next `npm run test:sdk`.

## Overall

- RB-21 closes N-R20-1 with the smallest correct change. A 1e-9 margin, used only by the governance injections, turns the ε 0 tie into a rejection, so a correct gate no longer fails governance. A real gate defect still would. The configured ε still drives the search.
- The round-20 repro now gives inconclusive with governance pass, and all ten seeds at ε 0 pass governance. Simulating the old code brings the false fail back, so the new test does its job.
- The SPEC changes for N-R20-2 and N-R19-3 match the code and the e2e.
- The one gap: the new `epsilonGate` field is documented in ADR-50 but not in the SPEC result schema.

Verdict: PASS with minors for RB-21. PASS for F-31, F-32, F-33 and F-34 (carried, or given RB-21). No commit fails.
