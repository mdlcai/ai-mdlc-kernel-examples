Commits reviewed: 3d2774caffc25ed6960e85930d419e8e534e3df9 (C-21), c47313223c6334893959ea3955778218e61bd099 (F-24), d98ba6dcd207174939b47f685fb680d88bc4fb85 (C-22), e86426f976a40ddd994186be94cc02aaca0a3e58 (F-25), 4b4855c1bf303bca4a86ead8c27b5867f7a2c82f (F-26)
Previous round reviewed: 8e71d00 (round 11, RB-9, C-19, RB-10)

# Reviewer Gate: Review Round 12 (Re-Entry R4: C-21 docs, F-24 SDK Orb Lab, C-22 wave 12, F-25 kind orb, F-26 Orb Lab web and Evolution Engine demo)

- **Mechanism:** a fresh-context subagent reviewed each commit on its own with `git show <sha>`. Governance named F-25 (GD-0114) and F-26 (GD-0115) as focus items. The commits were checked against SPEC.md §3 `orb`, §5 F-24..F-26, §6, §7 and W10, ARCHITECTURE.md §2.9 and §9, DECISIONS.md ADR-46, invariants.json and the CLAUDE.md conventions. It ran these:
  - SDK tests: `pytest -k orb` gave **31 passed, 303 deselected in 23.6 s**.
  - API tests: `tests/test_jobs_orb.py tests/test_migrations.py` against qai-test-pg and qai-test-valkey gave **10 passed in 112.1 s**.
  - Web checks: `npx vitest run app/app/orb components/results/orb-panels.test.tsx components/site app/app/optimization` gave **4 files, 44 tests passed**. `npx tsc --noEmit` exits 0.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (31 pass, **2 fail**), 1 manual. INV-1 fails, which is new this round. INV-2 fails, which is pre-existing.
  - Two throwaway probes in the session scratchpad, outside the repo, both run with `python -I`:
    - probe 1 timed `run_orb` at the default and at the limit corners, and measured when `durr_hoyer` first finds the optimum compared with the queries it reports;
    - probe 2 measured result JSON sizes and swept tolerance 1e-9 over 32 extreme combinations of radius, conductivity, pulse, qinRatio and mode 0/19.
  - The TE/TM Q0 closed forms were derived by hand and compared with the code's normalisation.

  It did not run Playwright or a browser (W10 was reviewed statically against the markup). It did not touch docker, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-21 | 3d2774c Orb Lab docs (SPEC, ARCHITECTURE §2.9, ADR-46) | **PASS** (1 minor, 1 note) | The docs are consistent on limits, outcomes and evidence. SPEC and ARCHITECTURE disagree on stage order (M-R12-3). The "2–3 s" runtime claim covers only the default (N-R12-1). |
| F-24 | c473132 SDK `qai.orb` (ORB-0) | **PASS on code; governance gap open** (1 major process, 1 minor, 2 notes) | The physics is correct: the closed forms check out, the ledger balances and the validation rejects bool, NaN and inf. **The commit has no governance decision, so INV-1 fails** (M-R12-0). The QAI `queries` metric is always equal to the budget (M-R12-1). |
| C-22 | d98ba6d BUILD-PLAN wave 12 | **PASS** (1 note) | Committed after F-24. The `files_write` manifests drift from what F-25 and F-26 actually touched (N-R12-5). |
| F-25 | e86426f API kind `orb`, migration 0011, `_budget` fix | **PASS** (1 note) | The params, codes, tenant isolation, evidence gating and the guarded downgrade are all correct and tested. The `_budget` fix is correct. Its sub-1 s case is not unit-tested (N-R12-6). |
| F-26 | 4b4855c Orb Lab screen, panels, Evolution Engine | **PASS** (1 minor, 4 notes, plus M-R12-3) | The limits mirror §3. Every chart has a table alternative and every string renders as text. The τ0 hint is wrong by a factor of 2 (M-R12-2). The stage eyebrows follow ARCHITECTURE, not SPEC (M-R12-3). |

## C-21 (3d2774c): docs

| Check | Result | Evidence |
|---|---|---|
| §3 `orb` params and limits | PASS | SPEC.md:151: seed required, radius 0.01–1, conductivity 1e5–1e8, modes 1–20, mode 0..modes−1, power 1e-3–1e6, pulse 0.5–50, qinRatio 0.01–100, qubits 3–8, budget 1–2048 (default ⌈8√2^q⌉), tolerance 1e-9–1e-3. circuitId gives 422 UNSUPPORTED. These match the SDK (orb.py:46-54), the API (schemas/jobs.py:187-217) and the web form (orb/_lib/form.ts). |
| Outcome and evidence rules | PASS | SPEC.md:525 (fail, then inconclusive, then pass) and ARCHITECTURE.md:133 (evidence passes only on a pass outcome) match orb.py:637-655 and runners.py (`"passed": r["status"] == "pass"`). |
| Stage order | FAIL (M-R12-3) | SPEC.md:520-522 orders 1 Model, 2 Simulate, 3 Optimize. W10 (SPEC.md:625) orders Model → Simulate → Optimize. ARCHITECTURE.md:128-129 orders 2 Optimize, 3 Simulate. |
| Size and runtime claims | PARTIAL (N-R12-1) | ADR-46 "about 90 KB" (DECISIONS.md:383): measured 89,733 B at the default. ARCHITECTURE.md:133 "2–3 s": measured 2.44 s at the default, but up to 15.3 s at a limit corner. |

## F-24 (c473132): SDK `qai.orb`

| Check | Result | Evidence |
|---|---|---|
| Mode roots and Q0 | PASS | TE modes solve j_l(x) = 0 and TM modes solve j_l + x·j_l' = 0. Q0 is computed by quadrature and checked against TE a/δ and TM (a/δ)(1 − l(l+1)/x²). Both closed forms were derived independently and match the code's normalisation. The tabulated-zero tests pass. |
| Ledger | PASS (N-R12-3) | DOP853 integrates 5 states. The rhs is supplied = s² − (−s + √(2γ_in)a)², extracted 2γ_ext·a², lost 2γ0·a² (orb.py:272-337). `relativeResidual` comes out near 1e-13. |
| Dürr–Høyer | PASS (M-R12-1) | It uses the BBHT schedule with λ = 6/5 (orb.py:354-387). For seed 7 it finds the classical optimum within the default budget of 128. |
| Input validation | PASS | `_int` rejects bool, and `_num` rejects bool, NaN and inf (orb.py:80-94). Tests cover a bool seed and a NaN radius. |
| Extremes | PASS | Tolerance 1e-9 over 32 extreme combinations gave 0 physics failures (probe 2). |
| Outcome logic | PASS | A physics failure gives fail, a comparison failure gives inconclusive, otherwise pass (orb.py:637-655). Tested with budget = 4 (inconclusive) and a monkeypatched ledger (fail, test_orb.py:216-232). |
| Governance | **FAIL (M-R12-0)** | `npm run lint:invariants`: `FAIL INV-1 c473132 "F-24: SDK Orb Lab (qai.orb, ORB-0)" has no governance decision`. GOVERNANCE-DECISIONS.jsonl holds GD-0112 (C-21), GD-0113 (C-22), GD-0114 (F-25) and GD-0115 (F-26), but nothing for F-24. |

## C-22 (d98ba6d): BUILD-PLAN wave 12

| Check | Result | Evidence |
|---|---|---|
| Wave content | PASS | The wave lists F-24, F-25 and F-26 with their dependencies and the W10 workflow. |
| Order | NOTE | C-22 landed after the F-24 code (c473132), so F-24 was built before its plan entry existed. |
| Manifests | NOTE (N-R12-5) | F-25 `files_write` lists services/jobs.py, models/releases.py and smoke-test.sh, which F-25 did not touch. It omits worker/evolution_runner.py, tests/test_evolution_api.py, tests/test_jobs_pipeline.py and tests/test_jobs_surrogate.py, which F-25 changed. F-26 omits components/results/orb-panels.css; its commit message acknowledges this. |

## F-25 (e86426f): API kind `orb`

| Check | Result | Evidence |
|---|---|---|
| OrbParams | PASS | `StrictInt` and `_bounded` (schemas/jobs.py:187-217, 258-264): a bool or non-number gives INVALID; out of range, NaN or inf gives OUT_OF_RANGE; `mode >= modes` gives OUT_OF_RANGE. All codes are in §7. The worker child serialises with `allow_nan=False`. |
| circuitId and tenancy | PASS | A circuitId on a circuit-less kind gives UNSUPPORTED (services/jobs.py:204). The lookup runs inside `tenant_session`, so a cross-tenant project gives 404. A viewer gets 403. All are tested in test_jobs_orb.py. |
| Runner and evidence | PASS | The runner calls `qai.orb.run_orb` and maps CircuitError to InputError. Evidence `passed` is true only when the status is pass. numQubits is `qubits`. |
| Migration 0011 | PASS | It recreates the three kind CHECKs. The downgrade refuses while any orb rows exist, and test_migrations.py tests that. |
| `_budget` fix | PASS (N-R12-6) | `cap = min(caps); return max(cap - elapsed, min(cap, 1.0))` (evolution_runner.py:34-47) never drops below min(cap, 1 s), and a cap under 1 s is no longer raised to 1 s. |
| Smoke | PASS | scripts/smoke-test.mjs gains an orb check in the F-18/F-22 pattern. It is not one of the pinned runners. |
| Runtime and size | PASS (N-R12-1) | Worst corner measured: 15.3 s and 261 KB, against JOB_TIMEOUT_SEC 120 (config.py:35) and MAX_OUTPUT_BYTES 64 MiB. |

## F-26 (4b4855c): web

| Check | Result | Evidence |
|---|---|---|
| Form limits | PASS | orb/_lib/form.ts copies every §3 bound. `decimalIn` combines a regex with a range check, `intIn` checks integers, and the budget may be blank or 1–2048. "Load example" restores ORB-0 (OrbForm.tsx:89-93) and announces it in a persistent status region (OrbForm.tsx:128). |
| Role and flow | PASS | RoleRequired gates member+. Projects load, and a submit pushes `/app/jobs/[id]`. |
| Panels | PASS (M-R12-2, N-R12-2, N-R12-4) | The render order is Report, Model, Simulate, Optimize, Compare, Validate, as SPEC F-26 requires, and the test asserts it. `parseOrbResult` degrades a malformed result instead of crashing, caps the grid at 64×64 and keeps inconclusive distinct from fail. Evidence is linked only on a pass. |
| Accessibility and INV-11 | PASS | Every LineChart carries a table alternative (test: every `figure` has a `table`). The field map has a details table. Alignment and success are tables. There is no `dangerouslySetInnerHTML`, and the hostile-string test passes. |
| Tokens | PASS (N-R12-4) | Every custom property used in orb-panels.css, evolution-engine.css and orb.css exists in globals.css. No token values were changed. |
| Evolution Engine and Optimization | PASS | The landing section shows the loop and the limits from SPEC.md:560 (QAOA MaxCut, ≤ 12 nodes, simulator only, 1–20 iterations). Optimization "Load example" fills ring5 (p 1, maxiter 20, 6 iterations, ε 0.001, 5 seeds, rngSeed 7), within SPEC.md:128. The optional rngSeed is in [0, 2^32 − 1], matching routes/evolution.py:66. The example button shows only to writers with a live project. Tests were added. |
| Nav and labels | PASS | Orb Lab sits after AI Simulation in the Execution group. The orb kind labels were added for jobs, validation and governance. |
| W10 e2e (static) | PASS (M-R12-1) | Every selector exists in the markup: `#orb-*` ids, `data-orb-lowest/settings/budget`, region names via Panel `aria-labelledby`, `tr[data-mode]`, `data-frequency`, `data-ledger`, `data-search`, `data-evaluations`, `data-queries`, `data-round`, `data-alignment`, `data-query-ratio`, `data-orb-outcome`, `data-driven-mode`, `data-orb-checks`, `data-check-status`, `data-orb-evidence`, and `li[data-evidence-kind]` (JobDetailView.tsx:381). The `queries ≤ 128` assertion and the `data-query-ratio` = `0.500` assertion (w10-orb.spec.ts:93, 103) are tautological, for the reason given in M-R12-1. |

## Findings

- **M-R12-0 (major, process, F-24): c473132 has no governance decision, so INV-1 fails.**
  - **Evidence:** `npm run lint:invariants` reports `FAIL INV-1 c473132 "F-24: SDK Orb Lab (qai.orb, ORB-0)" has no governance decision`. GOVERNANCE-DECISIONS.jsonl has GD-0112 to GD-0115 for C-21, C-22, F-25 and F-26, but none for F-24. INV-1 passed in round 11, so this failure is new. BUILD-STATE.md:10 already marks F-24 "UNGOVERNED" and asks a human to record it.
  - **Fix:** a human records the decision (`node scripts/govern.mjs decide --change F-24 --commit c473132`). This reviewer did not run govern.mjs. The code itself passes review. This is not a code blocker, but the release should not ship while INV-1 fails.
- **M-R12-1 (minor, F-24, also F-26): the QAI `queries` figure always equals the budget, so "QAI found the optimum in N queries" carries no information.**
  - **Code:** `durr_hoyer` loops `while queries < budget`, and each round's k is capped at `budget − queries − 1` (orb.py:363-369). It spends the whole budget and returns `queries == budget`. Probe 1 confirmed this for every budget and seed tested.
  - **Effect:** for seed 7 the optimum is first measured at query **61**, yet the reason (orb.py:650-655) says "with 128 queries". `queryRatio` (orb.py:549) is budget / 2^qubits, which is 0.5 by construction. The "no fewer queries than 2^qubits" note (orb.py:654, SPEC.md:525) simply tests budget ≥ N. The UI repeats the number: "128 oracle queries of 128" (orb-panels.tsx:641-642) and "Query ratio: QAI queries / classical evaluations" (orb-panels.tsx:746-749). The SPEC AC "≤ 128 queries" (SPEC.md:531, W10.3 at SPEC.md:628) and the e2e assertions (w10-orb.spec.ts:93, 103) can never fail.
  - **Fix:** record `queriesToOptimum`, the query count when the final best was first measured, and report it alongside the budget, in the reason, `queryRatio` and the Optimize table. Alternatively, reword every use to say "budget spent", and qualify SPEC/ADR-46 to match.
- **M-R12-2 (minor, F-26): the τ0 hint is wrong by a factor of 2.** orb-panels.tsx:471 labels the unloaded time constant "τ0 = 2Q0/ω", but the SDK computes `tau0 = q0 / omega` (orb.py:445, defined in the comment at orb.py:50). With γ = ω/2Q, the field decays as e^(−γt) and the energy as e^(−ωt/Q), so Q0/ω is correct as the energy time constant. The label disagrees with the value it labels. Fix: change the hint to "τ0 = Q0/ω".
- **M-R12-3 (minor, C-21 and F-26): the stage numbering is inconsistent.** SPEC F-24 (SPEC.md:520-522) and the W10 header (SPEC.md:625) order the stages 1 Model, 2 Simulate, 3 Optimize, and so do the orb.py stage comments (orb.py:457 "# 3." Optimize, orb.py:507 "# 2." Simulate). ARCHITECTURE.md:128-129 orders them 2 Optimize, 3 Simulate. The panel eyebrows follow ARCHITECTURE ("Stage 3" Simulate at orb-panels.tsx:529, "Stage 2" Optimize at orb-panels.tsx:599), even though the panels render in SPEC order. A user therefore sees "Stage 1, Stage 3, Stage 2". Fix: settle on one order. SPEC is the source of truth, so align ARCHITECTURE §2.9 and the eyebrows to it.
- **N-R12-1 (note, C-21): "a run is 2–3 s" holds only at the default.** Probe 1 measured these times. The host was loaded by the concurrent stack build, and every run passed its physics checks.

  | Params | Time |
  |---|---|
  | default | 2.44 s |
  | qubits 3, budget 2048 | 15.29 s |
  | qubits 4, budget 2048 | 11.01 s |
  | qubits 8, budget 2048 | 10.21 s |
  | modes 20, mode 19, pulse 50, qinRatio 0.01, budget 2048 | 7.85 s |

  All are well under the 120 s job timeout. At qubits 3 with budget 2048 (1,371 rounds), the result is 261,344 B, against the ~90 KB in ADR-46. Fix: qualify ARCHITECTURE.md:133 and DECISIONS.md:383 as "at the default; up to about 15 s and 260 KB at the limits".
- **N-R12-2 (note, F-26): the field map is drawn and tabulated upside down relative to its labels.** The SDK puts row 0 at z = +a. FieldMap draws row i at `y = size − 1 − i`, which puts z = +a at the bottom (orb-panels.tsx:382). The details table `.reverse()`s the rows while its caption says "z rows from top to bottom" (orb-panels.tsx:403-413). The m = 0 energy density is symmetric in z, so nothing looks wrong today, but the orientation labelling is incorrect.
- **N-R12-3 (note, F-24): two checks are weaker than they look.**
  - **energy_conservation:** the residual `0.0 + supplied − extracted − lost − u_end` (orb.py:302) holds by construction, because the integrated rates sum to the rhs. It can fail only through integrator error or tampering, which is how the test exercises it (a monkeypatched ledger). `initial` is fixed at 0.
  - **extraction_bound:** the check tests `worst_eta ≤ 1 + tolerance` but reports a tolerance of 1.0 (orb.py:601-607).
- **N-R12-4 (note, F-26): minor UI-contract points.**
  - **Raw error element:** OrbForm.tsx:201-205 renders a raw `<p className="field-error" role="alert">` in the route, for a conductivity error shown while a preset material is chosen. A components/ui Field or FormError would be the expected primitive.
  - **Descendant selectors:** orb-panels.css:54 and 65-81 and evolution-engine.css:126 style raw elements (`.orb-loop h3`, `.orb-loop-steps dt/dd`, `.evo-limits li`) outside components/ui.
  - **Preview budget:** it falls back to the default whenever any other field is invalid, even when the budget field itself is valid (OrbForm.tsx:113).
  - **Inconclusive pill:** StatusPill has no inconclusive entry (Badge.tsx:22-39), so inconclusive renders neutral with a clock icon. The label is correct and tested.
  - **Hard-coded id:** `id="orb-loop-title"` (orb-panels.tsx:665) would duplicate if two panels ever rendered on one page.
- **N-R12-5 (note, C-22): the wave 12 `files_write` manifests drift from the commits.** See the C-22 table. Planning metadata only. No code impact.
- **N-R12-6 (note, F-25): the `_budget` fix's sub-1 s path is untested.** The unit assertions (test_evolution_api.py:370-375) cover 9.0, 90.0, None, 65.0 and 1.0. The integration test (line 378) uses evolveSeconds 1e-6. No assertion covers a cap under 1 s with no elapsed time (for example `{"evolveSeconds": 0.2}` returning 0.2), which is the exact case the fix changes.
- Carried forward: N-R8-6 / INV-2 (the web baseline is stale: 3 unaccepted failures, 20 of 27 screens, source changed since d7df5cb) is still open and pre-existing. Playwright, the smoke test and the web baseline were not run in this round.

Counts: 0 blocker, 1 major (process), 3 minor, 6 notes.

## Overall

**PASS, with one condition.** The R4 code is correct and well tested:
- The ORB-0 physics matches its closed forms.
- Inputs are rejected consistently in the SDK, the API and the web form, including bool, NaN and inf.
- Tenant isolation and evidence gating follow the conventions, and migration 0011's downgrade is guarded and tested.
- The web panels meet INV-11 and the table-alternative rule.
- The 31 SDK, 10 API and 44 web tests pass, and tsc is clean.

The condition is M-R12-0: F-24 (c473132) needs its governance decision recorded by a human before INV-1 can pass and 0.6.0 can be released.

The three minors are honesty and labelling fixes worth an RB pass before release:
- M-R12-1: the queries figure is always the budget.
- M-R12-2: the τ0 hint is off by a factor of 2.
- M-R12-3: SPEC and ARCHITECTURE disagree on stage order.

Verdict: PASS (conditional on M-R12-0)
