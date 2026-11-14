# Security Audit — Pass 5

Project: QAI // QUANTUM SDK (ce0ff2f1-2252-49ea-8d2e-5a25f2e08102)
Auditor: fresh-context Security auditor, pass 5
Date: 2026-11-01
Tree audited: HEAD `9b9a0cd` against pass 4's tree `c52bf7b` (F-17, F-18, F-19, C-1..C-6; governance records and `.claude` excluded)
Verdict: **PASS-ADVISORY.** The hybrid photonic pipeline adds no medium-or-worse exposure. I found one low and three infos.

- **Resource bounds hold.** The worst job the API accepts (4 targets, hidden 64, epochs 5000, shots 100000) took about **8.4 s wall / 7.5 s CPU** on this heavily loaded host, with a Python heap peak of about 1.2 MB and a result of about 3.6 KB. That is far inside the child's 120 s wall clock, its RLIMIT_CPU and its 2048 MB RLIMIT_AS. It is also in the same range as an existing cost-1 job: a 10-node, p=5, maxiter=500 QAOA took 11.8 s on the same host.
- **Validation is strict.** Bools, strings, null, NaN and inf are rejected, as are extra fields, more than 4 targets and a circuitId. Every case returns 422 `VALIDATION_FAILED` with a `params.*` or `circuitId` field path. There is one gap: an integer too large to convert to a float escapes as a 500 (P5-1).
- **Tenancy is unchanged.** Submit still uses `Member` + `tenant_session` + a project lookup. In the targeted tests, a viewer got 403 and another tenant's project got 404.
- **Evidence is deterministic.** The hashed parts exclude every `ms` timing, so two identical runs give byte-identical canonical inputs and outputs. The evidence verifies on GET, the 0005 append-only trigger covers the new kind, and the 0009 downgrade guard refuses while pipeline rows exist.
- **The web side renders text only.** The new screens use no `dangerouslySetInnerHTML`, `innerHTML` or inline `style`, and they read no `process.env`. The POST goes through `apiSend`, which sends the CSRF header with same-origin credentials.
- **Dependencies.** No new runtime dependency landed. `npm audit --omit=dev` found 0 vulnerabilities. pip-audit is not installed in the venv, so it was not run.

The standing advisory still applies: there is no tooling for secret detection or TypeScript SAST.

## Scope and method

- I read `git diff c52bf7b..HEAD -- . ':!GOVERNANCE-DECISIONS.jsonl' ':!.claude'`, in particular:
  - `qai/classical_ai.py`, `qai/photonics.py`, `qai/pipeline.py`, `schemas/jobs.py` (PipelineParams), `worker/runners.py` (`pipeline`), `models/jobs.py`, `routes/releases.py` and `migrations/versions/0009_pipeline.py`;
  - `apps/web/app/app/pipeline/**`, `components/results/pipeline-panels.tsx` and the `jobs/[id]` wiring;
  - the C-1/C-2 version bumps in `package.json`, `package-lock.json`, the `pyproject.toml` files, `compose.yaml`, `.env.example`, `config.py`, `env.ts` and `invariants.json`.
- I also read, unchanged, `services/jobs.py` (submit), `services/ratelimit.py` (`job_cost`), `repositories/jobs.py` (`claim`), `worker/loop.py` and `worker/child.py` (limits), `routes/evidence.py`, `services/evidence.py` and `errors.py`.
- My scripts are in [local scratchpad path redacted]:
  - `p5_cost.py`: `run_pipeline` at the API max, the API min and the SDK max; evidence-part determinism; a 12×12 permanent; `fock_basis(8,4)`; and an 8-mode, 4-photon, 2000-element `output_distribution`.
  - `p5_validate.py`: 23 adversarial `params` against `validate_params("pipeline")`, the envelope's extra fields, and the worker runner with a tampered `hidden`.
  - `p5_http.py`: the real `JobIn` and `errors.install` handlers in a minimal FastAPI app, to confirm the status codes over HTTP.
  - `p5_compare.py`: the max cost-1 QAOA job for comparison.
- I made no source edits, no commits and no docker commands.
- **Timing caveat:** the host was heavily loaded, so all timings are upper-bound indications, not benchmarks.

| Category | Tool | Result |
|---|---|---|
| Pipeline worst case | `p5_cost.py` | API max: **8413 ms wall / 7594 ms CPU** (repeat: 8290 / 7453). The classical stage takes about 8.3 s of that; photonics 37 ms, quantum 4 ms, validation 5 ms. The Python heap peak is 1.22 MB and the result JSON is 3598 B. API min: 133 ms. SDK-only max (hidden 256, epochs 20000), which the API cannot reach: 56.5 s. |
| Photonics bounds | `p5_cost.py` | The pipeline only calls `dual_rail_state`: at most 16 amplitudes, each a permanent of at most 4×4. The SDK-only bounds are a 12×12 Ryser permanent in 108 ms, `fock_basis(8,4)` with 330 states in 0.7 ms, and 2000 elements, 8 modes and 4 photons for the full distribution in 176 ms. |
| Comparison | `p5_compare.py` | A 10-node complete-graph QAOA with p=5 and maxiter=500 (cost 1) took **11.81 s**. A 10-qubit Grover with 1000 iterations and 100k shots took 0.43 s. |
| Validation | `p5_validate.py`, `p5_http.py` | 21 of 23 cases give a 422 with the correct path. A huge integer in `targets[i]` or in `tolerance` gives **500 INTERNAL** (P5-1). With a tampered payload, the runner's SDK re-check raises `InputError hidden`, so it is defended in depth. |
| Determinism | `p5_cost.py` | `evidence_parts` from two identical runs are equal, and no `"ms"` appears in them. |
| API tests | `pytest -q -p no:randomly tests/test_jobs_pipeline.py` (qai_test_sec5, Valkey db 8) | **5 passed** in 88.7 s: the params contract, viewer 403 / other-tenant 404, W8 end to end with verified evidence and the kind=pipeline listing, failing evidence, and release `requiredKinds` with 7 kinds and 8 rejected. |
| SAST (Python) | `ruff check --select S,ASYNC,RUF006` on the 7 changed Python files | All checks passed. |
| Dependency vulns (npm) | `npm audit --omit=dev` (apps/web) | **found 0 vulnerabilities.** The lockfile diff is the version field only. |
| Dependency vulns (Python) | pip-audit | **Not run: not installed in `/c/CODE/.qai-venv`.** No Python dependency changed: the `pyproject.toml` diffs are `version =` only, and the new modules import only stdlib and numpy. |
| SAST (TS) / secrets | none | **ADVISORY ONLY — no external tooling available for SAST (TypeScript/web) or secret detection. Agent self-review is not a substitute for automated scanning.** |

## Findings (new in pass 5)

| id | severity | file:line | description | exploit scenario | fix | confidence |
|---|---|---|---|---|---|---|
| P5-1 | low | services/api/qai_api/schemas/jobs.py:147-148 (called at 138, 140); the same pattern is in packages/qai-sdk/qai/pipeline.py:45 and qai/photonics.py:41 | **An integer too large for a float escapes validation as a 500.** `_number` calls `math.isfinite(x)` on a Python `int`, which converts it to a float. For any integer above about 1.8e308 (309+ digits; `json.loads` accepts up to 4300 digits), that raises `OverflowError`. Pydantic only converts `ValueError` and `AssertionError` raised in a validator, so the exception leaves `validate_params` and the catch-all handler returns `500 INTERNAL` with an `unhandled_error` log line and traceback. SPEC §7 and §3 require a 422 `params.targets[i]` / `params.tolerance`. Confirmed over HTTP with the real handlers (`p5_http.py`). The transaction rolls back and nothing is enqueued or charged. | A member, or a member-role API key, posts `"targets":[1000…0]` (310 digits) or `"tolerance":1e309-as-an-int`. They get a 500 instead of a 422, and each request writes an error-level log with a traceback, which adds noise to alerting. Each request costs one ordinary validation, and the general per-principal and per-IP windows bound the rate. It has no data or tenancy impact. | In `_number`, reject integers outside the range before calling isfinite, e.g. `if isinstance(x, int): return abs(x) <= 1e6`, or wrap it in `try: ... except OverflowError: return False`. Apply the same change in SDK `_targets` and `photonics._finite`, so direct SDK callers get a `CircuitError`. Add the 310-digit case to `test_jobs_pipeline.py`. | high (measured) |
| P5-2 | info | services/api/qai_api/services/ratelimit.py:131-134; repositories/jobs.py:101-128; schemas/jobs.py:173 | **The worst-case pipeline job costs 1 unit for about 8 s of worker CPU.** `qubits_for` is `len(targets)` ≤ 4, so `job_cost` = 1. The cost tracks qubits only, not epochs × hidden. One tenant can keep `MAX_QUEUED_PER_TENANT` = 50 max-size pipeline jobs queued, about 7 worker-minutes in total. `claim` is FIFO across tenants with no per-tenant fairness, so other tenants see head-of-line delay. This is not new: a cost-1 QAOA (11.8 s) already allows the same thing, and the 120 s timeout, rlimits and queue caps bound each job. SPEC §9's "< 3 s" target covers the defaults only (hidden 16, epochs 1000). | A member loops submits that keep 50 max pipeline jobs queued. With the default `WORKER_CONCURRENCY` = 2, other tenants' jobs wait several minutes. The effect is degradation, not an outage, and it was already possible with QAOA. | Accept for v1, together with the existing queue model. Optional: weight the pipeline cost by `ceil(epochs × hidden / 64000)`, or add per-tenant round-robin to `claim` (e.g. at most N running per tenant). | high (measured costs; queueing computed) |
| P5-3 | info | services/api/qai_api/schemas/jobs.py:138-141 | **Type errors are reported as `OUT_OF_RANGE`.** A string, null, nested list or bool in `targets[i]` or `tolerance` returns `OUT_OF_RANGE`, where the other params use `INVALID` for a wrong type. Both codes are in SPEC §7 and the paths are correct, so this is cosmetic. | None. | Optional: report `INVALID` when `_number` is false, and `OUT_OF_RANGE` only for the range check. | high |
| P5-4 | info | packages/qai-sdk/qai/classical_ai.py:22-23; photonics.py:32-35 | **The SDK limits are looser than the API's.** The SDK allows `MAX_HIDDEN` 256 and `MAX_EPOCHS` 20000, which took 56.5 s here, against the API's 64 and 5000. The SDK-only photonics paths allow a permanent up to 12×12 and 2000 elements. None of these is reachable from the API: PipelineParams caps the inputs, and the runner passes only validated params. Even the SDK maximum is still under the 120 s child timeout, but with little margin on a loaded host. This only matters if a future runner forwards SDK-level limits. | None today. | Keep the API caps as the source of truth. If a future feature exposes `photonics.from_circuit` or `output_distribution` through a job, budget it with the measured costs above. | high |

Counts (new in pass 5): critical 0, high 0, medium 0, low 1, info 3.

## Threat checklist

| threat | result | evidence |
|---|---|---|
| 1. Resource exhaustion | **bounded** (P5-2 info) | targets ≤ 4 → ≤ 8 modes and ≤ 4 photons. `dual_rail_state` computes 2^n ≤ 16 permanents of n×n ≤ 4×4 (Ryser 2^4 terms), and `fock_basis` is not on the pipeline path. There are n BeamSplitters (≤ 4), far below MAX_ELEMENTS 2000. Training is 129 points × hidden 64 × 5000 epochs, which costs about 8.4 s. Shots ≤ 100000 go through one multinomial per model. Memory is O(hidden × 129), about 1 MB. The child has `JOB_TIMEOUT_SEC` 120 (`wait_for` + kill), `RLIMIT_CPU` timeout+1, `RLIMIT_AS` 2048 MB and no core dump. Per tenant: `jobs:tenant` 200 units/min, 50 queued, a global depth of 5000, and FIFO claim. |
| 2. Input validation | **holds except P5-1** | `extra="forbid"` on PipelineParams and JobIn. `StrictInt` for shots, seed, hidden and epochs, so bool and float are rejected. `_number` rejects bool, str, None, list, NaN and ±inf. targets has 1–4 entries, and 10000 entries are rejected at 0.1 ms. The circuitId check happens before any DB work and returns 422 `circuitId` `UNSUPPORTED` (services/jobs.py:203-204, tested). No new error codes: every code is in SPEC §7 (`VALIDATION_FAILED`, `OUT_OF_RANGE`, `TOO_SHORT`, `TOO_LONG`, `UNKNOWN_FIELD`, `REQUIRED`, `INVALID`, `UNSUPPORTED`). |
| 3. Tenancy and RBAC | **holds** | `submit_job` uses `Member` and runs everything under `tenant_session(principal.tenant_id, …)`, with the project lookup filtered on the tenant plus RLS, so a foreign project gives 404 (tested). A viewer gets 403 (tested). `GET /evidence?kind=pipeline`: `_KINDS` is derived from `EvidenceKind`, and the filter is added inside the tenant-scoped query, with an unknown kind giving 422 `kind`. `requiredKinds`: a `Literal` with 7 values and `max_length=7`, backed by the DB CHECK `<@ ARRAY[...]` with cardinality 1–7. Releases still use the session-only member and admin dependencies, unchanged. |
| 4. Evidence integrity | **holds** | `evidence_parts` hashes `inputs` (the 6 validated params) and `outputs` (status, checks, angles and both probability maps). The `ms` fields appear only in `run_results` and `metrics`. The record goes through the existing `services/evidence.write` (canonical SHA-256, ADR-16), and verify recomputes from the stored inputs, outputs and env, which returned `verified: true` in the test. The 0005 immutability trigger and the REVOKE apply table-wide, so the new kind is append-only too. The 0009 downgrade guard does a `DO $$ … RAISE` when any pipeline job, evidence row or required kind exists. The migration role is the table owner and there is no FORCE RLS, so the guard sees every tenant's rows. Even if it did not, re-adding the narrower CHECK would fail closed on the remaining rows. `test_migrations.py` covers both directions. The 0009 SQL interpolates only module constants, so there is no injection surface. |
| 5. Web | **holds** | The pipeline screen and panels have no `dangerouslySetInnerHTML`, `innerHTML`, `eval` or inline `style` (CSP `style-src 'self' 'nonce-…'` unchanged; `proxy.ts` not modified). All result data is rendered as React text. Links use `encodeURIComponent(id)`. `apiSend('POST','/api/v1/jobs')` sends `X-CSRF-Token` from `__Host-qai_csrf` with `credentials: 'same-origin'`, and the server's Origin and CSRF checks are unchanged. No `process.env` and no secrets in the client files. A viewer sees the `RoleRequired` notice, and the server enforces the role regardless. |
| 6. Dependencies | **no new runtime dependency** | npm: 0 vulnerabilities, and the lockfile change is the version only. Python: no manifest change and pip-audit unavailable. ruff S rules are clean. |

## Status of accepted residuals

P4-1, P4-2, P4-3 and P4-4: no change. This range touches no compile, CPU-slot, redaction or key-cap code (`services/cpu.py`, `routes/circuits.py`, `logging.py` and `routes/api_keys.py` are absent from the diff).

F-4, F-5, F-6, F-9, F-10, F-11, P2-3, P2-5 and P2-6: no change. F-5, the job child sharing the worker UID and network, now also covers the `pipeline` runner, which is pure NumPy with no I/O.

Verdict: PASS-ADVISORY
