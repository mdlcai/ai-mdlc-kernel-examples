Commit reviewed: 9b9a0cd064f6ed139e05fdcdb4d468b7614fa51c
Previous round reviewed: a59cc9ada09c4cd555644a0466543f5faac2480d

# Reviewer Gate: Review Round 4 (pending REVIEW items C-1, C-2, F-18 and F-19, plus a cold read of F-17)

- **Mechanism:** a fresh-context Task-tool subagent. It did a static review and ran the SDK suite, the invariant lint, ruff and two targeted API pytest files. It ran no e2e, docker, web baseline, Vitest or full API suite, and touched no containers. It did not read FRONTEND-AUDIT.json. It edited no source files and staged nothing.
- **Inputs:**
  - artifacts/reviewer/round-3.md (the previous round);
  - the diff `git diff a59cc9a..HEAD -- . ':!GOVERNANCE-DECISIONS.jsonl' ':!.claude'`. The focus commits are C-1 134ffb2, C-2 bc12571, F-17 fe3b857, F-18 a030137 and F-19 0226b44. The docs and plan commits are C-3 a42d371, C-4 79dbd45, C-5 0fe4dfd and C-6 456ac97. RB-4 (2d2ba1a) is also in the range but is not a focus ID, so it was not re-reviewed here;
  - SPEC.md (§2 kind rows, §3 pipeline params and result, §5 F-17..F-19 and W8, §6 Pipeline and Job detail rows, §7), ARCHITECTURE.md §2.7 and §9, invariants.json, DECISIONS.md ADR-33 and ADR-34, and the BUILD-PLAN.md wave 9 `feature:` blocks (files_write);
  - every source file in the F-17, F-18 and F-19 manifests. Each verdict below comes from the code, and was checked against the docs.
- **Re-run here:**
  - `python -m pytest packages/qai-sdk/tests -q -p no:randomly`: **267 passed, 1 skipped** in 188 s.
  - `npm run lint:invariants`: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web baseline), whose input FRONTEND-AUDIT.json is absent. This is the known environment blocker, not a code defect. INV-34 passes: 0.3.3 agrees across 10 files.
  - `PATH=/c/CODE/.qai-venv/Scripts:$PATH npm run lint:py`: "All checks passed!" and "150 files already formatted".
  - From services/api, `TEST_DB_NAME=qai_test_rev4 TEST_VALKEY_URL=redis://127.0.0.1:56379/7 pytest -q -p no:randomly tests/test_jobs_pipeline.py tests/test_migrations.py`: **8 passed** in 352 s. The host was slow, but nothing flaked.
  - Not run: Vitest, the W8 Playwright e2e and the smoke script. They were not in the allowed runs, and they need the stack.
  - Ad-hoc probe (in-process, no DB): `validate_params('pipeline', json.loads(...))` with an over-large integer target or tolerance (see N-R4-1).

## C-1 (134ffb2): Stage 4 delivery evidence, gates and INV-34

| Item | Verdict | Evidence |
|---|---|---|
| INV-34 version-agreement | PASS | invariants.json:35 adds a `version-agreement` check rooted at package.json, with one pattern for each non-manifest default (compose, .env.example, both pyprojects, `__version__`, config `app_version`, web env.ts). The lint here reports "0.3.3: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree". ADR-33 (DECISIONS.md) records the decision and the fixture exception. ARCHITECTURE §9 gains the INV-34 row in C-3. |
| Design R3 CSS and markup | PASS | jobs.css and validation.css add `grid-template-columns: minmax(0, 1fr)` plus `min-width: 0`. EvidenceList.tsx moves the 16rem minimum to a list-only class (`val-summary-cell--list`). settings.css replaces `1lh` with the label line box. These are layout-only, with no new colours or tokens. The px literals `16px * 1.55` and `15px * 1.55` mirror the label font size (info, not drift: they are not design tokens). |
| REPORT, VERSION, usage-report | PASS | VERSION.md adds the 0.3.2 and 0.3.3 rows. scripts/usage-report.mjs is a new pinned kernel script (header "copy verbatim"). It reads local transcripts only and writes nothing outside REPORT.md. It does not edit any existing pinned runner. Info: REPORT.md:34 still quotes an older lint tally (`33 total / 32 machine (32 proven...)`), while REPORT.md:141 has the current one. |

## C-2 (bc12571): release 0.3.3

| Item | Verdict | Evidence |
|---|---|---|
| Version moved in every location and nothing else | PASS | 10 files, 15 lines, each a `0.1.0` → `0.3.3` change: .env.example, apps/web/lib/env.ts, apps/web/package.json, compose.yaml (4), package-lock.json (3), package.json, both pyprojects, `qai/__init__.py` and config.py. This matches the ADR-33 rule. INV-34 passes. ADR-34 moves the version to 0.4.0 at ship. CHANGELOG.md:3 records "0.4.0 (unreleased)", which is consistent. |

## F-17 (fe3b857): SDK hybrid photonic pipeline (cold read; human-approved, never reviewed)

| Item | Verdict | Evidence |
|---|---|---|
| Manifest | PASS (note) | All code files are in files_write. `qai/validation/evidence.py` (adds the `pipeline` kind, evidence.py:37) was added to the manifest only afterwards, by C-4 79dbd45. The commit also edits SPEC.md:421 (the determinism AC now excludes `ms`) and CHANGELOG.md, neither of which is in files_write. The SPEC edit is a correct clarification, but it should have been a doc commit (N-R4-5). |
| Interferometer composition | PASS | photonics.py:120-125 left-multiplies each element onto the rows of its modes (`u[ms,:] = E @ u[ms,:]`), so U = E_k…E_1 in list order. BeamSplitter matrix photonics.py:59-64 = [[cos θ, −e^{−iφ} sin θ], [e^{iφ} sin θ, cos θ]], as in SPEC. |
| Fock amplitude and permanent | PASS | `amplitude` (photonics.py:177-183) takes `U[np.ix_(outs, ins)]` (rows = output modes repeated t_j times, columns = input modes repeated s_i times) / √(∏s!∏t!). That matches a†_i → Σ_j U[j,i] a†_j. Ryser (131-146): (−1)^n Σ_S (−1)^{\|S\|} ∏_i Σ_{j∈S} a_ij, with the empty permanent = 1 and a 12×12 cap. test_photonics.py:25 checks the brute-force sum to 1e-12 for n = 0..4. HOM (test_photonics.py:39) gives P(1,1) < 1e-12 and P(2,0) = P(0,2) = 0.5. |
| Dual-rail encoding | PASS | photonics.py:214 puts the input photon of qubit q in mode 2q (logical 0). photonics.py:217-220 places the output photon at mode 2q + bit_q, and the key is `format(idx, 0nb)` (highest qubit leftmost, matching the simulator). Post-selection is the total valid-pattern probability (:221). On a pair, BS(θ/2) with φ = 0 is [[c, −s], [s, c]] = Ry(θ). |
| ZYZ | PASS | photonics.py:224-254: α = arg(det g)/2, v = e^{−iα} g ∈ SU(2), a = e^{−i(β+δ)/2} cos(γ/2), b = e^{i(β−δ)/2} sin(γ/2). So γ = 2·atan2(\|b\|,\|a\|), β+δ = −2 arg a and β−δ = 2 arg b. The diagonal and antidiagonal branches are correct. The emission order is Rz(δ) → BS(γ/2) → Rz(β) → global phase, with Rz(t) = PS(m0, −t/2), PS(m1, t/2). Tests: every allow-list single-qubit gate exactly (test_photonics.py:97), the diagonal and antidiagonal cases (:103), and a hypothesis check that the dual-rail amplitudes equal the Statevector amplitudes including the global phase, on 1–4 qubits and 80 examples (:86-94). |
| KLM rejection | PASS | photonics.py:263-267: measure and barrier (`base is None`) are skipped. `len(op.qubits) != 1` (controls + targets) raises `CircuitError(MULTI_QUBIT, "ops[i].gate")`. The test covers cx, swap and ccx with exact path and message (test_photonics.py:110-116). The limits (≤ 8 modes, ≤ 4 photons, ≤ 2000 elements) are at :32-34, :95, :104 and :164, and are tested at :118. |
| Classical AI | PASS | classical_ai.py: a 1→H→1 tanh MLP on 2p−1, trained with seeded PCG64 initialisation and full-batch Adam on the 129-point grid against 2·asin(√p)/π. The backprop gradients (:79-84) are correct. The in-place `params[i] -= …` updates the model arrays. `predict` clips to [0, π]. |
| Pipeline stages and thresholds | PASS | pipeline.py:101 builds one BS(θ_q/2) per pair. :118-124 builds `Circuit(n).ry(θ_q)` plus measure, compiles it and takes the Statevector probabilities. The checks (:136-151) are unitarity ≤ 1e-10, \|1 − post\| ≤ 1e-10, TVD ≤ 1e-10, \|‖ψ‖ − 1\| ≤ 1e-10, and max \|P(q=1) − target\| ≤ tolerance, with the marginal read from `k[n−1−q]` (:59-60). The status is pass iff every check passes. The SDK bounds (hidden ≤ 256, epochs ≤ 20000, tolerance in (0, 0.5]) are wider than the API caps, which is acceptable. Tests: seeds 0–9 pass, and epochs 100 / tol 0.005 fails `target_marginals` while equivalence still passes (test_pipeline.py:43-58). Info: the `postselection` check stores `value = post` (≈ 1), not \|1 − post\| (N-R4-4). |
| Determinism and evidence | PASS | `evidence_parts` (pipeline.py:163-185) hashes the params, status, checks, angles and both distributions, and excludes `ms`. test_pipeline.py:61 asserts an equal result and an equal sha256. |

## F-18 (a030137): pipeline jobs and evidence

| Item | Verdict | Evidence |
|---|---|---|
| Manifest | PASS | Every file is in files_write: schemas/jobs.py, models/jobs.py, routes/releases.py, worker/runners.py, 0009_pipeline.py, both tests, and scripts/smoke-test.mjs (C-6 lists it). The BUILD-STATE and CHANGELOG changes are the usual bookkeeping. models/releases.py is listed but unchanged, because the required_kinds CHECK lives only in migrations. |
| Params validation and field paths | PASS (low finding) | schemas/jobs.py:121-148 `PipelineParams` (extra=forbid): targets 1–4, each a non-bool finite number in [0, 1], reported at `params.targets[i]`; shots StrictInt 1–100000; seed 0–2³²−1; hidden 4–64 (default 16); epochs 100–5000 (default 1000); tolerance 0.001–0.5 (default 0.03), reported at `params.tolerance`. `_FieldError` is mapped to `params.<path>` (:222-224). These match SPEC §3. A circuitId gives 422 `circuitId` UNSUPPORTED (services/jobs.py:203-204). numQubits = len(targets) (:173). test_jobs_pipeline.py:35-70 covers every bound and path. **But** `_number` (:148) calls `math.isfinite` on a raw JSON int, so an integer beyond the float range (for example a 401-digit literal) raises OverflowError. That escapes Pydantic, so the response is 500 INTERNAL instead of 422 (reproduced, N-R4-1). |
| RBAC and tenancy | PASS | A viewer gets 403 FORBIDDEN, and another tenant's project gets 404 NOT_FOUND through `tenant_session` + `get_project` (services/jobs.py:207-210). Tested at test_jobs_pipeline.py:73-80, which passes here. |
| Migration 0009 | PASS | 0009_pipeline.py:36-46 drops and recreates `ck_jobs_kind`, `ck_evidence_records_kind` and `ck_release_candidates_required_kinds` (cardinality 1..7, `<@` the 7 kinds). The names follow the base naming convention, and the recreated downgrade bodies match 0005:30-32 and 0008:30/61 exactly. The downgrade (:53-64) first raises if any pipeline job, evidence or required kind exists. The re-added CHECK would also fail on such rows regardless of RLS. Tests: scratch-DB up → base → up, constraint text contains `pipeline` and `<= 7`, a refused downgrade with a pipeline job, then a clean downgrade and upgrade (test_migrations.py). These pass here, and the shared DB is untouched. |
| Worker runner | PASS | runners.py:293-321 runs in the existing isolated child, maps `CircuitError` to `InputError(message, path)`, builds a single `pipeline` evidence entry with `passed = status == "pass"` (the same shape as the other runners), and adds per-stage `…Ms` metrics. The summary follows SPEC (numQubits, status, maxMarginalError, photonicTvd, checks[{name, status}]). An undertrained run succeeds as a job with fail evidence (test_jobs_pipeline.py:109-119). The evidence verifies (:101-106). |
| Kinds in the API | PASS (low finding) | models/jobs.py JobKind and EvidenceKind, routes/releases.py:26/34 (`Kind` plus max_length 7), and the evidence list filter (routes/evidence.py:25 from the EvidenceKind enum) all include `pipeline`. **But** routes/export_evidence.py:32 `Kind` still lists 6 kinds, so `GET /export/evidence?kind=pipeline` is a 422. SPEC §3 has the export accept the evidence-list filters (N-R4-2). That file is outside F-18's files_write. |
| Smoke | PASS | scripts/smoke-test.mjs adds an end-to-end pipeline check: 202, succeeded, 5/5 pass, 6 modes, one evidence record whose kind is `pipeline` and `verified === true`. It was not run here (it needs the stack). |

## F-19 (0226b44): pipeline screen and job result view

| Item | Verdict | Evidence |
|---|---|---|
| Manifest | PASS | All changes are under apps/web/app/app/pipeline/**, components/results/pipeline-panels(.test).tsx, app/app/jobs/[id]/**, components/shell/nav.ts and e2e/w8-pipeline.spec.ts, plus BUILD-STATE and CHANGELOG. |
| Form limits mirror §3 | PASS | _lib/form.ts:31-34 sets targets 1–4, hidden 4–64/16, epochs 100–5000/1000 and tolerance 0.001–0.5/0.03. Shots and seed reuse the Algorithms `intIn`. `decimalIn` rejects non-decimal text. Errors use the server's `params.*` paths (:100-130). `fieldMessage` and `targetErrors` route server 422s to the matching row, and anything else goes to the banner (`shownOnForm`, :162). The W8 defaults are 0.5/0.25/0.9, 1024 and seed 7. |
| RBAC UI | PASS | PipelineView.tsx:84-92 shows `RoleRequired role="member"` to non-writers and does not fetch projects. page.tsx:17 falls back to viewer when there is no session. The server is still authoritative (F-18 403). |
| Submit and navigation | PASS | PipelineView.tsx:60-69 sends `POST /api/v1/jobs {kind: 'pipeline', projectId, params}`, then `router.push('/app/jobs/' + encodeURIComponent(id))`. Problems come back to the form: QUEUE_FULL and RATE_LIMITED get banner copy in PipelineForm.tsx:29-40. Archived projects are filtered out (:51). |
| Text-only rendering, components/ui | PASS | There is no `dangerouslySetInnerHTML` or innerHTML in either file. Every API value is rendered as React text or a number. `parsePipelineResult` (pipeline-panels.tsx:122-181) narrows every field defensively (bitstring-keyed distributions only, finite numbers, string names). The primitives are Panel, TableWrap, Badge, StatusPill, Field, Input, Select, Button and Histogram. The raw `<table className="table">` inside `TableWrap` follows the algorithm-panels.tsx convention. pipeline.css uses only `var(--space-*)` tokens. The Vitest file includes "renders API strings as text". |
| Stage panels (SPEC F-19, W8.2–W8.5) | PASS | The panels render in order: Classical AI (model, loss, and target / predicted / exact table), Photonics (encoding, modes, photons, element table, post-selection to 6 dp, histogram plus table), Quantum (depth, gate counts, ops, distribution, gate-vs-photonic table) and Validation (checks with StatusPill, overall status, marginals table, and the evidence link `/app/validation/{id}`). These are pipeline-panels.tsx:229-545. |
| Job-detail wiring | PASS | algorithm-slot.tsx renders `PipelinePanels` with `shots` from params and the `pipeline` evidence id. JobDetailView.tsx:124 labels the job "Pipeline" and :242 suppresses the raw JSON result panel for pipeline. Polling and actions are unchanged. |
| Nav | PASS | nav.ts adds "Pipeline" after Algorithms (SPEC F-19). It reuses the `circuit` icon, which is a cosmetic choice. |
| `JobKind` lacks 'pipeline' (known gap) | NOTE (low, part of N-R4-3) | apps/web/app/app/jobs/_lib/types.ts:4 is outside F-19's files_write, so F-19 compares strings (`isPipeline`, algorithm-slot.tsx:31-33). At runtime this is correct: the job page renders and labels pipeline jobs properly, and the other `KIND_LABEL[j.kind] ?? j.kind` sites fall back to the raw `pipeline`. It is not a defect in F-19. It is a type-model gap that hides further UI omissions (N-R4-3), so it should be closed by a manifest that owns jobs/_lib. |
| Tests | NOT RE-RUN | The Vitest form and panel tests and the W8 e2e were read but not run here. The commit reports 126 Vitest tests passing serially and the e2e type-checking only. Both should be confirmed against the stack at the Verification Gate. |

## Findings

None blocking.

- **N-R4-1 (low): services/api/qai_api/schemas/jobs.py:148.** `_number` calls `math.isfinite(x)` on a raw JSON int. An integer above about 1.8e308 in `params.targets[i]` or `params.tolerance` raises OverflowError. Pydantic does not convert that exception, so `POST /jobs` returns 500 INTERNAL instead of a §7 422 (reproduced through `validate_params`). The input is still rejected and nothing is enqueued. **Fix:** in `_number`, reject an int whose `abs(x) > 1e6` before `isfinite`, or wrap the call in `try/except OverflowError: return False`. Add a test with a 400-digit literal.
- **N-R4-2 (low): services/api/qai_api/routes/export_evidence.py:32.** The export's `Kind` Literal lacks `pipeline`, so `GET /api/v1/export/evidence?kind=pipeline` gives 422, while the list endpoint (routes/evidence.py:25, which uses the enum) accepts it. **Fix:** derive the Literal from `EvidenceKind`, or add `"pipeline"`, and add a test. This needs an owning manifest, because F-18's files_write did not include the file.
- **N-R4-3 (low): web kind lists.** apps/web/app/app/jobs/_lib/types.ts:4 and jobs/_lib/format.ts:3 (`JobKind` and `KIND_LABEL` have no `pipeline`), JobsView.tsx:35, validation/_lib/format.ts:3 and governance/_lib/types.ts:6 (`EVIDENCE_KINDS` has 6 kinds) all omit pipeline. As a result:
  - the Jobs list cannot filter by pipeline and shows the lowercase "pipeline";
  - the Validation kind filter has no "Pipeline";
  - the Governance create form cannot require `pipeline` evidence, although SPEC §3 and F-18 (W5) allow it and the API accepts it.

  **Fix:** add `pipeline` to each list with the label "Pipeline", then drop `isPipeline` in favour of `job.kind === 'pipeline'`. Do this in a remediation manifest that lists these files.
- **N-R4-4 (info): packages/qai-sdk/qai/pipeline.py:146.** The `postselection` check records `value = post` (≈ 1) with tolerance 1e-10, while the other residual checks record the error. The Validation table therefore reads "1 ≤ 1.00e-10, passed" (pipeline-panels.tsx:478-479). **Fix:** record `abs(1 - post)` as the value and keep `postselection` in the photonics stage. Note that this changes the evidence hash for new runs.
- **N-R4-5 (info): process.** F-17 edited SPEC.md:421 and CHANGELOG.md outside its files_write. Its evidence.py write was added to the manifest only afterwards, by C-4 (79dbd45). The content is correct, but doc changes should land in a C-commit before the feature. Separately, SPEC §5 F-17 documents `transition_amplitudes(U, occupation)`, while the code takes an `Interferometer` (photonics.py:186). Align the doc.

Counts: 0 blocking, 3 low, 2 info.

Verdict: PASS
