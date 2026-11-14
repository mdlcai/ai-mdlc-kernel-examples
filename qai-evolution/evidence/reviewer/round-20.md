Commits reviewed: 1d076b8be7718cb7b783c287e1ed1331b0bdb64e (RB-20, Round 19 follow-ups: M-R19-1, N-R19-1, N-R19-2, N-R19-4, GD-0153), and again, given RB-20: 91d8e36 (F-31), f0da132 (F-32), 7931aff (F-33), e1fdd2e (F-34)
Previous round reviewed: round 19 (F-31 to F-34; F-33 FAIL on M-R19-1, others PASS / PASS with minors)

# Reviewer Gate: Review Round 20 (RB-20, the Round 19 follow-ups)

- **Mechanism:** a fresh-context subagent read `artifacts/reviewer/round-19.md` and `git show 1d076b8`. It checked the diff against SPEC.md §3 :161 (one queued or running protocol per tenant), §4 :119 (the requeue row), §5 F-31 :374 (the gate rule; "an accepted candidate becomes the checkpoint"), §5 F-32 :705 (random search) and :731 (the governance run), W12.1 at :862-866, and ADR-49 (DECISIONS.md :412, :432). It also read the surrounding code: services/jobs.py `submit`, `requeue`, `check_protocol_slot` and `enqueue`; repositories/jobs.py `lock_kind_slot`, `count_active_kind`, `get_job(lock=True)` and `owned_running`; every `with_for_update` and advisory-lock call site in qai_api; protocol.py `_run_once` and `_governance`; and ProtocolForm.tsx, form.ts and JobActions.tsx. It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider tests/test_protocol.py tests/test_evolution.py` gave **73 passed** in 141.31 s. That includes the two new tests and `test_defaults_with_seed_2026_pass_all_six`.
  - API: `cd services/api && python -m pytest -q -p no:cacheprovider tests/test_jobs_protocol.py tests/test_jobs.py` gave **55 passed** in 31.73 s (against the already-running qai-test-pg and qai-test-valkey).
  - Web: `npx vitest run app/app/protocol components/results/protocol-panels.test.tsx` gave **3 files, 37 tests passed**.
  - Harnesses in the session scratchpad (`r20/`), each under `python -I`:
    - `test_r20_race.py`: pytest from services/api with `-c pyproject.toml --rootdir .`, importing the `tests.conftest` fixtures. For 6 fresh tenants each, it fires three request pairs at once with `asyncio.gather`: (A) requeues of two different dead protocol jobs; (B) a protocol submit plus a requeue; (C) the same job requeued twice. `count_active_kind` is monkeypatched to sleep 50 ms, to widen the window between lock and count. A control test runs the same scenarios with `lock_kind_slot` stubbed out.
    - `gov.py`: checks `_random_config` against the pre-RB-20 inline draw (20 seeds × 50 draws). It then runs `_run_once` + `_governance` for seeds 1-10 at ε 0.002 and 0 (3 runs, 6 nodes, 3 candidates). For each case it compares the new selection rule with the old one, and the governance checkpoint with the run's adopted Ψ*.
    - `gov_none.py` and `gov_seed2.py`: governance when no run adopts. `gov_none.py` covers forced and natural cases at ε 0 and 0.002 (2 runs, 1 candidate, baseline `{p 2, maxiter 60}`). `gov_seed2.py` runs the full `run_protocol` on the natural seed-2 case at both ε values, and that case under the old rule.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual. The one fail is INV-2 (web baseline, the known carry). Nothing new fails.
  - `npm run lint:py`: "All checks passed!" and "163 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 0; pending review F-31, F-32, F-34 and RB-20; in rework none; blocked none.

  It did not run the full SDK, API or web suites, Playwright (W12 is unverified end to end), the web baseline or the smoke test, because host-to-Docker port forwarding is down. It did not check the React `<details>` toggle behaviour in a real browser (see N-R20-4). It did not touch Docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json, the local `.env` or any tasks/*.output file. The one background run (the SDK suite) wrote to a scratchpad log, which is what it read. It ran govern.mjs only as `queue`, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-20 | 1d076b8 Round 19 follow-ups | **PASS with minors** (0 blockers, 0 major, 2 minor, 2 notes) | M-R19-1 is fixed correctly and race-safely (harness: always exactly one admission, no deadlock), and the test covers the requeue path. N-R19-2 and N-R19-4 are fixed. N-R19-1 now follows SPEC :731, but it brings in a false governance fail at ε = 0 when no run adopts (N-R20-1). SPEC §4 does not list the new requeue 429 (N-R20-2). |
| F-31 | 91d8e36 SDK governance gate | **PASS** | RB-20 did not change it. The gate rule (SPEC :374) is what makes N-R20-1 happen, but the gate itself is correct: an equal J at ε = 0 is accepted, as SPEC says it should be. N-R19-6 is still a note. |
| F-32 | f0da132 qai.protocol | **PASS with minors** | N-R19-1 and N-R19-2 are closed against SPEC. The seed-2026 defaults still pass all six tests, and the draws are unchanged for the default baseline. RB-20's governance change brings in N-R20-1 (ε = 0 only, which is not the default) and N-R20-3 (a note). |
| F-33 | 7931aff protocol job kind | **PASS** (given RB-20) | M-R19-1 is fixed. `requeue` runs `check_protocol_slot` in the same `tenant_session` transaction, after the job row lock. There is no lock-ordering cycle with `submit`, the worker or T2. 429 QUEUE_FULL comes with `Retry-After: 60`, and the rollback leaves the job `dead`. N-R19-7 (retryable TIMEOUT) is still a note. |
| F-34 | e1fdd2e /app/protocol | **PASS with minors** (given RB-20) | N-R19-4 is fixed: an error in Advanced opens the section, and `shownOnForm` matches only paths that have a control. The N-R19-3 rationale is accepted, but SPEC W12.1 still describes the old order (carried). W12 was not run. |

## Check by check

| Check | Result | Evidence |
|---|---|---|
| M-R19-1: the cap is checked on requeue | PASS | services/jobs.py:322 takes the row lock (`get_job(..., lock=True)`), then :325-326 check existence and state, then :327-329 call `check_protocol_slot` when `job.kind == "protocol"`. All of it is inside the one `tenant_session` (:321), so the advisory lock (`pg_advisory_xact_lock`, repositories/jobs.py:50) is held until commit, and an ApiError rolls back. |
| M-R19-1: lock ordering and deadlock | PASS | `requeue` takes the row lock on job X, then advisory lock A. `submit` (:257-258) takes A but locks no existing job row: `enqueue` only inserts a job and an audit row. Two requeues of X and Y each lock their own row, then queue on A. Nothing else locks job rows and A together: `cancel` (:298) takes a row lock only; the worker's `owned_running` (repositories/jobs.py:164-176) and the claim (`skip_locked`, :128) take no advisory lock; T2 lives in scheduler/triggers.py. So no path holds A and then waits on a job row, and there is no cycle. Harness: 18 concurrent pairs with the lock gave 0 errors and no 500. |
| M-R19-1: race-safety | PASS (harness) | `test_r20_race.py` with the lock: (A) two requeues gave `[200, 429]` and 1 active job in 6 of 6 runs; (B) submit plus requeue gave `[202, 429]` or `[200, 429]`, 1 active, in 6 of 6; (C) the same job twice gave `[200, 409]`, 1 active, in 6 of 6. The control, with the lock stubbed out, gave `[200, 200]` with 2 active (A) and `[200, 202]` with 2 active (B) in 6 of 6. So the harness can see a race, and the lock is what closes it. Under READ COMMITTED, the count after the lock sees the committer's row. |
| M-R19-1: the new test | PASS | test_jobs_protocol.py:147-168: a dead protocol plus a live queued one makes requeue give 429 QUEUE_FULL with `retry-after: 60`, and the status stays `dead` (the rollback is checked). With the live job `succeeded`, requeue gives 200 `queued`. It runs through the real HTTP route. It is sequential, which the harness above covers. |
| M-R19-1: the UI handles the new 429 | PASS | JobActions.tsx:47-48 passes any problem to `onProblem(toProblem(e))`, so the SPEC §7 message ("already has an evidence protocol queued or running") is shown. |
| N-R19-1: the governance run | PASS against SPEC :731 | protocol.py:385-387 picks the first run with `record["adopted"]`, or run 1 with Ψ₀ if no run adopted. `gov.py` seeds 1-10: `governance.run` equals the first adopted run in every case. test_protocol.py:218-221 checks this on the `small` fixture. See N-R20-1 and N-R20-3. |
| N-R19-2: the random-search range | PASS against SPEC :705 | protocol.py:203-213: maxiter is drawn from `integers(10, min(500, 4 × baseline) + 1)`. `min_mi = min(10, max_mi)` is always 10, because the baseline maxiter is at least 10. test_protocol.py:202-215 sees min 10 and max 160 at baseline 40, and stays at or below 500 at baseline 200. |
| N-R19-2: draw order and determinism | PASS | The draw order is unchanged: p, then maxiter, then 2p angles, all from `rng_rand`, and then COBYLA's x0 from the same stream (protocol.py:290-299). `gov.py`: at baseline maxiter 10 (the default), the old and new draws are identical, and so is the end bit-generator state, for 20 seeds × 50 draws. So the seed-2026 defaults and W12 give byte-identical results. For a baseline maxiter above 10, the drawn values (and possibly how many raw bits `integers` uses up) change. The output is still a pure function of the seed, so `deterministic` and same-seed-same-evidence hold. |
| N-R19-4: Advanced opens on an error | PASS (jsdom) | ProtocolForm.tsx:94-95, 157-161: `open={advancedOpen \|\| advancedError}` with `onToggle → setAdvancedOpen(e.currentTarget.open)`. This is sound with React 19. When an error forces it open, the browser's toggle event sets `advancedOpen = true`, so the section stays open after the error is fixed and does not snap shut while the user types. A user's own open and close is kept. `inAdvanced` (form.ts:167-170) is `shownOnForm` minus `params.seed`, which matches the layout: only Project and Seed sit outside `<details>` (ProtocolForm.tsx:138-154). ProtocolForm.test.tsx: runs 1 opens it, and seed −1 does not. One leftover edge case is in N-R20-4. |
| N-R19-4: `shownOnForm` is narrower | PASS | form.ts:161-165 now ends `baseline\.p\|baseline\.maxiter)$`, so `params.baseline` and `params.baseline.initialParams` go to the SubmitBanner (ProtocolForm.tsx:37). Tested in form.test.ts:101-105. |
| N-R19-3 (kept) | Rationale accepted | A running protocol also takes the slot, so the UI-first order is only flaky if the first W12-sized job *finishes* before the second submit. With runs 2 and candidates 2, that is plausible on a fast worker. Posting the first job through the API, before the worker can drain it, is the deterministic way to meet a held slot. Both UI behaviours W12.1 needs (landing on the job page, and the 429 with its retry time) are still asserted through the UI. The leftover gap is the wording: SPEC :864-866 still describes the UI-first order (see Carried forward). |
| Regressions | 1 found | N-R20-1. Nothing else: the API, SDK and web suites pass, ruff and the invariant lint are unchanged (INV-2 only), and no other file changed. |

## Findings

- **N-R20-1 (minor, RB-20 / F-32): at ε = 0, if no run adopts, governance now reports fail. The "worse" injection is Ψ₀ scored against itself.**
  - Location: protocol.py:385-387 now falls back to Ψ₀ whenever no run *adopted*. Before RB-20 it fell back only when no run had a train-accepted checkpoint. When it falls back, `ck_cfg = chosen.psi0` (:393-395), and the "worse" candidate is `chosen.psi0` scored again (:409). These are the same configuration, seeds and shots, so J is equal. Under the gate rule (SPEC :374, J(S) ≤ J(Ψn) − ε), equal J is accepted at ε = 0. The "worse" injection is then accepted, `ok` turns false (:422), governance is `fail`, and the whole protocol outcome becomes `fail` (:595-607).
  - Scenario (`gov_seed2.py`): seed 2, nodes 6, runs 2, splits 1/1/1, candidates 1, decisionSeeds 1, evaluationSeeds 2, shots 64, baseline `{p 2, maxiter 60}`, epsilon 0. All of these are valid under SPEC :161, which allows epsilon from 0 to 0.1. Run 1 accepts a train candidate, but validation does not adopt it.
    - Before RB-20: governance tested against run 1's checkpoint, all three injections were rejected, and governance passed.
    - After RB-20: governance gives `fail` with `worse: accepted` (J 0.17825 = jCheckpoint 0.17825), and `run_protocol` gives **status fail**. The same params at ε 0.002 give inconclusive with governance pass.
    - The forced case (every run marked not adopted) gives the same split: pass at ε 0.002, fail at ε 0.
    - Natural cases with no train acceptance at all (seeds 3 and 10) already failed this way before RB-20. Round 19 recorded this as a known risk. RB-20 makes it happen more often: it now covers every run that accepts on train but not on validation, which is common.
  - Impact: a release that requires `protocol` evidence and was run at ε = 0 gets a false `fail` while the gate is working correctly. The default ε (0.002), W12 and the seed-2026 defaults are not affected.
  - The code follows SPEC :731-734 to the letter, so the fix is a SPEC change with an ADR. Options:
    - make the "worse" injection strictly worse than the checkpoint, for example the worst random-search configuration on train, or Ψ₀ with J raised by max(ε, a fixed margin);
    - or report governance as inconclusive ("no strictly worse candidate at ε = 0") when the checkpoint is Ψ₀ and ε = 0;
    - or restrict ε to above 0 for kind protocol.
    - Add a test at ε = 0 with no adoption.
- **N-R20-2 (minor, RB-20 / F-33): SPEC §4 does not list the new 429 on requeue.**
  - Location: SPEC.md:119, where the requeue row says "job (queued, attempts=0); 409 unless dead or failed". SPEC.md:161 speaks only of "a second submit". The behaviour is right and ADR-49 (DECISIONS.md:432) covers the intent, but the CLAUDE.md guardrail says to change the doc first. A client written against §4 would not expect 429 QUEUE_FULL with `Retry-After: 60` from `POST /jobs/{id}/requeue`.
  - Fix: add "; 429 QUEUE_FULL (Retry-After 60) for a protocol job when the tenant already holds one queued or running" to :119, and "a submit or requeue" to :161.
  - A related point that predates RB-20 and is not a regression: `requeue` does not call `check_backpressure`, for any kind (services/jobs.py:320-350). An admin can therefore requeue past MAX_QUEUED_PER_TENANT and the global depth. SPEC does not require that check, so this is noted only.
- **N-R20-3 (note, F-32): governance gates against the run's last accepted checkpoint, which is not always the adopted Ψ\*.**
  - protocol.py:372 stores `checkpoint=checkpoints[-1]`, the head of the train chain. Validation can adopt an earlier checkpoint (`sel_i < len − 1`, :278-284). `gov.py`: the governance checkpoint differs from the adopted Ψ* for seeds 5 and 10 at ε 0.002, and for seeds 5, 7 and 10 at ε 0, out of 10.
  - This agrees with SPEC :374 ("an accepted candidate becomes the checkpoint") and with the result text ("the last accepted state is unchanged", :445). So "that run's checkpoint" (:732) is read as the store's state, not Ψ*. That reading is acceptable.
  - The new test (test_protocol.py:218-221) checks only `governance.run`, not which checkpoint was used. If Ψ* was intended, change :387 to `checkpoints[sel_i]` (which would need `sel_i` on `_RunOutcome`). Otherwise, say "the run's last accepted checkpoint" at SPEC :732.
- **N-R20-4 (note, RB-20 / F-34): if the user collapses Advanced while it is forced open, a later Advanced error does not reopen it.**
  - ProtocolForm.tsx:159-160. With an Advanced error present, the `open` prop is `true`. If the user collapses the section, `onToggle` sets `advancedOpen = false`, but the prop stays `true`. React compares the old and new props, not the DOM, so it writes nothing, and the DOM stays closed. A second Advanced error, added while the first is still there, leaves the prop at `true`, so the section does not reopen. Nothing re-syncs until every Advanced error clears.
  - This only happens after the user has explicitly collapsed the section. The Run button is disabled, and the errors are on the fields. This was found by reading the code and was not checked in a browser: jsdom does not fire the native `toggle` event the same way.
  - Possible fix: key the forced open on the *set* of Advanced error paths, for example by calling `setAdvancedOpen(true)` from an effect whenever that set grows.

Counts: 0 blockers, 0 major, 2 minor, 2 notes.

## Carried forward (human action)

- **Queue:** F-31, F-32, F-34 and RB-20 are pending review, and this round is their review evidence. F-33 has no FAIL now. M-R19-1 is fixed by RB-20, which is in the pending list. The minors N-R20-1 (a SPEC and ADR decision on the ε = 0 governance injection) and N-R20-2 (the SPEC §4 :119 wording) can go in a later RB commit and do not block.
- **Round 19 items still open:** N-R19-3 (keep the e2e order, but reword SPEC W12.1 :864-866 to say "with a protocol already queued for the workspace, a UI submit shows the 429 retry time"). Notes N-R19-5, -6, -7 and -8 are unchanged.
- **INV-2:** still fails (web baseline stale; covers 2 of 29 screens). It needs a quiet-host baseline run or an explicit human acceptance.
- **W12 and smoke:** not run, because host-to-Docker port forwarding is down. Run `npm run e2e` (W12) and `./smoke-test.sh` before release 0.8.0.

## Overall

- RB-20 closes the Round 19 major. An admin requeue of a protocol job now takes the same advisory lock, inside the same transaction, as `submit`. A concurrency harness shows exactly one admission in every interleaving, with no deadlock, and the control without the lock shows the double admission it prevents.
- The SDK changes match SPEC §5 F-32 as written, and the default and W12 results are byte-identical.
- The governance selection, now as SPEC words it, exposes a SPEC flaw: at ε = 0 with no adoption, the "worse" injection is not actually worse. That case needs a SPEC and ADR decision.
- The web fix is sound.

Verdict: PASS for F-31 and F-33. PASS with minors for RB-20, F-32 and F-34. No commit fails.
