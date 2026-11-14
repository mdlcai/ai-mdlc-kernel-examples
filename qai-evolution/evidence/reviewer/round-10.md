Commits reviewed: cd1bffb9f95b8e2a6ecb8b78e027141fd1bb0775 (F-21), 236061482fb01705d8e83464361a1a7edc9ff9ac (F-21), 18633ba0d809bc008a0858b46acbf9a1f80b7753 (F-22), bb22a299df2fe6804417f83c3564817d86dcbed5 (F-23)
Previous round reviewed: 2c772ef (round 9, RB-8)

# Reviewer Gate: Review Round 10 (F-21, F-22, F-23: AI Simulation, Re-Entry R3)

- **Mechanism:** a fresh-context subagent reviewed each commit on its own with `git show <sha>`. It checked them against SPEC.md §3 (surrogate params and result), §5 F-21 to F-23, §7, W9 and the §6 AI Simulation row, ARCHITECTURE.md §2.8 and §9, DECISIONS.md ADR-41, and the CLAUDE.md conventions. It ran these:
  - SDK tests (`tests/test_surrogate.py`).
  - API tests (`tests/test_jobs_surrogate.py`, `tests/test_migrations.py`) against the running qai-test-pg and qai-test-valkey containers.
  - Web checks: `npx vitest run app/app/surrogate components/results/surrogate-panels.test.tsx`, `npx tsc --noEmit`, and `npx eslint --max-warnings 0` on the new web files.
  - `npm run lint:invariants`.
  - Three throwaway probes in the session scratchpad, outside the repo:
    - the worst case on one thread (`*_NUM_THREADS=1`, as the worker child sets): 10 qubits, 2,000 ops, 6 angles, 4,096 samples, 65,536 candidates, top-k 10;
    - the SPEC §9 performance row (4 qubits, 6 angles, defaults);
    - surrogate accuracy at spans ±π, ±0.25 and ±0.01, with tolerances 0.01 and 1e-6;
    - plus a direct call to `validate_params` to confirm the pydantic error codes, and a timing of the submit-time circuit parse.

  It did not run Playwright (`npm run e2e` needs the live stack and was not in the permitted commands). It did not touch docker, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-21 | cd1bffb SDK AI simulation (qai.surrogate) | **PASS** (1 minor, 1 note) | The basis, the 3^d features, `determined`, the hold-out split, the seeded design and the four checks all match §5 F-21. The fit is exact at span ±π, but not "to floating point" at smaller spans (M-R10-1). |
| F-21 | 2360614 Batch the exact stage | **PASS** (1 note) | The fused segment is mathematically correct (it applies U^T as a row-vector product). A hypothesis test checks it against `Statevector` on both the fused and the direct path. The worst case measured 54.9 s on one thread, against a 120 s timeout. |
| F-22 | 18633ba Surrogate jobs and evidence | **PASS** (1 minor, 3 notes) | Tenancy, RBAC, the worker runner, evidence hashing and verify, migration 0010 up/down and the release kinds are correct and tested. A list of `parameters` that is too long or empty returns TOO_LONG/TOO_SHORT, not the §3 codes (M-R10-2). |
| F-23 | bb22a29 AI Simulation screen, panels, W9 e2e | **PASS** (1 minor, 3 notes) | Client limits mirror the schema exactly. INV-11 is clean, the CSS uses tokens only, the panels are in spec order and the histogram has a table alternative. A `circuitId` server error does not land on the circuit field (M-R10-3). |

## F-21 (cd1bffb, 2360614): qai.surrogate

| Check | Result | Evidence |
|---|---|---|
| Basis and feature count | PASS | `fourier_basis` (surrogate.py:181-188) builds the tensor product of {1, cos x_j, sin x_j}. Its width is 3^d, and `FourierSurrogate.features = 3**d` (surrogate.py:200). The physics is right: rx/ry/rz/p/cp and each angle of u enter as e^{±iθ/2} or e^{iθ}·projector, so each probability is a degree-1 trigonometric polynomial per angle. The hypothesis test (test_surrogate.py:160-171, 40 examples, 1–4 qubits, 1–3 angles, including cp and u) gets held-out TVD ≤ 1e-6. |
| `determined` iff train > 3^d | PASS | `train_count > features` (surrogate.py:209-210), with `train_count = len(xs[holdout:]) = samples - holdout` (surrogate.py:288). |
| Hold-out = max(1, samples//5), held out first | PASS | `holdout = max(1, samples // 5)` (surrogate.py:265). The model trains on `xs[holdout:]` and the TVD is taken on `xs[:holdout]` (surrogate.py:288, 298). The bell AC test asserts 64 → 52/12 (test_surrogate.py:181-187). |
| Default samples max(64, 2·3^d) ≤ 4096 | PASS | surrogate.py:251-252. d = 6 gives 1,458 (test_surrogate.py:221). |
| Limits and paths | PASS | `_int`/`_num` reject bools, NaN and out-of-range values with the parameter's name as path (surrogate.py:48-62, 249-257). `numQubits` > 10 is rejected (surrogate.py:246-247). 16 parametrised limit cases plus the qubit cap pass (test_surrogate.py:240-268). |
| Slots and errors | PASS | The `parameters` / `parameters[i]` paths and messages are exactly §5 F-21 (surrogate.py:65-91). A u gate gives 3 slots, and an index listed twice or not parametrised gives `parameters[i]`. |
| Determinism by seed | PASS | There is one `PCG64(seed)` generator. It draws the design hypercube, then the candidate hypercube (surrogate.py:260, 264, 302). Ranking is `lexsort` by −score, with ties broken by index (surrogate.py:340). The test asserts identical stages without timings, an identical evidence sha256, `verify` true, and that seed 12 gives a different result (test_surrogate.py:224-237). My probe of the 4-qubit/6-angle case ran twice and gave identical `top` and `checks`. |
| Evidence excludes timings | PASS | `evidence_parts` outputs carry status, target, checks, parameters, hold-out TVDs, baseline, top and best. They carry no `ms`, `msPer*` or `speedup` (surrogate.py:383-402). `surrogate` is added to `EVIDENCE_KINDS` (evidence.py:37-46). |
| Checks | PASS | The four names and order match §5. The tolerances are `NORM_TOLERANCE = 1e-10` and the user tolerance. `no_regression` is `baseline − best ≤ tol`, the same as "best ≥ baseline − tol" (surrogate.py:323-358). |
| Batched exact stage (2360614) | PASS | The fused matrix has row b = U e_b, so it is U^T, and `psi_row @ U^T = (U psi)^T` is correct (surrogate.py:150-155, 167-168). The per-sample slot gates use einsum over the batch (surrogate.py:116-126). `measure` and `barrier` are skipped, as `Statevector` skips them. `bind` bypasses `append` validation, so a value near the 1e6 cap plus the span cannot raise. A hypothesis test (30 examples) covers u, cp, rz, barrier, measure and ccx/swap, on the fused path (batch 40 > 2^3) and the direct path (batch 8), to atol 1e-12 (test_surrogate.py:100-111). |
| Boundedness | PASS | The worst case, on one BLAS thread as the worker child sets (loop.py:68-70), measured **54.9 s** in total, 52.7 s of it the exact stage. The commit reports 47.7 s. This is under JOB_TIMEOUT_SEC 120 and RLIMIT_CPU 121. Peak arrays are about 6×16 MiB of fused unitaries plus 32 MiB of probabilities, well under JOB_MEMORY_MB 2048. The §9 row (4 qubits, 6 angles, defaults) measured 0.22 s against a budget of < 5 s. |
| Tests | PASS | `test_surrogate.py`: 33 passed in 2.0 s. |

## F-22 (18633ba): surrogate jobs, evidence, migration 0010

| Check | Result | Evidence |
|---|---|---|
| Params contract | PASS | `SurrogateParams` (schemas/jobs.py:146-165) checks: `seed` 0..2^32−1; `samples` 16–4096 or null; `candidates` 64–65536 (default 4096); `topK` 1–10 (default 5). `span` and `tolerance` go through `_bounded`, so strings, bools, null and 10^400 are INVALID or OUT_OF_RANGE, never an overflow. `extra="forbid"` is inherited. The test covers 19 paths (test_jobs_surrogate.py:62-101). |
| Circuit-dependent rules use the SDK | PASS (see M-R10-2) | `check_circuit_params` (schemas/jobs.py:171-206) returns `circuitId` TOO_MANY_QUBITS for > 10 qubits. It calls `parameter_slots` and maps the result to `params.parameters` NO_PARAMETERS / TOO_MANY_PARAMETERS and `params.parameters[i]` NOT_PARAMETRISED / DUPLICATE. A target of the wrong length gives `params.target` INVALID. It is tested end to end (test_jobs_surrogate.py:104-127). |
| §7 contract | PASS | The top-level code is VALIDATION_FAILED (422), and field codes reuse existing ones (DUPLICATE is already used at schemas/jobs.py:78 and :111). No new top-level code is introduced. |
| Tenancy and RBAC | PASS | The circuit is loaded inside the existing `tenant_session` block of `submit` before `check_circuit_params` (services/jobs.py:224-228). Another tenant's circuit returns 404 NOT_FOUND and a viewer gets 403 FORBIDDEN (test_jobs_surrogate.py:130-139). The S5 cost uses the circuit's numQubits through `CIRCUIT_KINDS` (schemas/jobs.py:39). |
| Worker runner | PASS | `run_surrogate` (runners.py:327-358) runs in the isolated child and passes normalised params. A `CircuitError` becomes an `InputError`. The evidence status follows `r["status"]`, and `metrics` keeps only the stages that have `ms`. The underdetermined run is still a succeeded job with fail evidence (test_jobs_surrogate.py:173-184). |
| Evidence hashing and verify | PASS | The inputs are `{circuit snapshot, normalised params}`, and the outputs/summary match §5 F-22 exactly (surrogate.py:383-412). GET /evidence/{id} returns `verified: true`. The record appears in the list and the JSON export filtered by `kind=surrogate` (test_jobs_surrogate.py:160-170). |
| Migration 0010 | PASS | The three CHECKs are dropped and recreated under their existing names, and the cardinality rises to 8 (0010_surrogate.py:36-45). The downgrade guard RAISEs while `surrogate` jobs, evidence or required kinds exist (0010_surrogate.py:52-64), following the 0009 pattern. No table uses FORCE ROW LEVEL SECURITY (grep over migrations/versions finds none), so the owner sees every row. Even if it could not, re-adding the old CHECK would fail closed. The test covers upgrade, refusal with a surrogate job, a clean downgrade and re-upgrade (test_migrations.py:116-155). It also pins the 0009 test at revision 0009. |
| Release kinds | PASS | `requiredKinds` accepts `surrogate` with max_length 8 (releases.py:26, 34), and the export `Kind` includes it (export_evidence.py:32). Eight kinds are accepted and nine are rejected (test_jobs_surrogate.py:187-202). |
| Smoke | PASS (static) | smoke-test.mjs:397-432 submits the AC ansatz and asserts 4/4 checks, `fourier-ridge`, one evidence id, `kind` and `verified`. It was not run in this round. |
| Tests | PASS | `test_jobs_surrogate.py` plus `test_migrations.py`: **10 passed in 42.1 s** against the test containers. |

## F-23 (bb22a29): AI Simulation screen and result panels

| Check | Result | Evidence |
|---|---|---|
| Client limits match schemas/jobs.py | PASS | form.ts:15-21 matches SurrogateParams: MAX_ANGLES 6, MAX_QUBITS 10, samples 16–4096, candidates 64–65536 (default 4096), topK 1–10 (default 5), tolerance 1e-6–0.5 (default 0.01). Seed is 0..SEED_MAX. The spans are π, π/2 and π/4, all inside [0.01, π]. `defaultSamples`, `trainCount` and `determined` reproduce the SDK formulas (form.ts:68-77). Vitest checks both boundaries of every limit (form.test.ts:76-96). |
| Form contents (§5 F-23) | PASS | It has project and circuit selects (SurrogateView.tsx:167-194). The checkboxes show gate, qubits and current angle, and the first ops that fit in 6 angles are ticked (form.ts:51-60; SurrogateForm.tsx:113-131). There are seed, span, target and an Advanced fieldset. The live d and 3^d are shown with the underdetermined warning (SurrogateForm.tsx:250-284). |
| Server errors map to fields | PARTIAL (M-R10-3) | `params.*` errors land on their controls through `fieldMessage`, including `params.parameters[i]` through the child-prefix match (pipeline/_lib/form.ts:121-127). Unmapped errors go to the banner (SurrogateForm.tsx:36). |
| Panels in spec order | PASS | Design and training, then Accuracy, then Exploration (top-k table plus `Histogram`, which renders a table alternative at Histogram.tsx:110-126), then Validation (status pills, overall status, evidence link) (surrogate-panels.tsx:427-442). The job page routes `surrogate` to them and suppresses the generic result block (algorithm-slot.tsx:30-33; JobDetailView diff). |
| INV-11 | PASS | There is no `dangerouslySetInnerHTML`, `innerHTML` or `style=` in app/app/surrogate or surrogate-panels.tsx. `parseSurrogateResult` coerces every field, and keys of the distribution must match `^[01]{1,16}$` (surrogate-panels.tsx:76-88). Vitest's "renders API strings as text" passes. |
| components/ui rule and tokens | PASS | The only raw element is the unstyled `<input type="checkbox">` (SurrogateForm.tsx:118-122), the same precedent as SimulatorView, GovernanceView and RunValidation. surrogate.css uses only `var(--space-*)`, `--color-muted` and `--color-warning`, which exist at globals.css:19, :33 and :89. It has no hex or px literals. |
| Navigation and kind labels | PASS | `/app/surrogate` sits after `/app/pipeline` in the Execution group (nav.ts). The KIND_LABEL entries for jobs, validation and governance and the `EVIDENCE_KINDS` lists include `surrogate`. Their tests are updated. |
| Accessibility | PASS (1 note) | The angle fieldset has a legend and `aria-describedby` hint/error, and errors use `role="alert"`. Each checkbox is wrapped by its label. Tables have `scope` headers, and check rows use `<th scope="row">`. `StatusPill` is a `<span>`, so it nests validly in a `<p>` (Badge.tsx:46). |
| e2e W9 | PASS (static; 1 note) | w9-surrogate.spec.ts covers each step: W9.1 nav link, selection, defaults, disabled submit and client limit (:44-92); W9.2 model, features and determined (:96-101); W9.3 max TVD < 1e-6 (:104-107); W9.4 five top rows with \|pred − exact\| < 1e-5, best ≥ saved, histogram (:110-122); W9.5 four pass checks, status pass, evidence listed on /app/validation and sha256 verified against the API (:125-152). The spec was not executed in this round. |
| Lint / types / unit tests | PASS | Vitest: 2 files, 11 tests passed. `npx tsc --noEmit` exits 0. eslint `--max-warnings 0` on the new files exits 0. `npm run lint:invariants`: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (`missing FRONTEND-AUDIT.json`), the known pre-existing state. |

## Findings

- **M-R10-1 (minor, F-21): "exact to floating point once determined" holds only at span ±π.** The claim appears in DECISIONS.md:310 (ADR-41), packages/qai-sdk/qai/surrogate.py:3-6 and the web note at apps/web/app/app/surrogate/SurrogateForm.tsx:285-289.
  - **Cause:** for a small span, the basis {1, cos θ, sin θ} on [θ0 − s, θ0 + s] is nearly collinear. The Gram matrix's smallest eigenvalues fall far below `RIDGE = 1e-9` (surrogate.py:31, 216), so the ridge term dominates and the fit is no longer exact.
  - **Measured (3 qubits, seed 7, `determined: true` in every case):**

    | d | Span | Hold-out max TVD |
    |---|---|---|
    | 2 | ±0.01 | 1.7e-6 |
    | 4 | ±0.25 | 9.0e-6 |
    | 6 | ±0.25 | 4.9e-6 |

    At ±π the same circuits give about 1e-11.
  - **Consequence:** a determined run with tolerance near the 1e-6 minimum fails `holdout_accuracy`. Its detail text gives no reason, because the "underdetermined" suffix appears only when not determined (surrogate.py:336-341).
  - **Coverage:** the checks stay honest, and the default tolerance of 0.01 passes. But the SPEC AC "predicts held-out distributions to TVD ≤ 1e-6" (SPEC.md:463) is tested only on a ±π box (test_surrogate.py:164), and the UI offers ±π/2 and ±π/4.
  - **Fix:** qualify the claim in ADR-41, the SPEC AC and the UI note ("exact over ±π; for narrower spans accuracy is limited by the ridge, about 1e-5"). Alternatively, scale the ridge to the Gram trace or use `lstsq`. Then add a small-span test.
- **M-R10-2 (minor, F-22): a `parameters` list that is too long or empty returns TOO_LONG or TOO_SHORT, not the §3 codes.** The cause is `Field(min_length=1, max_length=6)` at services/api/qai_api/schemas/jobs.py:151. SPEC.md:150 says 1–6 angles fail at `params.parameters` with NO_PARAMETERS or TOO_MANY_PARAMETERS.
  - **Verified with `validate_params`:** 7 indices give `TOO_LONG` ("List should have at most 6 items after validation, not 7"), and `[]` gives `TOO_SHORT`.
  - **Impact:** the path is right, and the web client never sends these lists, so only API-key clients see the mismatch.
  - **Fix:** drop the pydantic length bounds and let `parameter_slots` return the §3 codes. Alternatively, amend §3.
- **M-R10-3 (minor, F-23): a server `circuitId` error does not land on the circuit field.** SPEC.md:485 says "`params.*` and `circuitId` errors land on their fields". The circuit `Field` at apps/web/app/app/surrogate/SurrogateView.tsx:179-194 has no `error` prop. The error is rendered instead as a free-standing `role="alert"` paragraph at the top of the form (SurrogateForm.tsx:97-102). Nothing links it to the `#sur-circuit` select (no `aria-invalid` or `aria-describedby`), so a screen-reader user tabbing to the select does not hear it. Fix: lift the message to the circuit `Field` (`error={...}`), or reference `sur-circuit-error` from the select.
- **N-R10-1 (note, F-22): the target pattern error leaks a regex.** A malformed `target` returns the pydantic message "String should match pattern '^[01]{1,10}$'" (schemas/jobs.py:156; verified). The form shows server messages under the field. A plain sentence like the length message at schemas/jobs.py:205 would read better.
- **N-R10-2 (note, F-22): error codes are chosen by matching message text.** schemas/jobs.py:198-202 picks NO_PARAMETERS vs TOO_MANY_PARAMETERS by `"no rotation" in exc.message`, and DUPLICATE vs NOT_PARAMETRISED by `"twice"`. Rewording an SDK message silently changes the API code. Also, DUPLICATE and the out-of-range index (which maps to NOT_PARAMETRISED) are not spelled out in SPEC §3. The tests do pin the current mapping (test_jobs_surrogate.py:115-119).
- **N-R10-3 (note, F-22): the submit-time circuit parse runs on the event loop, before the job rate limit.** `check_circuit_params` runs `Circuit.from_dict` plus `parameter_slots` synchronously at services/jobs.py:228, before `enforce_job_submit` at :231. That measured about 10 ms for a 2,000-op, 10-qubit circuit. It is small and bounded by the per-IP and per-principal limits, but it is CPU work outside `cpu.run_bounded` and ahead of the cost limit.
- **N-R10-4 (note, F-21/F-22): little timeout headroom, and a surrogate costs the minimum rate weight.**
  - The worst case measured 54.9 s on one thread here, against 120 s. That is about 2.2x headroom, and the exact stage is 96 % of it.
  - Under S5 a 10-qubit surrogate costs 1 cost unit, the same as a trivial `simulate`. A tenant can therefore keep MAX_QUEUED_PER_TENANT (50) near-minute jobs queued.
  - This is spec-conformant and bounded by backpressure, but worth watching on slower hosts.
- **N-R10-5 (note, F-21): ranking and reporting use different predictions.** Candidates are ranked by the raw column (surrogate.py:339-340), but the reported `predicted` is the clipped and renormalised value (:341). The two agree when the model is determined. In an underdetermined run the reported order may not be monotone in `predicted`.
- **N-R10-6 (note, F-23): the underdetermined warning is never exercised by a test, and the e2e comment that claims it is is wrong.** apps/web/e2e/w9-surrogate.spec.ts:81-83 says "A sample count under the 3^d features warns before submit". It fills 16 samples at d = 2 (13 train > 9 features, so the model is determined) and asserts the warning count is **0**. No Vitest or e2e test renders `[data-underdetermined]`, which is a §6 screen state.
- **N-R10-7 (note, F-23): the warning's live region may not be announced.** `role="status"` sits on the warning, which is inserted conditionally (SurrogateForm.tsx:275-284). Some screen readers do not announce a live region that is added along with its content. An always-present container would be more reliable.
- Carried forward: N-R8-6 (no web baseline evidence, INV-2 pre-existing) is still open and out of scope. Playwright W9 and the smoke check were not executed in this round, because they need the live stack.

Counts: 0 blocker, 0 major, 3 minor, 7 notes.

## Overall

**PASS.** The surrogate is sound. Its basis contains the true response of every allow-listed parametrised gate. The feature count, the `determined` rule, the hold-out split, the default samples and the limits match SPEC §5 F-21 and §3 exactly, and the results are deterministic by seed down to the evidence sha256. The batched exact stage is provably equal to `Statevector` and keeps the worst case under the job timeout.

The API path is correct:
- Circuit rules use the SDK.
- Tenancy uses `tenant_session`, and another tenant's circuit is 404.
- The migration is reversible, with a refusal guard.
- The evidence hashes and verifies.

The web screen mirrors the server limits. It is INV-11 clean and token-only, and its e2e test covers W9.1 to W9.5.

The three minors to fix are:
- M-R10-1: the over-strong exactness claim for narrow spans.
- M-R10-2: the TOO_LONG/TOO_SHORT codes where §3 names TOO_MANY_PARAMETERS/NO_PARAMETERS.
- M-R10-3: the `circuitId` error not attached to its field.

None of them blocks the gate.

Verdict: PASS
