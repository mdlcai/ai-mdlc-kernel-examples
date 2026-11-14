Commits reviewed: b4491d4721b7885d9b7b04fc76c2c919767646b0 (RB-18, round 16 follow-ups)
Previous round reviewed: round 16 (RB-17 9692ded, C-25 177e197 release 0.7.0)

# Reviewer Gate: Review Round 17 (RB-18 round 16 follow-ups)

- **Mechanism:** a fresh-context subagent reviewed `git show b4491d4` (formula.py `_map` and `_amplitudes`, test_formula.py) against artifacts/reviewer/round-16.md (M-R16-1, N-R16-1, N-R16-2, N-R16-3), SPEC.md §3 (map params at SPEC.md:156) and §5 F-28 (the map rule at :609, the evidence outputs at :614). It also read the worker's error path (services/api/qai_api/worker/child.py:73-86, runners.py:13-15 and :405-420) and the web consumers (apps/web/components/results/formula-panels.tsx:111-118, :183, :261, :274, :727-730, :820-821; apps/web/app/app/formula/). It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider` gave **477 passed, 1 skipped** in 76.72 s.
  - API: `cd services/api && python -m pytest -q -p no:cacheprovider tests/test_jobs_formula.py` gave **9 passed** in 38.41 s (against the already-running qai-test-pg and qai-test-valkey).
  - Web: `npx vitest run components/results/formula-panels.test.tsx app/app/formula/_lib/form.test.ts lib/expr.test.ts` gave **3 files, 93 tests passed**.
  - A float-extreme harness in the session scratchpad (`r17/edge.py`). It ran `run_formula` in state mode on 27 amplitude expressions and qubit counts under `python -I`, recording warnings, against three trees: RB-18 (the working tree), RB-17 (`git archive b4491d4^`) and pre-RB-17 (`git archive 9692ded^`). The cases include `exp(-745)`, `1e-310`, `1e-320*k`, `5e-324*k`, `exp(709.7)*(1+i)`, `exp(709)` on 1 to 4 qubits, `exp(708)` on 4 and 8 qubits, `exp(709)*(k+1)` and `1e300*1e8*(1+i)`.
  - The same failing case run through the worker entry point, `qai_api.worker.child.run({"kind": "formula", ...})`, under `python -I`.
  - Old-against-new comparisons (`r17/ident.py`, `r17/ev.py`). The first hashes the sorted result JSON, with `ms` removed, for 16 state expressions on 1, 2, 3, 4, 6, 8 and 10 qubits (112 runs). The second hashes the `evidence_parts` triple and the full result for 28 state runs, 17 map runs, 1 objective run and 1 physics run. Both run on all three trees.
  - Map boundary and precedence cases (`r17/m.py`): a non-finite E at n = 1 with |Ψ| above the bound, Ψ₀ exactly at the bound, Ψ₀ just outside it, and a non-finite E₀.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual.
    - INV-1 passes (142 decisions, 84 commits since the plan, 0 undone, 28 human answers).
    - INV-10 (349 files), INV-11 (569 files) and INV-34 pass.
    - INV-2 fails: "8 unaccepted failure(s) ... report covers 2 screens, SPEC UI Surface lists 28 ... source changed since gitSha 3f35306 (53 file(s) ...): stale".
  - `npm run lint:py`: "All checks passed!" and "160 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 0; pending review RB-18; nothing in rework or blocked.
  - `grep -E '^(APP_VERSION|GIT_COMMIT)=' .env` (the two version lines only).

  It did not run the full API suite, Playwright, the web baseline or the smoke test. It did not touch docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json or any tasks/*.output file. It ran govern.mjs only as `queue`, edited only this file and staged nothing. The harness files are in the session scratchpad, outside the repo.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-18 | b4491d4 Round 16 follow-ups | **FAIL** (1 blocker, 2 notes) | M-R16-1's two edges (a subnormal peak, a complex modulus that overflows) are fixed with no RuntimeWarning. N-R16-2 and N-R16-3 are resolved, and N-R16-1 is resolved for the evidence hash. The fix adds a regression: when the norm overflows but the largest part is below 2^1023, `math.ldexp` raises OverflowError. The job then fails as a retryable RUNNER_ERROR instead of returning a result. This includes round 16's own `exp(709)` on 4 qubits, which passed in RB-17 (B-R17-1). |

Carried forward (human action): the queue (RB-18 pending review; this round is its review evidence, and it needs a rework) and INV-2 (stale for 53 files, 8 failures, no `/app/formula` screen in the report).

## RB-18 (b4491d4): finding by finding

| Check | Result | Evidence |
|---|---|---|
| M-R16-1: subnormal peak | RESOLVED | formula.py:518-523 takes the largest real or imaginary part and shifts by `frexp(peak)[1]` with `np.ldexp`. On RB-18, `exp(-745)`, `1e-310`, `1e-320*k`, `1e-300*1e-23*(k+1)` and `5e-324*k` all pass with no warnings. `nonzero` values are 1e-323, 2e-310, 3.74e-320, 5.4e-323 and 6e-323, which equal rawNorm. On RB-17 each failed `nonzero` and emitted 2 RuntimeWarnings. |
| M-R16-1: complex modulus overflow | RESOLVED | `exp(709.7)*(1+i)` (2 qubits) and `exp(709.5)*(1+i)` (1 and 3 qubits) pass. RB-17 failed them with value 0.0. Using the largest part instead of `np.abs` means `peak` is finite whenever every part is finite. |
| M-R16-1: no RuntimeWarning | PASS | The harness records 0 warnings for every RB-18 case. The new test (test_formula.py:374-385) runs under `simplefilter("error")`. |
| M-R16-1: rawNorm null and honest detail when the norm overflows | PASS where it is reached | When `shift == 1024`, `norm = inf` (formula.py:525), rawNorm is `null`, and the detail reads "the largest part before normalising is > 0 (the norm itself overflows a float)". `exp(709)*(k+1)`, `exp(709.7)`, `1e300*1e8*(1+i)` and `exp(709)*(1+i)*exp(i*k)` all give this. The value is the largest part, which matches the wording, and the JSON is strict. This closes N-R16-2. |
| M-R16-1: norm overflows but `shift < 1024` | **BLOCKER (B-R17-1)** | See the finding. `exp(709)` on 3 or 4 qubits and `exp(708)` on 8 qubits raise `OverflowError: math range error`. RB-17 gave pass with rawNorm `null`. |
| M-R16-1: tests | GAP | The parametrisation (test_formula.py:374) has 2-qubit cases only, and each overflow case has `shift == 1024`. The `math.ldexp` branch is never given an overflowing result. |
| N-R16-1: ordinary states, evidence | RESOLVED | Across 28 ordinary state runs (7 expressions on 1, 3, 6 and 10 qubits), the `evidence_parts` hash on RB-18 equals the pre-RB-17 one in every case. RB-17 differed in 12 of them (`sin(k)+i*cos(3*k)`, `exp(i*pi*k/N)` on 3 qubits, `cos(k*q)-i*sin(k^2)` and `(k+1)^3*exp(i*k)`), and RB-18 restores them. |
| N-R16-1: ordinary states, full result JSON | NOTE (N-R17-1) | Out of 112 runs, the full result JSON matches pre-RB-17 except the expected `1e±200` cases (which failed before RB-17) and `cos(k*q)-i*sin(k^2)` on 1 to 6 qubits. In those runs the only differing byte is `"im": -0.0` turning into `"im": 0.0` in `measure.top` for k = 0. `np.linalg.norm` on the shifted input is exact, because a power-of-two shift commutes with squaring, summing and `sqrt` when nothing is subnormal, so `scaled` and the drift match. The sign is lost when the array is rebuilt as `real + 1j*imag` (formula.py:523). `re` and `im` are not in the evidence headline (formula.py:730), so evidence is unaffected. |
| N-R16-1: pinning test | PARTIAL | test_formula.py:388-399 compares only the `normalised` drift with `raw / ||raw||`. The function name and the comment at formula.py:520-521 claim bit-identity, which the signed zero contradicts. |
| N-R16-3: Ψ₀ against the bound | RESOLVED | formula.py:373-374 stops at n = 0 with "out of bounds" when E₀ is finite and |Ψ₀| > bound. `cos(psi)` from 1e6 with bound 10 used to be **inconclusive**, because it iterated back inside the bound and `bounded` passed with value 1e6. It now fails, with "|psi| left the bound at n = 0". Ψ₀ exactly equal to the bound passes, as it does inside the loop (`>`, not `≥`). |
| N-R16-3: a non-finite E₀ | PASS | With `log(n)` and Ψ₀ = 1e6 > 10, `stopped` is `{n: 0, why: "not finite"}`, and both `finite` and `bounded` fail. The new test covers this (test_formula.py:402-412). |
| N-R16-3: precedence of stop reasons | PASS | "not finite" wins at n = 0 (:372 before :373) and at n ≥ 1 (the E check at :378 comes before the bound check at :384). `bounded` is now computed from the trajectory (:391-393), so it can fail alongside `finite`. For `psi+100` with energy `1/(psi-101)`, bound 10, the run stops "not finite" at n = 1 with Ψ = 101. RB-17 passed `bounded` at 101 > 10. RB-18 fails it. That closes the same N-R16-3 class for n ≥ 1. |
| N-R16-3: the `finite` check and JSON strictness | PASS | `finite` (:389) is unchanged and still keys on `stopped.why`. Every map case dumps with `allow_nan=False`. `widest` skips `None` entries, and `trajectory[0]` is never `None`, so `max` cannot be empty. |
| N-R16-3: SPEC §3 text | PASS | SPEC.md:609, "stopping early on a non-finite value or \|Ψ\| > bound", now holds for Ψ₀ as well. The SPEC AC maps (:619) are not affected: their Ψ₀ is the 0.5 default. |
| Other modes | PASS | Map runs inside the bound give the same evidence and full-result hashes on RB-17 and RB-18. Only the Ψ₀-outside cases change, as intended. The objective and physics runs are identical. |
| Evidence hashing | PASS, 1 consequence | The detail and value changes in `nonzero` and `bounded` go into evidence `checks` (formula.py:746-752), so evidence hashes change for overflow states and for out-of-bound Ψ₀. Only same-version reproducibility is promised (SPEC.md:337), so this is expected. |
| Web | PASS | formula-panels.tsx:729 renders "Stopped at step {formatNumber(n)}", and `formatNumber(0)` gives "0" (app/app/jobs/_lib/format.ts:57-62). With a single trajectory point the charts are skipped (`psi.length > 1`, :732). rawNorm `null` reads as `undefined` through `num` (:134), so the KPI is hidden (:820). No web code matches on the detail strings or assumes `stopped.n > 0`. 93 tests pass. |
| Lint, invariants | PASS | ruff check and format are clean. INV-1, INV-10, INV-11 and INV-34 pass. |
| Commit message | INACCURATE | It says a complex modulus beyond the float range "normalise[s]", and "ordinary states are bit-identical to raw / \|\|raw\|\| again". The first fails for B-R17-1 inputs. The second is off by a signed zero (N-R17-1). |
| Governance | PASS (pending) | The queue lists RB-18 as pending review. |

## Round 16 notes: still open?

| Note | Status | Evidence |
|---|---|---|
| N-R16-1 bit-identity of ordinary states | RESOLVED for evidence, see N-R17-1 | See the table above. |
| N-R16-2 `nonzero.value` when the norm overflows | RESOLVED where reached | The detail now says what the value is. The B-R17-1 inputs never reach it. |
| N-R16-3 `bounded` above its threshold | RESOLVED | See the table above. |
| N-R16-4 local `.env` pins the old version | **OPEN** | `.env` still has `APP_VERSION=0.6.0` and `GIT_COMMIT=5ce9f06`. The stack is being rebuilt now, so the images will be tagged 0.6.0 unless `.env` is edited first. |
| N-R15-1 to N-R15-4 | OPEN (unchanged) | RB-18 does not touch the parsers, the objective check, the test gaps or the baseline. |

## Findings

- **B-R17-1 (blocker, RB-18 regression): `math.ldexp` raises OverflowError when the norm overflows but the largest part is below 2^1023.**
  - Location: formula.py:525, `norm = math.ldexp(scaled, shift) if nonzero and shift < 1024 else ...`. `shift < 1024` only says the peak fits. When `scaled` (up to √(2·2^q)) pushes `scaled · 2^shift` past the float range, `math.ldexp` raises instead of returning inf.
  - Reproduced: `{"mode":"state","expression":"exp(709)","qubits":4}` (round 16's own N-R16-2 example), `exp(709)` on 3 qubits and `exp(708)` on 8 qubits all raise `OverflowError: math range error` from `run_formula`. RB-17 gave status pass, rawNorm `null`, for all three. Through the worker, `child.run` returns `{"ok": false, "code": "RUNNER_ERROR", "message": "OverflowError: math range error", "retryable": true}`. The job is then requeued with backoff and ends `dead` after JOB_MAX_ATTEMPTS (runners.py:13-15), with no result and no evidence, for an input the API accepted.
  - It is reachable by any member who can submit a formula job, and it is deterministic. Each retry costs one child process.
  - Fix: decide overflow from the exponents, for example `norm = math.ldexp(scaled, shift) if nonzero and shift + math.frexp(scaled)[1] <= 1024 else (math.inf if nonzero else 0.0)`, or catch OverflowError and use `math.inf`. Then the existing "the norm itself overflows" detail and the rawNorm `null` apply. Add `("exp(709)", 4)` and `("exp(708)", 8)` to the test_formula.py:374 parametrisation, which needs a qubits parameter.
- **N-R17-1 (note, RB-18): rebuilding the shifted array loses the sign of zero, so the result JSON is not fully bit-identical to `raw / ||raw||`.**
  - Location: formula.py:523, `np.ldexp(raw.real, -shift) + 1j * np.ldexp(raw.imag, -shift)`. `1j * (-0.0)` has imaginary part +0.0, so an amplitude with imaginary part -0.0 is stored as +0.0.
  - Example: `cos(k*q)-i*sin(k^2)` on 1 to 6 qubits. `measure.top[k=0].im` is `-0.0` before RB-17 and `0.0` on RB-18. Nothing else differs. Probabilities and the evidence hash are unaffected.
  - Fix, if exact identity matters: `unit = np.empty_like(raw); unit.real = np.ldexp(raw.real, -shift); unit.imag = np.ldexp(raw.imag, -shift)`. Have test_formula.py:388 compare `state.data` (or the `top` entries with `math.copysign`) as well as the drift. Otherwise, soften the comment at formula.py:520-521 and the test name.
- **N-R17-2 (note, RB-18): the overflow tests only cover `shift == 1024`.** All four cases at test_formula.py:374 are on 2 qubits. Three have a largest part ≥ 2^1023. `exp(709)*(1+i)*exp(i*k)` reaches 1.14e308 through the real part. None of them takes the `math.ldexp` branch with an overflowing result, which is why B-R17-1 got through 477 passing tests.

Counts: 1 blocker, 0 major, 0 minor, 2 notes.

## Carried forward (human action)

- **Queue:** RB-18 is pending review. This round is its review evidence, and it needs a rework for B-R17-1.
- **INV-2:** fails with 8 unaccepted failures. The report is stale for 53 files and covers 2 of 28 screens, without `/app/formula`. It needs a quiet-host baseline run or an explicit human acceptance before the 0.7.0 ship banner (N-R15-4).
- **N-R16-4:** set `APP_VERSION=0.7.0` and `GIT_COMMIT` to the release head in `.env` before the rebuild that is under way, and quote the `/api/health` line in the smoke record.

## Overall

- RB-18 closes both float edges from M-R16-1, makes the overflow detail accurate (N-R16-2) and closes N-R16-3 at n = 0 and at n ≥ 1, with no web impact. Evidence hashes for ordinary states again equal the pre-RB-17 ones.
- The new exponent guard covers only an overflowing peak, not an overflowing norm. States whose norm exceeds the float range while every part stays below 2^1023 now crash the runner and end as dead jobs, where RB-17 returned a result (B-R17-1).

Verdict: FAIL (RB-18). It needs a one-line fix at formula.py:525 with two more test cases. N-R17-1 and N-R17-2 can go with that fix.
