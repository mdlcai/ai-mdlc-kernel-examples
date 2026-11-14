Commits reviewed: 91d8e36 (F-31, SDK governance gate with evidence completeness, GD-0148), f0da132 (F-32, qai.protocol six-test evidence protocol, GD-0149), 7931aff662c23a7cb82591c5b7191ff8f6351664 (F-33, protocol job kind, worker runner, migration 0013, PROTOCOL_TIMEOUT_SEC, GD-0150), e1fdd2e59b853064f93a5bddd4e2422a73429502 (F-34, /app/protocol screen, six-test panels, BoxPlot, e2e W12, GD-0151)
Previous round reviewed: round 18 (RB-18 b4491d4 + RB-19 87f9737, PASS)

# Reviewer Gate: Review Round 19 (F-31 to F-34, the Evidence Protocol)

- **Mechanism:** a fresh-context subagent read `git show` for the four commits against SPEC.md §3 (the protocol params and result shape, "a tenant may hold one queued or running protocol job" at :161), §5 F-31 to F-34 (random search at :705, the governance run at :731, the 429 AC at :765, the ACs at :788), §6 (Evidence Protocol row at :902), §7 codes, W12 at :862-873, ARCHITECTURE.md §9 and DECISIONS.md ADR-49. It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider --durations=8` gave **524 passed, 1 skipped** in 220.48 s. The slowest test is `test_defaults_with_seed_2026_pass_all_six` at 143.31 s; its `slow` marker is declared but not deselected, so it runs in the default suite.
  - API: `cd services/api && python -m pytest -q -p no:cacheprovider tests/test_jobs_protocol.py tests/test_migrations.py` gave **30 passed** in 74.06 s (against the already-running qai-test-pg and qai-test-valkey).
  - Web: `npx vitest run components/results/protocol-panels.test.tsx app/app/protocol components/shell` gave **3 files, 48 tests passed**.
  - Harnesses in the session scratchpad (`r19/`), each under `python -I` against the working tree (`packages/` clean):
    - `cobyla.py` and `cobyla2.py`: `qai.protocol._cobyla` budget accounting over p 1-5, budgets 2-9 and five objective shapes (240 cases).
    - `gov.py`: `run_protocol` with 3 runs, 6 nodes, 3 candidates, seeds 1-6, at epsilon 0.002 and 0. It records the governance run, the first adopted run, the first run with a train-accepted candidate and the injection decisions.
    - `gov2.py`: `_run_once` over 4 runs per seed, baselines `{p 2, maxiter 60}` and `{p 1, maxiter 40}`, comparing "first run with a checkpoint" against "first run that adopted".
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual. INV-2 fails: "8 unaccepted failure(s) ... report covers 2 screens, SPEC UI Surface lists 29 ... source changed since gitSha 3f35306 (75 file(s) ...): stale". This is the same carried-forward INV-2 failure, now with `/app/protocol` added to the UI surface.
  - `npm run lint:py`: "All checks passed!" and "163 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 0; pending review F-31, F-32 and F-34; nothing in rework or blocked. F-33 is not listed as pending.

  It did not run the full API suite, Playwright (so W12 is unverified end to end), the web baseline or the smoke test, because host-to-Docker port forwarding is down. It did not touch Docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json, the local `.env` or any tasks/*.output file. When a background test run finished, it re-ran the command in the foreground or logged to the scratchpad instead of opening the task output. It ran govern.mjs only as `queue`, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-31 | 91d8e36 SDK governance gate | **PASS** (0 blockers, 0 major, 0 minor, 1 note) | The gate rejects in the order SPEC gives: missing evidence, then V, then J against J(Ψn) − ε through `decide`. Every rejection reason is tested, and `decide` is unchanged. N-R19-6: the completeness check counts records but does not check which pairs they are. |
| F-32 | f0da132 qai.protocol | **PASS with minors** (0 blockers, 0 major, 2 minor, 3 notes) | The six tests, the bootstrap CIs, test-once, the seeded streams, determinism and the evidence all behave as the docs say, and the defaults with seed 2026 pass all six. Two SPEC deviations: which run governance tests (N-R19-1) and the random-search maxiter range (N-R19-2). |
| F-33 | 7931aff protocol job kind | **FAIL: rework needed** (0 blockers, **1 major**, 0 minor, 1 note) | Params validation, the §7 codes, RLS, cross-tenant 404, the race-safe 429, migration 0013 up and down, and the timeout selection are all correct. M-R19-1: an admin requeue gets past the one-protocol-per-tenant cap. |
| F-34 | e1fdd2e /app/protocol | **PASS with minors** (0 blockers, 0 major, 2 minor, 3 notes) | The parser handles hostile input safely, every chart has a table alternative, the panels follow SPEC order, and INV-11 holds. The e2e test reverses the W12.1 order (N-R19-3), and errors on fields inside the collapsed Advanced section are hidden (N-R19-4). W12 was not run. |

## Check by check

| Check | Result | Evidence |
|---|---|---|
| F-31 gate order and reasons | PASS | evolution.py:279-305. Missing evidence (wrong count, `result is None`, duplicate pair, no reproducible record) comes first, then `_checks` (normalisation, probabilitySum, countsSum, historyFinite, ratioInRange, reproducibility), then `decide(True, j, j_baseline, eps)`. Ties: equal J is accepted at ε = 0 and rejected at ε = 1e-9 (test_evolution.py). |
| F-32 test-once | PASS | `_TestSet.evaluate` raises `ProtocolIntegrityError` on a second call (protocol.py:170-182). The "twice" hook test raises. |
| F-32 seeds and determinism | PASS | `SeedSequence(seed, spawn_key=(run, k))` streams (protocol.py:163), the stats stream uses spawn_key (31,), and the seed sets are disjoint by construction (protocol.py:149). The same params give the same result and evidence; a different seed gives a different result. |
| F-32 outcome rules | PASS | An integrity failure gives fail. Runs < 30, or an improvement CI that does not clear 0, gives inconclusive, with the reason "inconclusive: repeatability (2 of 30 runs)". |
| F-32 equal budget | PASS for the real objective | B = candidates + 1 for evolved, random and COBYLA. `cobyla.py`: `_cobyla` spends exactly `budget` evaluations in 238 of 240 cases. The two exceptions (p 1, budget 8 or 9, a constant objective) stop at 7 because COBYLA converges. In that case the Baselines budget check would report fail, not pass silently. See N-R19-5. |
| F-32 governance injections at ε = 0 | PASS on the harness seeds | `gov.py`: at both ε 0.002 and ε 0, for seeds 1-6, worse, invalid and missing are all rejected and governance passes. The brief's known risk (the worse injection accepted at ε = 0 when nothing was adopted) did not occur on these seeds. The code reports it honestly if it does: the state sha changes and governance fails (protocol.py:373-436). |
| F-33 params and codes | PASS | schemas/jobs.py: StrictInt bounds, `_bounded` floats, unknown keys give `UNKNOWN_FIELD` at `params.<key>` (including `params.baseline.j`), circuit gives `UNSUPPORTED`, `INVALID` at `params.baseline.<path>`. All codes are in §7. |
| F-33 one protocol per tenant: concurrent submits | PASS | services/jobs.py:258 calls `check_protocol_slot` (:134-146) inside the submit `tenant_session` transaction, after backpressure and before enqueue. repositories/jobs.py `lock_kind_slot` takes `pg_advisory_xact_lock(hashtext('jobs:protocol:<tenant>'))` before `count_active_kind`. Under READ COMMITTED (no override in qai_api), the second submitter waits for the first commit and then sees its row. The test is sequential only. |
| F-33 one protocol per tenant: requeue | **FAIL** | M-R19-1. |
| F-33 RLS and 404 | PASS | Every path goes through `tenant_session`. A cross-tenant job id gives 404 (tested). Viewer gets 403 (RBAC, tested). |
| F-33 migration 0013 | PASS | It recreates `ck_jobs_kind`, `ck_evidence_records_kind` and `ck_release_candidates_required_kinds` (cardinality 1..11). The downgrade refuses with a DO block while protocol rows exist. Up, refuse, down and re-up are tested. |
| F-33 worker timeout | PASS | worker/loop.py:210-211 uses `protocol_timeout_sec` for kind protocol and `job_timeout_sec` otherwise. `maxSeconds` can only lower it, and RLIMIT_CPU is `int(timeout) + 1`. Config allows 60-3600, default 900, and `.env.example` and compose carry it. N-R19-7 covers retries. |
| F-34 parser | PASS | Unknown statuses map to `fail` (protocol-panels.tsx:117). Rows are capped at 64 and lists at 8, and non-finite numbers are dropped. The hostile-input test passes. |
| F-34 accessibility and styling | PASS | BoxPlot and Histogram each have a table alternative in `div.table-wrap role=region`, the figure is labelled by its caption, and `StatusPill` shows text as well as colour. Styles use tokens from protocol.css. INV-10 and INV-11 pass. |
| F-34 W12.2 and W12.3 assertions | PASS (static) | w12-protocol.spec.ts checks six rows, four passes, "2 of 30 runs", inconclusive, three rejected injections with sha before = after, 2 lock rows with testEvaluations 1, and evidence listed and verified. These rely on the default ε 0.002. Not run. |

## Findings

- **M-R19-1 (major, F-33): an admin requeue gets past the one-protocol-per-tenant cap.**
  - Location: services/api/qai_api/services/jobs.py:320-350 (`requeue`) never calls `check_protocol_slot` (:134-146). Only `submit` calls it (:258). This breaks SPEC §3 :161 ("A tenant may hold one queued or running protocol job") and the reasoning in ADR-49, where this cap is what stops one tenant holding both worker replicas.
  - Scenario: protocol A runs out of time three times and goes `dead`. The tenant submits protocol B, which is queued. An admin then requeues A (`POST /api/v1/jobs/{A}/requeue`). `requeue` puts A back in `queued` with `attempts=0`. The tenant now has two active protocol jobs, and both worker replicas can be held for up to `PROTOCOL_TIMEOUT_SEC` (900 s) each, three attempts each. Other tenants wait. Requeueing several failed protocol jobs in a row makes it worse. Schedules only create benchmark jobs (repositories/schedules.py:65), so they are not a second route around the cap.
  - Fix: in `requeue`, when `job.kind == "protocol"`, call `check_protocol_slot(db, principal.tenant_id)` inside the same `tenant_session`, after the row lock and before `set_fields`, and return 429 QUEUE_FULL with Retry-After. Add an integration test where requeueing a dead protocol job while another is queued returns 429, and one where it is allowed when the slot is free.
- **N-R19-1 (minor, F-32): governance runs on the first run with a train-accepted checkpoint, not "the first run that adopted a checkpoint".**
  - Location: protocol.py:374 `chosen = next((o for o in outcomes if o.checkpoint is not None), outcomes[0])`, with `checkpoint` set at :362 from the train gate's accepted checkpoints. SPEC :731 says "the first run that adopted a checkpoint", where adopted means validation chose Ψ* ≠ Ψ₀ (the record's `adopted`, :333).
  - Scenario (`gov2.py`): seed 2, baseline `{p 2, maxiter 60}`, 4 runs. Run 1 accepts a train candidate, but validation does not adopt it (`adopted: false`). Run 2 is the first to adopt. Governance tests run 1's checkpoint, while SPEC calls for run 2's. The gate is still exercised on a real accepted state, so the test stays meaningful, but the reported `governance.run` differs from the documented rule. Seeds 1 and 3-5 agree.
  - Fix: pick `next((o for o in outcomes if o.record["adopted"]), ...)` and use its adopted checkpoint, or change SPEC :731 and add an ADR.
- **N-R19-2 (minor, F-32): the random-search maxiter range starts at the baseline's maxiter, not 10.**
  - Location: protocol.py:276-280 draws `rng_rand.integers(psi0.maxiter, max_mi + 1)`. SPEC :705 says "maxiter from 10 to 4 × the baseline's".
  - Scenario: with baseline maxiter 60, random search draws from 60 to 240 instead of 10 to 240. The defaults (maxiter 10) are unaffected.
  - Fix: use `integers(10, max_mi + 1)`, which is MIN_MAXITER, or update SPEC.
- **N-R19-3 (minor, F-34): the e2e test reverses the W12.1 order.**
  - Location: apps/web/e2e/w12-protocol.spec.ts:60-83. The first protocol is posted through the API (:61). The UI's first Run (:67) is the one that gets the 429 (:70), and the second Run (:83) lands on the job page.
  - SPEC :864-866 says the UI submit lands on the job page, and then a second submit while the first is queued shows the 429. Both behaviours are exercised through the UI, but "a second submit while **the first** is queued" is not what is tested: the queued job is one the browser never submitted. Either reorder the steps (UI submit lands on the job page, go back, submit again, get the 429) or reword W12.1.
- **N-R19-4 (minor, F-34): errors on fields inside the collapsed Advanced section are hidden.**
  - Location: ProtocolForm.tsx:153-154 (`<details className="protocol-advanced">`, no `open` when an Advanced field has an error) and :213 ("Fix the highlighted values").
  - Scenario: a user types runs 0, collapses Advanced, or gets a server 400 on `params.candidates`. Run is disabled, and the side panel says "Fix the highlighted values", but nothing visible is highlighted. A related gap: `shownOnForm` (form.ts:161-165) treats every `params.baseline.*` path as shown on the form, but only `p` and `maxiter` have controls. A server error on `params.baseline.initialParams` would appear neither on a field nor in the banner. The form never sends that key, so this is unlikely in practice.
  - Fix: open the `<details>` when any Advanced field has an error, and narrow `shownOnForm` to `baseline\.(p|maxiter)`.
- **N-R19-5 (note, F-32): two checks are true by construction, and the determinism check evaluates run 1's test set again.**
  - `lockedBeforeTest` compares the constants `step = {"lock": 1, "test": None}` and `step["test"] = 2` (protocol.py:297-307), and the sha is the digest of the same object. It can only fail under the drop-lock hook.
  - The random entry in `budget_ok` is the constant `budget` (:353), so the Baselines budget check (:499-500) cannot fail for random.
  - `deterministic = _run_once(prm, 0).record == outcomes[0].record` (:446) scores run 1's test graphs a second time, on a fresh `_TestSet`. The result is not used for selection, so it does not break test-once, but it goes against the spirit of evaluating the test set once.
  - `_cobyla` can spend fewer than `budget` evaluations on a flat objective (`cobyla.py`: p 1, budget 8 or 9). Baselines then fails rather than passing silently, so this is safe.
- **N-R19-6 (note, F-31): the completeness check counts records but does not check which pairs they are.** evolution.py:279-295 rejects a wrong count, a None result and a duplicate pair. It does not check that the pairs are exactly the expected instance × seed set, so a record for an unexpected seed in place of an expected one would pass the completeness check. `run_protocol` always passes the records `evaluate_set` produced, so this cannot happen today. "A NaN probability" is modelled through `probability_sum`, because `QaoaResult` keeps no probability vector.
- **N-R19-7 (note, F-33): TIMEOUT is retryable, so a protocol that times out holds a worker for up to 3 × PROTOCOL_TIMEOUT_SEC.** worker/loop.py:105-107 marks TIMEOUT `retryable: True`, and JOB_MAX_ATTEMPTS is 3. The protocol is deterministic, so a retry with the same params times out again: 45 minutes of one replica at the default 900 s. Consider making TIMEOUT non-retryable for kind protocol.
- **N-R19-8 (note, F-34): smaller UI points.**
  - `qaoaRuns` (form.ts:128-137) counts validation as `candidates × validation × decisionSeeds`, but the SDK scores every checkpoint on the evaluation seeds. It also leaves out the reproducibility re-runs and the governance re-scoring, so "expected time" is low.
  - The Inconclusive status is not in the StatusPill map (components/ui/Badge.tsx:42), so it shows a neutral pill with a clock icon. The text says Inconclusive, so it does not rely on colour alone.
  - React keys use controls by name, injections by kind and comparisons by versus. Duplicate values in a hostile result only cause key warnings.

Counts: 0 blockers, 1 major, 4 minor, 4 notes.

## Carried forward (human action)

- **Queue:** F-31, F-32 and F-34 are pending review. This round is their review evidence. F-33 is not in the pending list. It needs M-R19-1 fixed (an RB commit) before it is approved.
- **INV-2:** still fails, with 8 unaccepted failures. The report is stale for 75 files and covers 2 of 29 screens, now without `/app/protocol` or `/app/formula`. It needs a quiet-host baseline run or an explicit human acceptance.
- **W12 and smoke:** not run, because host-to-Docker port forwarding is down. Run `npm run e2e` (W12) and `./smoke-test.sh` once the stack is reachable, before the release ships.

## Overall

- The SDK side (F-31 and F-32) is sound. The gate rejects in the documented order. The protocol is deterministic, evaluates each test set once, compares methods at equal budget, and turns its six tests into an honest outcome. The defaults with seed 2026 pass all six. The two SPEC deviations are small, and each can be settled by a one-line change to the code or to SPEC.
- The API (F-33) submit path is correct and race-safe, but the requeue path has no cap check, which defeats the one-protocol-per-tenant limit that ADR-49 relies on.
- The web screen (F-34) is robust and accessible. Its e2e test checks everything W12 lists but runs the W12.1 steps in a different order.

Verdict: FAIL for F-33 until M-R19-1 is fixed. PASS for F-31. PASS with minors for F-32 and F-34; their minors can go in the same rework commit or a later round.
