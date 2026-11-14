Commits reviewed: b9f93ebd9545734be51f9906b57fd6f4fccdc421 (C-24), 8c5007a61760c49587f93146d1e77612bd49441b (F-27), fe1c622d0ce9ad17d7c9be2c2b2ccb64ab4e517f (F-28), 42130b11c1cd6178801450aab0b3efa15cd51c91 (F-29), ce92d65f32fc65dda700bc85cf8dd8af8057f624 (F-30); also re-checked: de336a770c3438bd99a201fc675bf204cbb3aa7b (RB-16, round 14 follow-ups) and the Formula Lab delivery evidence (API suite, web tests, e2e W11, web baseline)
Previous round reviewed: round 14 (RB-15 538f0f5, C-23 a5201bc release 0.6.0, security pass 7)

# Reviewer Gate: Review Round 15 (Formula Lab: C-24 docs, F-27 qai.expr, F-28 qai.formula, F-29 API, F-30 web; RB-16)

- **Mechanism:** a fresh-context subagent reviewed `git show` for each commit against SPEC.md §3 (formula params at SPEC.md:152-160, the result shape at :209-225), §5 F-27 to F-30 (SPEC.md:596-656), W11 (:722-726), the test matrix (:916-919), DECISIONS.md ADR-48 (:395-410) and ARCHITECTURE.md §2.10 (:137-145) and §9. For RB-16 it compared against artifacts/reviewer/round-14.md (M-R14-1, N-R14-5). It checked the delivery claims against the logs in the session scratchpad (`formula/api-suite.log`, `formula/web-test.log`, `formula/web-checks.log`, `formula/e2e-w11.log`, `formula/baseline.log`, `formula/baseline2.log`, `formula/baseline3.log`). It ran these:
  - SDK: `python -I -m pytest -q -p no:cacheprovider tests/test_expr.py tests/test_formula.py` gave **130 passed** in 15.84 s.
  - Hostile parser inputs through `python -I`, on qai.expr: 2000 `(`, 1999 `-`, `x^` and `x+` chains, `'1'*1999`, `1e400`, `9*400`, `1e-400`, `__import__`, `x.real`, `x[0]`, `lambda`, `nan`/`inf` names, Unicode digits and control whitespace, and a 256-deep left chain under `sys.setrecursionlimit(300)`. It also timed the worst-case parses.
  - Worst-case CPU and result size for every F-28 mode through `python -I` (the table under F-28). It reproduced the candidate findings: amplitudes `1e200` and map energy `log(n)`.
  - Web: `npx vitest run lib/expr.test.ts app/app/formula/_lib/form.test.ts components/results/formula-panels.test.tsx` gave **3 files, 90 tests passed**. `npx tsc --noEmit -p .` exits 0. `npx eslint --max-warnings 0` on every .ts/.tsx file in ce92d65 exits 0. `npx prettier --check` on every file in ce92d65 reports "All matched files use Prettier code style!".
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual. INV-1 now **passes** (132 decisions, 81 commits since the plan, 0 undone). INV-10 (no pickle/eval/exec, 343 files) and INV-11 (no dangerouslySetInnerHTML, 563 files) pass. INV-2 fails: "8 unaccepted failure(s): landing/load, landing/cwv, invite-accept/load ... report covers 2 screens, SPEC UI Surface lists 28 ... source changed since gitSha 3f35306 (44 file(s) ...): stale".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval C-24 (GD-0128); pending review F-27, F-28, F-29 and F-30; nothing in rework or blocked.

  It did not run the full API suite, Playwright, the web baseline or the smoke test (the host is contended). It did not touch docker, did not read FRONTEND-AUDIT.json, ran govern.mjs only as `queue`, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-16 | de336a7 Round 14 follow-ups | **PASS** (0 findings) | Resolves M-R14-1. ADR-47 and W10.3 were amended first (docs first). The API half and the web half of N-R14-5 are closed. |
| C-24 | b9f93eb Formula Lab docs | **PASS** (0 findings) | SPEC §3, §5 F-27 to F-30, W11, the test matrix, ADR-48 and ARCH §2.10 agree with each other and with the code. It awaits human approval (GD-0128). |
| F-27 | 8c5007a qai.expr | **PASS** (1 minor, 1 note) | No eval, exec or compile. Parsing is bounded, iterative where it matters, and fast (worst case 4.3 ms). Errors inside a definition line report a column relative to the body (M-R15-1). Python accepts Unicode digits and whitespace that the TS mirror rejects (N-R15-1). |
| F-28 | fe1c622 qai.formula | **PASS** (2 minors, 1 note) | The four modes and the ADR-46 outcome rules behave correctly. The worst case is 8.5 s against the 120 s job timeout, and results stay under 90 KB. A valid amplitudes state with huge entries fails `nonzero` (M-R15-2). The map `finite` check skips E_0 but still claims "every psi and E is finite" (M-R15-3). |
| F-29 | 42130b1 API, migration 0012 | **PASS** (2 minors, 1 note) | Submit-time parsing, per-mode keys, the circuit lookup inside tenant_session (404 across tenants) and the runner's evidence rule are correct. The 0012 downgrade restores the 0011 constraints exactly. Two F-29 ACs are not exercised as written (M-R15-4, M-R15-5). |
| F-30 | ce92d65 web Formula Lab | **PASS** (0 findings beyond M-R15-1) | All text renders as React text, with no raw-HTML sink. expr.ts passes every shared vector. Form limits mirror §3. eslint, prettier, tsc and vitest are clean. |
| Delivery evidence | API suite, web tests, e2e W11, baseline | **JUSTIFIED, with qualifications** (1 note) | 309 API passed, 285 web passed and W11 2/2 match their logs. The baseline was never measured, and the record should say so (N-R15-4). |

Carried forward (human action): the queue (C-24 GD-0128 HUMAN_APPROVAL; F-27 to F-30 pending review, and this round is their review evidence) and INV-2 (stale, 8 failures). M-R12-0 is resolved: INV-1 now passes.

## RB-16 (de336a7): finding by finding

| Check | Result | Evidence |
|---|---|---|
| M-R14-1: Compare hint | RESOLVED | orb-panels.tsx:750 now reads "Queries to reach the QAI best / classical evaluations", which is the hindsight `queryRatio` that SPEC §3 defines. The ADR-47 amendment and the W10.3 rewording come first in the same commit. |
| N-R14-5: wording recorded in ADR-47 / W10.3 | RESOLVED | The "optimum/best" rule is now in ADR-47 and W10.3. |
| N-R14-5: legacy result asserts `data-found-at` absent | RESOLVED | The orb-panels test now asserts both the hint text and that `data-found-at` is absent. |
| N-R14-5: missed-optimum fixture outcome | RESOLVED | The fixture now reports fail, which a real missed-optimum run would. |
| N-R14-5: API half (N-R13-3) | RESOLVED | test_jobs_orb.py adds the `foundAt` and `queryRatio` asserts. |

## C-24 (b9f93eb): finding by finding

| Check | Result | Evidence |
|---|---|---|
| §3 params and result shape | PASS | SPEC.md:152-160 lists per-mode keys, defaults and limits that match FORMULA_KEYS (schemas/jobs.py), the SDK constants and form.ts. SPEC.md:209-225 matches `run_formula` output, including that non-finite values are stored as null. |
| ACs F-27 to F-30, W11, test matrix | PASS | SPEC.md:596-656, :722-726 and :916-919 each name a test that exists, apart from the two gaps under F-29. |
| ADR-48 / ARCH §2.10 | PASS | Submit-time parsing runs inline, not through run_bounded. The ADR justifies this with the parse-cost bound, and the timings in this round confirm it (worst case 4.3 ms, within the SPEC perf row of < 5 ms). |

## F-27 (8c5007a): finding by finding

| Check | Result | Evidence |
|---|---|---|
| INV-10: no eval | PASS | expr.py has no eval, exec, compile or getattr on user input. It lexes with one TOKEN regex (:97-100), parses by recursive descent into tuples (:138-227) and evaluates through a fixed NumPy dispatch table inside `np.errstate(all="ignore")` (:361-372). `__import__`, `x.real`, `x[0]`, `lambda`, `nan` and `inf` are all rejected as unknown names or unexpected characters. |
| Bounded parsing | PASS | Depth is checked before each recursion. 2000 `(` and 1999 `-` both stop at column 34 (MAX_DEPTH 32). An `x^` chain stops at column 67. An `x+` chain hits MAX_NODES 512 at column 514. No input raised RecursionError, including a 256-deep left chain (511 nodes) under `sys.setrecursionlimit(300)`. MAX_CHARS 2000 and MAX_LINES 13 are enforced before lexing. |
| Literal bounds | PASS | `'1'*1999`, `1e400` and `9*400` give "number is too large". `1e-400` is accepted as 0. |
| Overflow at evaluation | PASS | `exp(exp(exp(x)))`, `10^10^10^x` and `1/0` return inf, and `(-8)^(1/3)` returns nan. None of them raise, so the F-28 finite checks catch them. |
| Parse timing | PASS | The worst cases take 3.8 ms (2000 `(`) and 4.3 ms (a 1999-character rejected chain). The lexer is O(n²) because of `text[pos:].strip()`, but that is negligible at 2000 characters. |
| Error locations | **MINOR** | M-R15-1. |
| Parity with expr.ts | PASS, 1 note | Every accept and reject vector in tests/data/expr_vectors.json gives the same canonical form, node count, value (to 1e-12) and message in both parsers. The only divergence found is the Unicode and whitespace one (N-R15-1). |

## F-28 (fe1c622): finding by finding

| Check | Result | Evidence |
|---|---|---|
| Four modes against the ACs | PASS | test_formula.py covers objective, physics, map and state (amplitudes and circuit), with the SPEC AC values. |
| ADR-46 outcome rules | PASS | A physics-check failure gives fail, and a comparison-check failure gives inconclusive. The repros below show both. |
| CPU and size bounds | PASS | Objective, 10 qubits, budget 4096, ~511-node formula: 8.4 s, 90 KB. Objective 2-D, same limits: 8.0 s. Map, 1000 steps, large formula and energy: 3.8 s, 71 KB. Physics, 32 points, large formula: 3.5 s. Amplitudes, 10 qubits, 100k shots: 0.03 s. Circuit, 10 qubits, ~1990 ops, 64 points, 100k shots: 8.5 s, 57 KB. All are well under the 120 s timeout and about the ~90 KB that the ADR states. |
| Evidence inputs | PASS | The inputs hold the raw params, including the source text. The sha256 therefore changes with whitespace even when the outputs are the same, which is what SPEC says. |
| Amplitudes with huge entries | **MINOR** | M-R15-2. |
| Map E_0 | **MINOR** | M-R15-3. |
| Objective early stop | NOTE | N-R15-2. |

## F-29 (42130b1): finding by finding

| Check | Result | Evidence |
|---|---|---|
| Submit validation | PASS | In schemas/jobs.py FormulaParams, the `_keys` before-validator returns UNKNOWN_FIELD for keys outside FORMULA_KEYS for the mode. The `_modes` after-validator applies the SDK limits and parses with qai.expr, mapping errors to INVALID or REQUIRED with the field path. `_check_formula_circuit` covers TOO_MANY_QUBITS, INVALID, NO_PARAMETERS and the target length. Every code is in SPEC §7. |
| Tenant isolation | PASS | In services/jobs.py, `formula_needs_circuit` and `projects_repo.get_circuit` (filtered on `Circuit.tenant_id == tenant_id`) run inside `tenant_session`, and a miss returns 404 "Circuit not found". RLS backs this up. |
| Runner | PASS | In runners.py, `run_formula` re-parses in the child process, and evidence passed is `status == "pass"`. |
| Migration 0012 | PASS, 1 note | It drops and re-adds the CHECKs in the same way as 0011. The downgrade has a `DO $$` guard and restores exactly the 0011 constraints. test_migrations.py:200-239 covers upgrade, refusal to downgrade while a formula job exists, a clean downgrade and re-upgrade. Only the jobs branch of the guard is tested (N-R15-3). |
| Tests against the F-29 ACs | **MINOR** | M-R15-4 and M-R15-5. |

## F-30 (ce92d65): finding by finding

| Check | Result | Evidence |
|---|---|---|
| INV-11: text rendering | PASS | There is no dangerouslySetInnerHTML, innerHTML, eval or `new Function` in app/app/formula, components/results/formula-panels.tsx or lib/expr.ts. The canonical form goes through `CodeBlock` (`<code>{code}</code>`), and the reason through `<p>{r.report.reason}</p>` (formula-panels.tsx:463-476). |
| Parse as you type | PASS | lib/expr.test.ts runs every shared vector: canonical form, node count, value within 1e-12, the canonical fixed point and the exact messages. The preview samples 64 points and says when values are not finite (FormulaForm.tsx:112-132). |
| Form limits | PASS | form.ts mirrors §3, including the variable glossary and checking the circuit target bitstring length against the circuit. |
| Panels | PASS | The order is Report, Formula, the mode panels, then Validate, and each chart has a table alternative. The W11 spec selects them by region name and data attributes. |
| Lint and types | PASS | eslint, prettier, tsc and vitest (90 tests) are clean on the committed files. |

## Delivery evidence

| Claim | Result | Evidence |
|---|---|---|
| API suite 309 passed | JUSTIFIED | api-suite.log: "309 passed in 683.02s (0:11:23)", exit=0 (14:01, after F-29). |
| Web typecheck and vitest 285 | JUSTIFIED | web-test.log: next typegen and tsc, then "Test Files 29 passed (29), Tests 285 passed (285)" (14:15). |
| Web lint | JUSTIFIED as committed | web-checks.log (14:01) shows `prettier --check` failing on 13 files, including FormulaForm.tsx and formula-panels.fixtures.json. That run is before ce92d65 (14:27). This round's prettier and eslint checks on the committed F-30 files pass. The other 11 files in that log are outside F-30. |
| e2e W11 | JUSTIFIED | e2e-w11.log: "[desktop] ... [mobile] ... 2 passed (2.5m)". w11-formula.spec.ts covers W11.1 to W11.4 as SPEC lists them. |
| Web baseline | **NOT MEASURED** | N-R15-4. |

## Findings

- **M-R15-1 (minor, F-27 and F-30): errors inside a definition line report a column relative to the body after `=`, not to the line.**
  - `a = 2x` on line 1 reports "line 1, column 3", but the `x` is at column 6.
  - Both parsers pass the body alone to the line parser: expr.py:263 and :272 (`_parse_line(body, ...)` with `m.group(2)`), and expr.ts:316 and :354 (`parseLine(m[2], ...)`).
  - No shared vector covers an error inside a definition, so the parity tests could not catch it.
  - Fix: pass the body's start offset (`m.start(2)` / `m[1]`-derived index) into the line parser in both files, and add a reject vector.
- **M-R15-2 (minor, F-28): a valid amplitudes state with huge finite entries fails `nonzero`.** At formula.py:511-512, `np.linalg.norm(raw)` overflows to inf for entries near 1e200. Then `nonzero = math.isfinite(norm) and ...` is false. Reproduced: `{"mode":"state","source":"amplitudes","expression":"1e200","qubits":3,"seed":1}` gives finite pass, nonzero **fail** ("the norm before normalising is > 0"), status **fail**, with a numpy overflow RuntimeWarning. The state can be normalised. Fix: scale by `max|raw|` before taking the norm, or report a distinct reason.
- **M-R15-3 (minor, F-28): the map `finite` check does not check E_0, yet claims it does.** formula.py:370 records `energy_of(psi0, 0)` in the trajectory, but the loop at :372-376 checks only `nxt` and `e_next`. Reproduced: `{"mode":"map","seed":1,"expression":"psi + 1","energy":"log(n)","psi0":1,"steps":5}` stores `trajectory[0].E = null` (log 0 = -inf), while `finite` **passes** with "every psi and E is finite". Fix: check E_0 before the loop and stop at n = 0.
- **M-R15-4 (minor, F-29): a cross-tenant `circuitId` on a formula job is not tested.** The F-29 AC asks for a 404 on a cross-tenant circuit. test_jobs_formula.py:160-165 tests only a cross-tenant project. The code path is correct by construction (`get_circuit` filters on tenant inside `tenant_session`) and is shared with simulate, which test_jobs.py:209 covers. Fix: add a state/circuit submit with another tenant's circuit id and assert 404 NOT_FOUND.
- **M-R15-5 (minor, F-29): the failing-formula AC is exercised with a different formula.** The AC names the physics steady-state approximation (about 11 % off the ledger, so fail). test_jobs_formula.py:229-232 uses the objective `log(x - 0.5)`, which fails on the finite check. That checks the evidence-fail path but not the comparison against the ledger at the API level. The SDK test covers the physics case. Fix: run the AC's physics formula, or amend the AC (docs first).
- **N-R15-1 (note, F-27 parity): Python accepts Unicode digits and whitespace that JS rejects, and the reverse for U+FEFF.** Python `\d` and `\s`/`str.strip` match Unicode, so `٣*x` and fullwidth `３*x` parse as `3 * x`, and `x\x1c+1` parses. JS `\d` is ASCII-only, and JS `trim` keeps `\x1c` but strips `﻿`, which Python rejects as an unexpected character. The locations are expr.py:98 (and the `strip` in the lexer) and expr.ts:133. The impact is small: the server is authoritative and the web would show an error the API does not. Fix: use `re.ASCII` or `[0-9]` and an explicit whitespace set in both files, and add vectors.
- **N-R15-2 (note, F-28): when the objective is not finite, `grover_circuit_agreement` is recorded as pass with "not run".** formula.py:176-178. The outcome is still fail, which is correct, but a check that did not run should not read as a pass. A skipped status, or leaving the check out, would be clearer.
- **N-R15-3 (note, F-29 and the tests): small coverage gaps.**
  - A missing `mode` returning REQUIRED is not asserted. The missing-seed test checks the field but not the code.
  - An inconclusive result giving evidence fail is not tested at the API level.
  - Only the jobs branch of the 0012 downgrade guard is tested.
- **N-R15-4 (note, delivery evidence): the web baseline was never measured for F-30.**
  - baseline.log: baseline-prep died with `ECONNRESET`.
  - baseline2.log: exit 2, because a previous report exists and no mode was given.
  - baseline3.log (`--screen app,jobs,job-detail,validation`): "baseUrl https://localhost:8443 unreachable", so nothing was measured.
  - INV-2 therefore stays stale (gitSha 3f35306, 44 changed files including the formula screen), and SPEC's UI Surface now lists 28 screens. The record should say "not measured", not "baseline run". `/app/formula` needs a quiet-host run before any release.

Counts: 0 blocker, 0 major, 5 minor, 4 notes. M-R12-0 is resolved, since INV-1 passes.

## Carried forward (human action)

- **Queue:** C-24 (GD-0128) awaits human approval. F-27, F-28, F-29 and F-30 are pending review, and this round is their review evidence.
- **INV-2:** fails with 8 unaccepted failures, and the report is stale for 44 files. It needs a quiet-host run covering `/app/formula`, or an explicit human acceptance (N-R15-4, and N-R14-3 before it).
- **M-R12-0:** resolved. INV-1 passes, with 27 human answers recorded.

## Overall

- RB-16 closes M-R14-1 and every part of N-R14-5, docs first.
- C-24 is a coherent specification, and the code matches it.
- F-27 satisfies the security priorities: no eval, depth and node limits checked before recursion, literal bounds, overflow returned as values, and parse cost far below the ADR-48 bound. Its one defect is the column in definition-line errors, which is shared with expr.ts (M-R15-1). Parity is otherwise exact on the shared vectors.
- F-28's modes, outcome rules and CPU and size bounds hold. Two edge cases report a wrong check result (M-R15-2, M-R15-3).
- F-29's tenant isolation, submit-time parsing, runner and the reversible migration 0012 are correct. Two ACs are tested less directly than they are written (M-R15-4, M-R15-5).
- F-30 renders everything as text and is clean on every web check.
- Delivery claims hold, except that the web baseline was not measured (N-R15-4).

Verdict: PASS (RB-16, C-24, F-27, F-28, F-29, F-30). No FAIL lines. The five minors suit one RB follow-up. M-R15-5 may instead be met by amending the AC (docs first).
