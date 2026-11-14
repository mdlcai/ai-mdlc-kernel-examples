Commits reviewed: 9692ded8eff8b697bbc5b8fedc7343dc7d524ec1 (RB-17, round 15 follow-ups), 177e1976f44ad23f5cc149f7d55e4e4854c03e97 (C-25, release 0.7.0)
Previous round reviewed: round 15 (C-24 b9f93eb, F-27 8c5007a, F-28 fe1c622, F-29 42130b1, F-30 ce92d65, RB-16 de336a7)

# Reviewer Gate: Review Round 16 (RB-17 round 15 follow-ups, C-25 release 0.7.0)

- **Mechanism:** a fresh-context subagent reviewed `git show 9692ded` against artifacts/reviewer/round-15.md (M-R15-1 to M-R15-5, N-R15-1 to N-R15-4), SPEC.md §3 (formula params and result shape, SPEC.md:209-225), §5 F-27 to F-29 (SPEC.md:596-632, especially the state checks at :610, the ACs at :618-620 and :625-632) and DECISIONS.md ADR-48 (:395-410). It reviewed `git show 177e197` against INV-34 (invariants.json:35), the C-23 precedent (`git show --stat a5201bc`) and `git log --oneline a5201bc..177e197`. It ran these:
  - SDK: `cd packages/qai-sdk && python -I -m pytest -q -p no:cacheprovider` gave **470 passed, 1 skipped** in 76.40 s.
  - API: `cd services/api && python -m pytest -q -p no:cacheprovider tests/test_jobs_formula.py` gave **9 passed** in 42.16 s (against the already-running qai-test-pg and qai-test-valkey).
  - Web: `npx vitest run lib/expr.test.ts app/app/formula/_lib/form.test.ts components/results/formula-panels.test.tsx` gave **3 files, 93 tests passed**. `npx tsc --noEmit -p .` exits 0. `npx eslint --max-warnings 0 lib/expr.ts` exits 0.
  - A Python/TypeScript parity harness in the session scratchpad. It copies lib/expr.ts to `r16/expr.mts`, runs it with `node --experimental-transform-types` (Node 22.19), and runs `qai.expr.parse` through `python -I` on the same 31 hostile inputs. The inputs cover definition lines with leading spaces, tabs, `a=2x` without spaces, `\t a\t=\t2x`, an empty body, an unclosed paren, `1e400`, depth 34 inside a definition, non-BMP characters (U+1F600) before and inside the body and in a comment, a lone `\r`, U+2028, U+00A0, U+001C and U+FEFF. 29 of the 31 give the same message, column included. The 2 that differ are the U+001C and U+FEFF prefixes, which are N-R15-1.
  - An old-against-new comparison in the scratchpad. `git archive 9692ded^ packages/qai-sdk/qai` was extracted, and `run_formula` was run on 16 state and map cases with both trees under `python -I` (once with `-W error`, once with `-W ignore`). The comparison hashes the sorted JSON (with `ms` removed) and prints the checks.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual.
    - INV-1 passes (138 decisions, 83 commits since the plan, 0 undone, 27 human answers).
    - INV-10 (347 files) and INV-11 (567 files) pass.
    - INV-34 passes: "0.7.0: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree".
    - INV-2 fails: "8 unaccepted failure(s) ... report covers 2 screens, SPEC UI Surface lists 28 ... source changed since gitSha 3f35306 (53 file(s) ...): stale".
  - `npm run lint:py`: "All checks passed!" and "160 files already formatted".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval C-24 (GD-0128); pending review RB-17 and C-25; nothing in rework or blocked.
  - Greps: `git grep 0.6.0` outside CHANGELOG and VERSION, the 0.6.0 matches in package-lock.json, and the APP_VERSION and GIT_COMMIT lines of the local, gitignored `.env`.

  It did not run the full API suite, Playwright, the web baseline or the smoke test. It did not touch docker or any container and made no request to https://localhost:8443. It did not read FRONTEND-AUDIT.json or any tasks/*.output file. It ran govern.mjs only as `queue`, edited only this file and staged nothing. The scratchpad holds the harness files and is outside the repo.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-17 | 9692ded Round 15 follow-ups | **PASS** (1 minor, 3 notes) | M-R15-1, M-R15-3, M-R15-4 and M-R15-5 are resolved. M-R15-2 is resolved for the 1e±200 range it names. At the two float extremes (a subnormal peak, and a complex modulus that overflows), a finite state that can be normalised still fails `nonzero` (M-R16-1). |
| C-25 | 177e197 Release 0.7.0 | **PASS** (0 findings, 1 note) | It touches the same 12 files as C-23. INV-34 passes. No stale 0.6.0 default is left in tracked files. CHANGELOG 0.7.0 lists exactly the 7 non-governance, non-release commits in a5201bc..177e197. VERSION.md has the row. The gitignored local `.env` still pins APP_VERSION=0.6.0 for the coming rebuild (N-R16-4). |

Carried forward (human action): the queue (C-24 GD-0128 HUMAN_APPROVAL; RB-17 and C-25 pending review, and this round is their review evidence) and INV-2 (stale for 53 files, 8 failures, no `/app/formula` screen in the report).

## RB-17 (9692ded): finding by finding

| Check | Result | Evidence |
|---|---|---|
| M-R15-1: Python column | RESOLVED | expr.py:269 now passes the whole line and `m.start(2)` to `_parse_line`. `_Lexer` starts at `pos = start` (:104), so every token's start, and the "end" token at `len(text)`, counts from column 1 of the line. `a = 2x` gives "line 1, column 6". |
| M-R15-1: TS column | RESOLVED | expr.ts:355 passes `text.length - m[2].length`. DEFINITION (`([\s\S]*)$`, :172) always captures to the end of the line, so that offset equals Python's `m.start(2)`. |
| M-R15-1: leading whitespace, tabs, `name=` without spaces | PASS | Harness: `a=2x` gives column 4, `\ta = 2x` column 7, `\t a\t=\t2x` column 8, and `   a   =    x $ 1` column 15, the same in both. An empty body (`a =`, `a = \t `) gives "end of line: expression is empty" in both, as before the fix. |
| M-R15-1: non-BMP characters | PASS | The text before the body is ASCII plus BMP whitespace, so UTF-16 and code-point offsets agree. A non-BMP character in the body is always the first error ("unexpected character"), so no column is ever counted after one. U+1F600 before or inside the body, on a later line and in a comment gives the same column and character in both parsers. |
| M-R15-1: shared vectors | PASS | expr_vectors.json gains 3 reject vectors: column 6, "end of line: expected ')'" on line 2, and two leading spaces with `$` at column 9. Both suites run them: 470 SDK tests and 93 web tests pass. |
| M-R15-2: 1e200, 1e-200, 1e300·phase | RESOLVED | formula.py:512-517 divides by `peak = max|a(k)|` before taking the norm. Old tree: `1e200` raised a numpy overflow and failed `nonzero`, and `1e-200` failed `nonzero` (the old 1e-150 floor). New tree: both pass, with rawNorm 2.828e±200 and normalised drift 1.1e-16. The three parametrised tests (test_formula.py:353-359) also assert `allow_nan=False`. |
| M-R15-2: ordinary states unchanged? | PASS for the SPEC cases, NOTE otherwise | The SPEC AC phase state (4 qubits), the 10-qubit Gaussian at 100k shots, `k+1` and `0*k` give byte-identical result JSON in the old and new trees. A generic complex state (`sin(k)+i*cos(3*k)`, 6 qubits) differs in the last bits (N-R16-1). |
| M-R15-2: the extremes | **MINOR (M-R16-1)** | See the finding. |
| M-R15-2: rawNorm and the check value when the true norm overflows | NOTE (N-R16-2) | `exp(709)` on 4 qubits (‖a‖ ≈ 3.3e308) and `1e300*1e8*(1+i)` give rawNorm `null`, and the check value is `peak`, a lower bound. Both pass, and the JSON stays strict. |
| M-R15-3: E_0 finiteness | RESOLVED | formula.py:370-373 evaluates E_0, stores `E: null` if it is not finite, sets `stopped = {n: 0, why: "not finite"}` and skips the loop. Round 15's repro (`psi + 1`, energy `log(n)`) used to pass `finite` and come out inconclusive. Now it fails `finite` with "psi or E is not finite at n = 0", and the outcome is fail. |
| M-R15-3: bounded check and JSON strictness | PASS, 1 note | `bounded` is still computed from the trajectory (:400) and is unaffected for every n ≥ 1 stop. The early return at :406-408 keeps the non-finite path's check list (finite, bounded). Every case dumps with `allow_nan=False`. With |psi0| > bound and a non-finite E_0, `bounded` passes with a value above its threshold (N-R16-3). |
| M-R15-4: cross-tenant circuitId | RESOLVED | test_jobs_formula.py:168-190 creates a circuit in tenant A, submits state/circuit from tenant B with A's circuit id and gets 404 NOT_FOUND. A positive control with B's own circuit gets 202. This matches the AC at SPEC.md:631. |
| M-R15-5: steady-state physics formula | RESOLVED | test_jobs_formula.py:260-279 runs the AC's own formula `power*pulse*4*gin*gext/g^2` at pulse 10 and tolerance 1e-6 (SPEC.md:618 and :632). The job succeeds, `formula_agreement` fails, `ledger_conservation` passes, the worst relError lies in (0.05, 0.2) (the AC says ≈ 11 %), and the evidence is fail and verified. |
| Lint, types, invariants | PASS | ruff check and format, tsc and eslint are clean. INV-10 and INV-11 pass. |
| Governance | PASS (pending) | The queue lists RB-17 as pending review. BUILD-STATE.md names GD-0137. |

## C-25 (177e197): finding by finding

| Check | Result | Evidence |
|---|---|---|
| Version agreement | PASS | INV-34 passes across 10 files. The diff touches the same 12 files as C-23 (a5201bc): .env.example, CHANGELOG, VERSION, env.ts, apps/web/package.json, compose.yaml (4 defaults), package-lock.json (3 entries), package.json, both pyproject.toml files, `qai/__init__.py` and config.py. |
| No stale 0.6.0 | PASS | `git grep 0.6.0`, excluding CHANGELOG and VERSION, finds only history: DECISIONS.md:383 (ADR-46 "Releases 0.6.0"), reviewer rounds 12, 14 and 15, and security pass 7. The 8 package-lock.json matches are third-party (emoji-regex 10.6.0, decimal.js 10.6.0 and a `^10.6.0` engines range). ADR-48 (DECISIONS.md:410) says "Releases 0.7.0", which agrees. The gitignored local `.env` is not part of the commit (N-R16-4). |
| CHANGELOG scope | PASS | `git log --oneline a5201bc..177e197`, without governance records and C-25 itself, is RB-16 de336a7, C-24 b9f93eb, F-27 8c5007a, F-28 fe1c622, F-29 42130b1, F-30 ce92d65 and RB-17 9692ded. The 0.7.0 section lists exactly these 7 commits. a5201bc (C-23) is the 0.6.0 release, so leaving it out is correct. |
| CHANGELOG content | PASS, 1 caveat | The F-27 to F-30 and C-24 lines match round 15's findings (2,000 characters, 512 terms, never `eval`, migration 0012 with a guarded downgrade, the W11 panels). The RB-17 line says "huge or tiny amplitudes normalise". That is true for the range the tests cover, but not at the float extremes (M-R16-1). |
| VERSION.md | PASS | The row `0.7.0 · 2026-11-07 · Re-Entry R5 · W11 Formula Lab: F-27, F-28, F-29 (migration 0012), F-30 (ADR-48); RB-16; RB-17` matches the CHANGELOG date and the commit date (2026-11-07 14:49 −06:00). No tag is claimed yet, which is correct: the latest tag is v0.5.0. |
| Commit message | PASS | "Bumps every manifest, the lockfile and the version defaults to 0.7.0 (INV-34), adds the VERSION row and the CHANGELOG 0.7.0 section for F-27..F-30, RB-16..RB-17 and C-24" is accurate. |
| Governance | PASS (pending) | The queue lists C-25 as pending review. BUILD-STATE.md names GD-0138. |

## Round 15 notes: still open?

| Note | Status | Evidence |
|---|---|---|
| N-R15-1 Unicode digits and whitespace parity | **OPEN** | RB-17 did not change TOKEN or the definition regex. Harness: `\x1ca = x $` is a definition in Python (`\s` matches U+001C) and gives "column 8: unexpected character '$'". TS gives "only the last line can be the result". U+FEFF is the reverse. |
| N-R15-2 `grover_circuit_agreement` "not run" recorded as pass | **OPEN** | formula.py:176-178 is unchanged. |
| N-R15-3 coverage gaps (missing `mode` REQUIRED, inconclusive giving evidence fail at the API level, the runs branch of the 0012 guard) | **OPEN** | RB-17 added only the M-R15-4 and M-R15-5 tests. |
| N-R15-4 web baseline not measured for F-30 | **OPEN** | INV-2 is stale for 53 files since 3f35306. The report covers 2 screens against SPEC's 28. BUILD-STATE.md records the baseline as blocked. |

## Findings

- **M-R16-1 (minor, RB-17 / F-28): peak scaling still fails `nonzero` for finite, normalisable states at the two float extremes.**
  - Location: formula.py:512-517.
  - **Subnormal peak.** When the peak is below about 2.2e-308, `raw / peak` divides a complex array by a float, numpy's complex division overflows, and `unit` becomes `inf+nanj`. `scaled` is then nan, `nonzero` is false, and a numpy RuntimeWarning is emitted. Reproduced for `exp(-745)`, `1e-310`, `1e-320*k` and `1e-300*1e-23*(k+1)` on 2 qubits: each gives status fail with nonzero **fail**, a check value of 5e-324, 1e-310, 3e-320 or 4e-323 against a threshold of 0 and the text "the norm before normalising is > 0". The value printed is greater than 0 while the check says fail. Before RB-17 these failed with value 0.0, so the outcome is not a regression, but the case is in the class M-R15-2 named and the commit message claims fixed ("neither overflow nor underflow").
  - **Complex modulus overflow.** `np.abs` overflows when both components are finite but |a| > 1.8e308. `exp(709.7)*(1+i)` gives peak inf, nonzero **fail** with value 0.0 and rawNorm null.
  - Fix: scale by a power of two instead of by `peak`, for example `e = np.frexp(max(np.max(np.abs(raw.real)), np.max(np.abs(raw.imag))))[1]` and then `unit = np.ldexp(raw.real, -e) + 1j * np.ldexp(raw.imag, -e)`. Power-of-two scaling is exact, so it removes both edges and also makes ordinary results bit-identical to the old `raw / norm` path (N-R16-1). Add `exp(-745)` and `exp(709.7)*(1+i)` to the test_formula.py:353 parametrisation. If the edges are instead accepted, reword the CHANGELOG RB-17 line.
- **N-R16-1 (note, RB-17): ordinary complex states no longer give bit-identical results to the pre-RB-17 code.**
  - `{"mode":"state","source":"amplitudes","expression":"sin(k)+i*cos(3*k)","qubits":6,"seed":9}` now gives normalised drift 2.22e-16 (was 1.11e-16) and a TVD that differs in the 17th digit, so the result JSON and its evidence hash change.
  - The SPEC AC states (the phase state on 4 qubits, the Gaussian on 10 qubits, `k+1`) are byte-identical.
  - SPEC promises only same-seed reproducibility within a version (SPEC.md:337), and that still holds, so this is not a defect.
  - The power-of-two fix under M-R16-1 would remove the difference.
- **N-R16-2 (note, RB-17): when the true norm overflows, `nonzero.value` is the peak rather than the norm.**
  - Examples: `exp(709)` on 4 qubits (‖a‖ ≈ 3.3e308) and `1e300*1e8*(1+i)`.
  - The check passes with a value of 8.2e307 or 1.41e308, and rawNorm is `null`, as SPEC §3's null-for-non-finite rule requires.
  - The value is a sensible lower bound and the JSON is strict. The detail still reads "the norm before normalising", so a word such as "(at least)" would make the record exact.
- **N-R16-3 (note, F-28, made reachable at n = 0 by RB-17): `bounded` can pass with a value above its threshold.**
  - Example: `{"mode":"map","expression":"2*psi","energy":"log(n)","psi0":1e6,"bound":10,"steps":5}`. It stops at n = 0 because E_0 is not finite, and `bounded` passes with value 1e6 against threshold 10 and the text "|psi| stays within 10".
  - The outcome is still fail, because `finite` fails.
  - The cause is older than RB-17: psi0 is never compared with `bound`, and any not-finite stop marks `bounded` as passing. Fix: check |psi0| ≤ bound before the loop, or record `bounded` as skipped when the run stops for non-finiteness.
- **N-R16-4 (note, C-25 operations): the local `.env` still pins the old version for the rebuild.**
  - The gitignored `.env` has `APP_VERSION=0.6.0` and `GIT_COMMIT=5ce9f06`. compose.yaml reads `${APP_VERSION:-0.7.0}`, so the value in `.env` wins.
  - If the stack is rebuilt for the 0.7.0 smoke without editing `.env`, the images will be tagged `qai-api:0.6.0` and `qai-web:0.6.0`, and `/api/health` will report 0.6.0 and a stale commit. That repeats N-R14-4.
  - Set `APP_VERSION=0.7.0` and `GIT_COMMIT=177e197` (or the RB-18 head) before `docker compose up -d --build`, and quote the health line in the smoke record.

Counts: 0 blocker, 0 major, 1 minor, 4 notes.

## Carried forward (human action)

- **Queue:** C-24 (GD-0128) awaits human approval. RB-17 and C-25 are pending review, and this round is their review evidence.
- **INV-2:** fails with 8 unaccepted failures. The report is stale for 53 files and covers 2 of 28 screens, without `/app/formula`. It needs a quiet-host baseline run or an explicit human acceptance before the 0.7.0 ship banner (N-R15-4).

## Overall

- RB-17 fixes M-R15-1 in both parsers, and they agree on every whitespace, tab and non-BMP case tried. The only divergences left are the known N-R15-1 Unicode ones.
- M-R15-3, M-R15-4 and M-R15-5 are closed as the ACs are written, with positive controls.
- M-R15-2 is closed for the stated range. Scaling by the peak leaves two float-extreme edges (M-R16-1), and a power-of-two scale would close them and restore bit-identical ordinary results.
- C-25 is a clean release bump that follows the C-23 precedent. Its CHANGELOG matches the commit range exactly.

Verdict: PASS (RB-17, C-25). No FAIL lines. M-R16-1 suits a small RB-18. The notes can go with it or be deferred.
