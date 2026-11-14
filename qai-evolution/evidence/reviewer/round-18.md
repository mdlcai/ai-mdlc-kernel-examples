Commits reviewed: b4491d4721b7885d9b7b04fc76c2c919767646b0 (RB-18, round 16 follow-ups), 87f9737de1da20a499189ad5358638703c8c6e6f (RB-19, round 17 follow-ups)
Previous round reviewed: round 17 (RB-18 b4491d4, FAIL on B-R17-1)

# Reviewer Gate: Review Round 18 (RB-18 and its fix RB-19)

- **Mechanism:** a fresh-context subagent reviewed `git show 87f9737` (formula.py `_amplitudes` at :509-545, test_formula.py :374-446) together with `git show b4491d4`, against artifacts/reviewer/round-17.md (B-R17-1, N-R17-1, N-R17-2) and round-16.md, SPEC.md §3 (the formula result shape at :209-225 and "a non-finite number is stored as null", the map params at :156) and §5 F-28 (the state checks at :610, the map rule at :609, the ACs at :620). `git diff b4491d4^ 87f9737 -- SPEC.md apps services` is empty, so the web consumers (apps/web/components/results/formula-panels.tsx:118, :274, :820-821) and the worker (services/api/qai_api/worker/child.py:60-86) are unchanged since round 17. It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider` gave **481 passed, 1 skipped** in 86.18 s.
  - API: `cd services/api && python -m pytest -q -p no:cacheprovider tests/test_jobs_formula.py` gave **9 passed** in 45.21 s (against the already-running qai-test-pg and qai-test-valkey).
  - Web: `npx vitest run components/results/formula-panels.test.tsx app/app/formula/_lib/form.test.ts lib/expr.test.ts` gave **3 files, 93 tests passed**.
  - Harnesses in the session scratchpad (`r18/`), each run under `python -I` against three trees: RB-19 (the working tree, `packages/` clean), RB-18 (`git archive b4491d4`) and pre-RB-17 (round 17's `r17/pre17`):
    - `edge.py` (round 17's 27 float-extreme state expressions, warnings recorded).
    - `ident.py` (the sorted result JSON without `ms`, 16 state expressions on 1, 2, 3, 4, 6, 8 and 10 qubits, 112 runs).
    - `ev.py` (the `evidence_parts` triple and the full result for 28 state, 17 map, 1 objective and 1 physics run).
    - `hunt.py`, which replaces `formula.expr.evaluate` with a fixed array and wraps `formula.Statevector` to capture the normalised vector. It runs 27 hand-built boundary arrays under `np.errstate(all="warn")` and a fuzz of 3,000 + 2,000 random states (1 to 10 qubits, parts spread over 10^±60 and 10^±150, 10 % exact zeros, 20 % of imaginary parts forced to -0.0) under `warnings.simplefilter("error")`. The fuzz compares the whole captured vector bit for bit (`view(np.uint64)`) with `raw / np.linalg.norm(raw)`.
    - `sub.py` (subnormal relative parts, captured vector and full-result hash, on RB-19 and pre-RB-17).
    - `child.py` (the B-R17-1 inputs through `qai_api.worker.child.run({"kind": "formula", ...})`).
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual.
    - INV-1 passes (143 decisions, 85 commits since the plan, 0 undone, 28 human answers).
    - INV-10 (350 files), INV-11 (570 files) and INV-34 pass.
    - INV-2 fails: "8 unaccepted failure(s) ... report covers 2 screens, SPEC UI Surface lists 28 ... source changed since gitSha 3f35306 (53 file(s) ...): stale".
  - `npm run lint:py`: "All checks passed!" and "160 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 0; pending review RB-18 and RB-19; nothing in rework or blocked.

  It did not run the full API suite, Playwright, the web baseline or the smoke test. It did not touch docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json, the local `.env` or any tasks/*.output file. It ran govern.mjs only as `queue`, edited only this file and staged nothing. The harness files are in the session scratchpad, outside the repo.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-18 | b4491d4 Round 16 follow-ups | **PASS with RB-19** (blocker resolved) | Round 17's findings on M-R16-1, N-R16-2 and N-R16-3 still hold. B-R17-1, its one blocker, is fixed by RB-19. On its own, RB-18 still crashes on `exp(709)` with 3 or 4 qubits and `exp(708)` with 8, so the two commits must ship together. |
| RB-19 | 87f9737 Round 17 follow-ups | **PASS** (0 blockers, 0 major, 0 minor, 2 notes) | B-R17-1, N-R17-1 and N-R17-2 are resolved. No input found raises or warns in `_amplitudes`. Every ordinary state's full result JSON and evidence equal pre-RB-17 again. Two notes: a code comment that claims more than the code does (N-R18-1) and an untested boundary (N-R18-2). |

Carried forward (human action): the queue (RB-18 and RB-19 pending review, with this round as their evidence) and INV-2 (stale for 53 files, 8 failures, no `/app/formula` screen in the report).

## RB-19 (87f9737): finding by finding

| Check | Result | Evidence |
|---|---|---|
| B-R17-1: the reported inputs | RESOLVED | formula.py:529 now uses `shift + math.frexp(scaled)[1] <= 1024`. `exp(709)` with 3 and 4 qubits and `exp(708)` with 8 qubits all pass with rawNorm `null`, the value 8.218e307 or 3.023e307 (the largest part), the detail "the largest part before normalising is > 0 (the norm itself overflows a float)" and 0 warnings. RB-18 raised `OverflowError` on all three. Through `child.run`, all three give `ok: true` with a pass result that dumps under `allow_nan=False`. Every other `edge.py` line is unchanged from RB-18. |
| B-R17-1: is the guard exact? | PASS | `scaled = m·2^e` with m in [0.5, 1), so `ldexp(scaled, shift)` is finite exactly when `shift + e <= 1024`, and the guard never rejects a representable norm. `scaled >= 0.5` whenever `nonzero`, so `frexp(scaled)` never sees 0. `math.ldexp` returns a subnormal or 0 on underflow instead of raising, so a negative `shift` is safe. When `nonzero` is false, `and` short-circuits and :530 sets the norm to 0.0. |
| B-R17-1: boundary hunt (`hunt.py`) | PASS | No case raises. Every result dumps under `allow_nan=False`. In detail: `[max, 0]`, `[max/2]×4` and `[max/32]×1024`, whose norm is exactly the float max, give rawNorm 1.7976931348623157e308 (`shift + e == 1024`, the `ldexp` branch). One ulp more (`[max/2+ulp]×4`, `[max/32+ulp]×1024`) and `[max, max]` and `max·(1+i)` give rawNorm `null` with the largest part as the value. `[max/√2]×2` and its ±1-ulp neighbours are finite. `[2^1023, 2^1023·i]` gives 1.271e308. Subnormal-only states (`[5e-324, 5e-324·i]`, `5e-324`×1024) give rawNorm 5e-324 and 1.6e-322. Mixed huge and subnormal parts (`[1e308 + 5e-324i, 5e-324 − 1e-320i]`) pass. |
| No RuntimeWarning | PASS, 1 caveat | Under numpy's default error state (`under="ignore"`, which qai and the worker leave alone; the only `errstate` is expr.py:368, set to ignore) nothing warns. That holds across the 5,000 fuzz runs under `simplefilter("error")` and the parametrised test at test_formula.py:374-399. Only when the harness forces `np.errstate(all="warn")` does `np.ldexp` report "underflow" for parts more than 2^1074 below the peak. That is the mechanism behind N-R18-1, not a production warning. |
| All zero, negative zero, NaN, inf | PASS | `[0, 0]`, `[-0.0, -0.0]` and `(-0-0i)`×4 pass `finite` and fail `nonzero` (value 0.0, rawNorm 0.0, overall fail). `nan`, `nan+nan·i`, `0+inf·i` and `-inf` fail `finite` and `nonzero` with value 0.0. Every one dumps under `allow_nan=False`. |
| N-R17-1: signed zeros | RESOLVED | formula.py:523-526 copies `raw` and assigns `.real` and `.imag` in place. `cos(k*q)-i*sin(k^2)` with 1 to 6 qubits now gives the same full-result hash as pre-RB-17 (`ident.py`). These are the only lines that differ from RB-18. In `hunt.py`'s signed-zero array, the captured vector equals `raw / ‖raw‖` bit for bit, including `0.577 − 0.0i`. Numpy's complex-by-real division still turns `-0.0 + 1i` into `+0.0` and `-1 − 0.0i` into `−0.577 + 0.0i`. It does the same in the reference and in pre-RB-17, so this is not a regression. |
| N-R17-1: whole result, not only evidence | RESOLVED | `ident.py`: all 98 ordinary state runs give the same full-result hash on RB-19 and on pre-RB-17. Only the 14 runs of `1e200*(k+1)` and `1e-200*exp(i*k)` differ, and those failed `nonzero` before RB-17, as intended. `ev.py`: all 28 state runs match pre-RB-17 on both the evidence hash and the full-result hash. The fuzz finds 0 of 4,998 vectors that differ bitwise from `raw / ‖raw‖` over the whole normalised vector, not only `top`. |
| N-R17-1: test | PASS | test_formula.py:427-446 rebuilds `raw / ‖raw‖` and compares every `top` entry's `(re, im)` pair and the sign of `im` (`math.copysign`). |
| N-R17-2: test coverage | RESOLVED | The parametrisation (test_formula.py:374-386) gains a `qubits` axis and the cases `("exp(709)", 3)`, `("exp(709)", 4)` and `("exp(708)", 8)`. These take the guard's inf branch with `shift < 1024`. All 7 cases pass under `simplefilter("error")`. |
| Other modes | PASS | `ev.py`: all 19 non-state runs (map, objective, physics) give the same evidence and full-result hashes on RB-19 and RB-18. The 3 map runs that differ from pre-RB-17 are the intended M-R15-3 and N-R16-3 changes (`psi+1`/`log(n)`, and Ψ₀ = 1e6 against bound 10). |
| Evidence hashing and JSON strictness | PASS | RB-19 changes no evidence hash compared with RB-18. Only the full result of the signed-zero runs changes, and `re` and `im` are not in the evidence (formula.py `evidence_parts` keeps `basis`, `probability` and `count`). Every harness result dumps under `allow_nan=False`. |
| SPEC §3 | PASS | SPEC.md:221 lists `rawNorm`, and :209 says "a non-finite number is stored as null", which covers rawNorm `null` when the norm overflows. :610 (`nonzero`, `normalised` ≤ 1e-10) and :620 (`0*k` fails `nonzero`) still hold. The map text at :609 is unchanged and still matches RB-18's Ψ₀ check. No SPEC edit is needed. |
| Web | PASS (unchanged) | No web file changed in b4491d4^..87f9737. formula-panels.tsx:274 maps a `null` rawNorm to `undefined` through `num`, and :820 hides the KPI then. 93 tests pass. |
| Lint, invariants | PASS | ruff check and format are clean. INV-1, INV-10, INV-11 and INV-34 pass. The only failure is INV-2, which is carried forward. |
| Commit message | ACCURATE, 1 caveat | The B-R17-1 and N-R17-1 claims hold, and so do "SDK 481 passed, 1 skipped" and "9 passed". "Ordinary results match raw / \|\|raw\|\| exactly" is true of every reported field. The internal vector can differ in parts far below the peak (N-R18-1). |
| Governance | PASS (pending) | The queue lists RB-18 and RB-19 as pending review. |

## Round 16 and 17 items: still fixed?

| Item | Status | Evidence |
|---|---|---|
| B-R17-1 | RESOLVED (RB-19) | See above. |
| N-R17-1 | RESOLVED (RB-19) | See above. |
| N-R17-2 | RESOLVED (RB-19) | See above. |
| M-R16-1 subnormal peak, complex modulus overflow | STILL FIXED | `exp(-745)`, `1e-310`, `1e-320*k`, `5e-324*k`, `exp(709.7)*(1+i)` and `exp(709.5)*(1+i)` give the same results on RB-19 as on RB-18 (`edge.py`), with 0 warnings. |
| N-R16-1 bit-identity of ordinary states | RESOLVED | Now holds for the full result as well as the evidence. |
| N-R16-2 honest `nonzero` detail on overflow | STILL FIXED | It now also covers the B-R17-1 inputs. |
| N-R16-3 Ψ₀ against the bound | STILL FIXED | The map hashes match RB-18, and test_formula.py:402-424 passes. |
| N-R16-4 local `.env` version | NOT RE-CHECKED (as instructed) | The tracked .env.example:6 has `APP_VERSION=0.7.0`. The main session sets APP_VERSION=0.7.0 in the shell for the rebuild. The local `.env` was not read. |
| N-R15-1 to N-R15-4 | OPEN (unchanged) | RB-18 and RB-19 touch neither the parsers, the objective check, those test gaps nor the baseline. |

## Findings

- **N-R18-1 (note, RB-18/RB-19): the scaling comment claims more than the code delivers for parts far below the peak.**
  - Location: formula.py:520-521, "so no part overflows or underflows ... the ratio unit / scaled is then bit-identical to raw / ||raw|| whenever that one is representable".
  - When a part is at least 2^1022 below the peak, `np.ldexp(part, -shift)` rounds it into the subnormal range or to zero, and dividing by `scaled` rounds it a second time. `sub.py` shows this: `[1, 1.5e-323]` gives `2e-323` in the vector where `raw / ‖raw‖` (and pre-RB-17) gives `1.5e-323`, and `[1, 1.0000000009313e-310]` gives `1.00000000093126e-310`.
  - It is not observable. Such a part has probability 0 (it squares to 0) and sits far below the `top` filter (`probs > 1e-15`, formula.py:487), so the full-result hash is identical on RB-19 and pre-RB-17 for these inputs. The fuzz found no difference among normal-range states.
  - Fix, optional: reword the comment, for example "no part overflows, and a part only underflows when it is over 2^1022 below the peak, so every reported value is bit-identical to raw / ||raw||".
- **N-R18-2 (note, RB-19): the exact boundary `shift + frexp(scaled)[1] == 1024` is not in the tests.** The parametrisation (test_formula.py:374-386) has cases on both sides of the boundary but none on it, where the norm is a finite value in [2^1023, max]. `hunt.py` confirms the boundary is right (`[max/2]×4` and `[max/32]×1024` give rawNorm = max; one ulp more gives `null`). The expression grammar rejects literals that large ("number is too large"), so such a test would have to inject the array directly or use something like `exp(709.78)`.

Counts: 0 blockers, 0 major, 0 minor, 2 notes.

## Carried forward (human action)

- **Queue:** RB-18 and RB-19 are pending review. This round is their review evidence, and the two should be approved together.
- **INV-2:** fails with 8 unaccepted failures. The report is stale for 53 files and covers 2 of 28 screens, without `/app/formula`. It needs a quiet-host baseline run or an explicit human acceptance before the 0.7.0 ship banner (N-R15-4).
- **N-R16-4:** quote the `/api/health` version line (0.7.0) in the smoke record for the rebuild under way.

## Overall

- RB-19 replaces RB-18's peak-only overflow guard with an exact exponent test on the norm. Every state whose parts are finite now returns a result. If the norm fits, rawNorm is that norm. If it overflows, rawNorm is `null` and the detail says the norm overflows. Nothing raises or warns, at the float max, one ulp past it, at the subnormal floor, with mixed parts or with signed zeros.
- Assigning the parts in place restores pre-RB-17 output byte for byte for every ordinary state, in both the full result and the evidence. Other modes are unchanged.

Verdict: PASS (RB-18 together with RB-19). The two notes are optional and can wait for a later round.
