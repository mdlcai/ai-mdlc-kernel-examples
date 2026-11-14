Commits reviewed: 1d4aeff37ac93fbacad08dc1465abe827bacc6d7 (RB-9), e9113ce29d6e57fb43aa7f201a0d8afec77c548a (C-19), 8e71d009e2971ba4658123e75e82a2c0f43f704f (RB-10)
Previous round reviewed: bb22a29 (round 10, F-21..F-23)

# Reviewer Gate: Review Round 11 (RB-9 round 10 follow-ups, C-19 release 0.5.0, RB-10 e2e selectors)

- **Mechanism:** a fresh-context subagent reviewed each commit on its own with `git show <sha>`. It checked RB-9 finding by finding against `artifacts/reviewer/round-10.md`, and checked all three commits against SPEC.md §3 `surrogate`, §5 F-21 and §7, DECISIONS.md ADR-41/ADR-42, invariants.json INV-34 and the CLAUDE.md conventions. It ran these:
  - API tests: `tests/test_jobs_surrogate.py` against the running qai-test-pg and qai-test-valkey containers.
  - Web checks: `npx vitest run app/app/surrogate` and `npx tsc --noEmit`.
  - `npm run lint:invariants`.
  - `curl /api/health` on the live stack.
  - Two throwaway probes in the session scratchpad, outside the repo:
    - `validate_params("surrogate", …)` called directly, to check error codes, error order and how long a huge list takes;
    - `simulate_surrogate` on 3 qubits with d = 2, 4 and 6 at spans ±π, ±π/4, ±0.25, ±0.05 and ±0.01 (seed 7, tolerance 1e-6), to test the M-R10-1 wording.

  It did not run Playwright (`npm run e2e` was not among the permitted commands), so RB-10 was reviewed statically. It did not touch docker, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-9 | 1d4aeff Review round 10 follow-ups (ADR-42) | **PASS** (1 minor, 3 notes) | M-R10-2, M-R10-3, N-R10-1, N-R10-6 and N-R10-7 are closed correctly. M-R10-1 is closed in ADR-41, the SDK docstring and the form note, but not in SPEC.md, although ADR-42 says SPEC was qualified (M-R11-1). N-R10-2 is closed only in its SPEC part. |
| C-19 | e9113ce Release 0.5.0 | **PASS** (1 minor) | Every manifest, the lockfile and all 7 version defaults agree on 0.5.0. INV-34 passes and the live `/api/health` reports 0.5.0. The CHANGELOG 0.5.0 section leaves out RB-7, RB-8 and RB-9, and repeats the unqualified exactness claim (M-R11-2). |
| RB-10 | 8e71d00 e2e selectors | **PASS** | The W5 `exact: true` fix is correct and needed. The W8 drawer branch copies the W9 pattern. |

## RB-9 (1d4aeff): round 10 follow-ups

| Finding | Result | Evidence |
|---|---|---|
| M-R10-1 exactness claim | PARTIAL (M-R11-1, N-R11-3) | Qualified in ADR-41 (DECISIONS.md:310), the SDK docstring (packages/qai-sdk/qai/surrogate.py:6-8) and the form note (apps/web/app/app/surrogate/SurrogateForm.tsx:293-297). SPEC.md was not changed: the AC at SPEC.md:463 still says "a surrogate fitted on more than 3^d samples predicts held-out distributions to TVD ≤ 1e-6", with no span qualifier. Yet ADR-42 (DECISIONS.md:328) lists "SPEC" among the qualified documents and says "the SPEC acceptance test stays at ±π", which the SPEC text does not say. No small-span test was added. |
| M-R10-2 op-list codes | PASS | `Field(min_length=1, max_length=6)` was removed. The model validator now raises `params.parameters` NO_PARAMETERS for `[]` and TOO_MANY_PARAMETERS for more than 6 entries (services/api/qai_api/schemas/jobs.py:160-165). That matches SPEC.md:150. The probe confirmed both codes, and `[0]*7` gives TOO_MANY_PARAMETERS. The test pins both cases (test_jobs_surrogate.py:103-113). §7 has no field-code enum, and both codes were already emitted by `check_circuit_params` (schemas/jobs.py:198-202), so no new code is introduced. |
| M-R10-3 `circuitId` on the circuit field | PASS | The free-standing alert paragraph is gone. SurrogateForm reports `err('circuitId')` (client or server) through `onCircuitError` (SurrogateForm.tsx:99-102). SurrogateView keeps it in state and passes it as `error` on the `sur-circuit` Field (SurrogateView.tsx:55, 184, 226). Field wires `aria-describedby="sur-circuit-error sur-circuit-hint"` and `aria-invalid` to the Select, and renders the message as `role="alert"` (components/ui/Field.tsx). Effects: the first effect pushes each change; the second clears on unmount. React runs unmount cleanups before the new instance's mount effects, so a key change (another circuit or version) or a loading state clears a stale error and cannot overwrite the new form's error. `setCircuitError` is stable, so neither effect loops. `shownOnForm('circuitId')` stays true, so the message is not also shown in the banner. The Vitest test covers report, no in-form duplicate, disabled submit and unmount clear (SurrogateForm.test.tsx:42-52). |
| N-R10-1 regex leak | PASS | The `pattern=` was replaced by an explicit length and alphabet check that returns "Must be a bitstring of 0 and 1, highest qubit first" with code INVALID (schemas/jobs.py:166-167). The test asserts that no `^` or `pattern` appears in the message, for `0a`, `""` and 11 characters (test_jobs_surrogate.py:107-113). The circuit-length rule is unchanged (schemas/jobs.py:205-206). |
| N-R10-2 code by message text | PARTIAL (N-R11-2) | SPEC.md:150 now names DUPLICATE for a repeated index. The `"no rotation"` / `"twice"` substring mapping (schemas/jobs.py:198-202) is unchanged, and ADR-42 neither fixes it nor lists it among the accepted notes. |
| N-R10-6 warning untested | PASS | `SurrogateForm.test.tsx:30-40` renders d = 6. At 500 samples it asserts "Underdetermined: 400 training samples for 729 features" (holdout 100, train 400, which is correct). The w9 comment now matches its assertion (w9-surrogate.spec.ts:81: 16 samples train 13 > 9 features). |
| N-R10-7 live region | PASS | There is now a persistent `<div role="status" aria-live="polite">` with the warning rendered conditionally inside it (SurrogateForm.tsx:281-292). The test asserts the same status node before and after the change (SurrogateForm.test.tsx:39). `aria-live` repeats the implicit value of `role="status"`, which is harmless. The div is unstyled, so the components/ui rule holds. |
| Docs consistency (ADR-41/42, SPEC, SDK) | PARTIAL (M-R11-1) | ADR-41, ADR-42, the SDK docstring and the form note agree with one another. SPEC:463 does not. The bound "1e-6 to 1e-5" is slightly low at the top end (N-R11-3). |
| Regressions | PASS (N-R11-1) | `test_jobs_surrogate.py`: **6 passed in 10.0 s**. Vitest app/app/surrogate: **2 files, 9 tests passed**. `tsc --noEmit` exits 0. Moving the list and target checks into the after-validator changes when errors are reported (N-R11-1). |

## C-19 (e9113ce): release 0.5.0

| Check | Result | Evidence |
|---|---|---|
| Manifests and lockfile | PASS | package.json, apps/web/package.json, package-lock.json (root, `""` and `apps/web` entries), packages/qai-sdk/pyproject.toml and services/api/pyproject.toml are all 0.5.0. A grep finds no 0.4.1 left outside third-party lockfile entries (`@eslint/plugin-kit`, `levn`, `json-schema-traverse`), CHANGELOG/VERSION history and BUILD-STATE/REPORT/DECISIONS prose. |
| Version defaults | PASS | `.env.example` APP_VERSION, compose.yaml (4 places: python env, qai-api image, qai-web image, web env), apps/web/lib/env.ts, services/api/qai_api/config.py and `qai.__version__` are all 0.5.0. |
| INV-34 | PASS | `npm run lint:invariants`: "0.5.0: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree". Totals: 34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual. The only FAIL is INV-2 (missing FRONTEND-AUDIT.json), the known pre-existing state. |
| Live stack | PASS | `/api/health` returns `{"status":"ok","version":"0.5.0","commit":"e9113ce",…}` and every check is ok. |
| VERSION.md row | PASS (see M-R11-2) | VERSION.md:15 has 0.5.0, 2026-11-02, Re-Entry R3, F-21..F-23, migration 0010 and ADR-41. It does not mention RB-9/ADR-42. The 0.4.0 row did list its RB items and ADRs. |
| CHANGELOG vs `git log v0.4.1..e9113ce` | PARTIAL (M-R11-2) | The log since tag v0.4.1 (e558094) holds RB-7 05d558b, C-14 7916f64, C-15 ce0900b, RB-8 2c772ef, C-16 467de0e, C-17 322ae67, C-18 9a8a300, F-21 cd1bffb/2360614, F-22 18633ba, F-23 bb22a29 and RB-9 1d4aeff. The 0.5.0 section (CHANGELOG.md:5-12) lists F-21, F-22, F-23 and C-15 to C-18. It omits RB-7, RB-8 and RB-9. C-14 changes only REPORT.md, so leaving it out is acceptable. |

## RB-10 (8e71d00): e2e selectors

| Check | Result | Evidence |
|---|---|---|
| W5 "Simulation" checkbox | PASS | KIND_LABEL now contains both 'Simulation' and 'AI Simulation' (governance/_lib/format.ts:8, 15), so a substring role-name match finds two checkboxes and `.check()` fails in strict mode. `exact: true` (w5-release.spec.ts:76-77) is the correct fix. The uncheck loop's labels (Reference, Benchmark, Algorithm, Evolution) are each unique. Pipeline and AI Simulation are not among DEFAULT_KINDS (GovernanceView.tsx:35), so the release still requires exactly simulation and validation, as W5.2 asserts. |
| W8 nav link in the drawer | PASS | The new branch (w8-pipeline.spec.ts:28-40) copies w9-surrogate.spec.ts:46-55. It waits for the projects to load, opens the "Navigation menu" drawer when no visible link exists, asserts the href, closes with Escape and checks `aria-expanded="false"`. On desktop it asserts the tab link directly. The extra "Loading projects" wait before the W8.1 one is redundant but harmless. |
| Execution | NOT RUN | The commit reports "Full e2e: 44 passed", and BUILD-STATE.md:9 records e2e 44/44. This round did not run Playwright. |

## Findings

- **M-R11-1 (minor, RB-9): SPEC.md was not qualified, although ADR-42 says it was.**
  - **Claim:** DECISIONS.md:328 says the exactness claim is qualified in "ADR-41, SPEC, SDK docstring, form note", and that "the SPEC acceptance test stays at ±π".
  - **Actual:** RB-9 changed only SPEC.md:150. The F-21 AC at SPEC.md:463 still reads "For random circuits on 1–4 qubits with 1–3 angles (hypothesis), a surrogate fitted on more than 3^d samples predicts held-out distributions to TVD ≤ 1e-6", with no span.
  - **Why it matters:** read literally, the AC is false inside its own scope. This round's probe at d = 2 (in scope) gave a held-out max TVD of 1.32e-6 at ±0.05 and 1.58e-6 at ±0.01, both with `determined: true`. SPEC is the source of truth (CLAUDE.md Guardrails).
  - **Fix:** add "over the full span ±π (the default)" to SPEC.md:463. Optionally add the same qualifier to §5 F-21 or the F-23 note.
- **M-R11-2 (minor, C-19): the CHANGELOG 0.5.0 section is incomplete and repeats the claim RB-9 qualified.**
  - **Missing entries:** the section (CHANGELOG.md:5-12) has no entry for RB-7 (05d558b, the wordmark tagline collapses below 1280 px, N-R7-1), RB-8 (2c772ef, the 177-line HeroConsole CSS removed and the Artwork props cleaned up) or RB-9 (1d4aeff). RB-9 changes the wire: `params.parameters` now returns NO_PARAMETERS / TOO_MANY_PARAMETERS where it returned TOO_SHORT / TOO_LONG, and the `params.target` message changes. Earlier sections list their RB items (0.4.0 lists RB-5 and RB-6), and VERSION.md:15 also omits RB-9/ADR-42.
  - **Repeated claim:** the F-21 bullet, written 18 s after RB-9, says "exact when training samples exceed 3^d" with no span qualifier. That repeats the M-R10-1 wording that ADR-42 had just qualified.
  - **Fix:** add RB-7, RB-8 and RB-9 (ADR-42) bullets, qualify the F-21 bullet to ±π, and mention ADR-42 in the VERSION row.
- **N-R11-1 (note, RB-9): the op-list and target checks now run only after every field passes.** They moved from `Field(...)` constraints into `model_validator(mode="after")`, so they report only when every field is valid, and only the first failure is reported.
  - **Probe:** `{"parameters": []}` without `seed` returns only `params.seed` REQUIRED. Before RB-9 the response also carried the `parameters` error.
  - **Probe:** `parameters` is now unbounded at the schema level. Pydantic validates the whole list before the length check, so a 200,000-entry list took 8 ms (bounded by Caddy's 1 MB `request_body` limit, infra/caddy/Caddyfile:23-24).
  - **Impact:** the spec is still met (each error has the right path and code), and the web client checks these values first. One option is to keep `max_length=4096` as a cheap backstop and translate the pydantic `too_long`/`too_short` codes into the §3 codes inside `validate_params`.
- **N-R11-2 (note, RB-9): N-R10-2 is only half closed.** SPEC now names DUPLICATE, but `check_circuit_params` still picks NO_PARAMETERS / TOO_MANY_PARAMETERS and DUPLICATE / NOT_PARAMETRISED by substring-matching SDK messages (`"no rotation"`, `"twice"`, schemas/jobs.py:198-202). ADR-42's Consequences list accepts N-R10-3/4/5 but says nothing about this. Either give `CircuitError` a code from `parameter_slots`, or record the matching as accepted in ADR-42.
- **N-R11-3 (note, RB-9): the stated bound of "1e-6 to 1e-5" is slightly low, and no test pins it.** The probe (3 qubits, seed 7, `determined: true` throughout) measured these held-out max TVDs:

  | d | Span | Hold-out max TVD |
  |---|---|---|
  | 4 | ±π/4 | 9.9e-6 |
  | 4 | ±0.25 | 1.39e-5 |
  | 6 | ±π/4 | 1.66e-5 |

  Every span ±π case gave ≤ 2.1e-10. Every narrow-span case is under the advised 1e-4 tolerance, with a margin of 6x or more, so the form's advice (SurrogateForm.tsx:296-297) holds. But ADR-41, the docstring and the note say "near 1e-6 to 1e-5" / "near 0.00001", and round 10's suggested small-span test was not added.
- Carried forward: N-R8-6 (no web baseline evidence, INV-2 pre-existing) is still open and out of scope. N-R10-3, N-R10-4 and N-R10-5 are accepted in ADR-42. Playwright and the smoke test were not executed in this round.

Counts: 0 blocker, 0 major, 2 minor, 3 notes.

## Overall

**PASS.** RB-9 closes the round 10 code findings correctly:
- The API returns the SPEC §3 codes NO_PARAMETERS / TOO_MANY_PARAMETERS at `params.parameters`, and the target message no longer exposes a regex. The 6 API tests pass.
- The `circuitId` error now sits on the circuit select with `aria-invalid` and `aria-describedby`. The effects push and clear it without leaks or loops across re-keying and unmount.
- The underdetermined warning sits in a persistent live region, and a component test renders it.

C-19 brings every manifest, the lockfile and all version defaults to 0.5.0 (INV-34 passes, and the live health check reports 0.5.0). RB-10's selector fixes are correct.

The two minors are documentation accuracy issues and neither blocks the gate:
- M-R11-1: the SPEC AC was not qualified, although ADR-42 says it was.
- M-R11-2: the CHANGELOG 0.5.0 section omits RB-7, RB-8 and RB-9 and repeats the unqualified exactness claim.

Verdict: PASS
