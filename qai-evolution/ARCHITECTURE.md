# ARCHITECTURE — Ψ ◈ QAI // QUANTUM SDK

Version 0.1.0 · 2026-10-31 · build_depth: comprehensive · scale tier: **large** · review_gates: auto

## 1. Summary

**Problem.** Quantum software work is fragmented. Circuits are written in one tool, simulated in another, validated by hand (if at all), and released with no governance trail. Results are rarely reproducible across machines and seeds.

**Solution.** Two deliverables share one numerical core:

1. **`qai` Python SDK** (`packages/qai-sdk`). It covers:
   - the circuit API, an allow-listed gate set and a minimal compile pass;
   - a NumPy complex128 statevector simulator (≤ 20 qubits) with seeded sampling;
   - Grover and QAOA-MaxCut, with classical baselines;
   - Qiskit differential validation, benchmarks and an evidence record format;
   - the governed evolution engine, which accepts a candidate iff `V(S)=1 ∧ J(S) ≤ J(Ψn) − ε`.

   The SDK is pure Python and pip-installable on native Windows.
2. **QAI Platform.** A multi-tenant web platform (Next.js 16 + FastAPI + PostgreSQL 18) that runs SDK workloads as queued jobs. It records every result as evidence and computes release-readiness from that evidence. It enforces RBAC, rate limits and an append-only audit log. It serves over HTTPS only (Caddy `tls internal` locally).

**Users.** Quantum researchers, developers and AI engineers (W1–W4, W6). Tenant admins and compliance reviewers (W5, W7). Educators (read-only viewers).

**Non-goals (V1).** Real QPU execution, a tensor-network backend, distributed simulation beyond 20 qubits, OpenQASM import (export only), email delivery, SSO/OIDC, billing.

## 2. System Architecture

### 2.1 Layers

```
Browser --HTTPS--> Caddy edge (TLS, :80->:443, rate limit, body caps, static cache)
                     |--> web x2  (Next.js 16 App Router, proxy.ts CSP nonce, React Context state)
                     +--> api x2  (FastAPI /api/v1, stateless)
                              |--> PostgreSQL primary   (writes, jobs queue, audit, evidence)
                              |--> PostgreSQL replica   (read-only list/report queries)
                              +--> Valkey               (cache-aside, rate-limit buckets)
worker x2 --claim SKIP LOCKED--> PostgreSQL jobs --> qai SDK (simulate / algorithms / benchmark / evolution)
scheduler x1 --> scheduled benchmarks, stuck-job watchdog, retention sweep
```

### 2.2 Modules

| Module | Path | Responsibility |
|---|---|---|
| SDK core | `packages/qai-sdk/qai/{circuit,gates,qasm,compile}.py` | Circuit IR, gate matrices, OpenQASM 2 export, compile pass |
| SDK sim | `packages/qai-sdk/qai/simulator.py` | Statevector apply (tensordot over qubit axes), probabilities, seeded sampling (`numpy.random.Generator(PCG64(seed))`) |
| SDK algorithms | `packages/qai-sdk/qai/algorithms/` | Grover (optimal iteration count), QAOA p-layer MaxCut + SciPy COBYLA, exact brute-force and greedy baselines |
| SDK validation | `packages/qai-sdk/qai/validation/` | Mathematical checks (norm, unitarity), Qiskit differential (optional `[reference]` extra), benchmark harness, evidence record (sha256 of canonical inputs + outputs + env) |
| SDK hybrid pipeline | `packages/qai-sdk/qai/{classical_ai,photonics,pipeline}.py` | NumPy MLP angle surrogate (seeded Adam), ideal linear-optics Fock simulator (beam splitters, phase shifters, Ryser permanent, dual-rail encoding, ≤ 8 modes / ≤ 4 photons), and the four-stage W8 pipeline with its validation checks (ADR-34) |
| SDK AI simulation | `packages/qai-sdk/qai/surrogate.py` | Angle slots of a saved circuit, seeded Latin-hypercube design, exact statevector training data, physics-informed Fourier ridge surrogate (3^d basis, d ≤ 6), chunked candidate exploration, exact re-validation and the W9 checks (ADR-41) |
| SDK Orb Lab | `packages/qai-sdk/qai/orb.py` | ORB-0 spherical-cavity experiment: closed-form TE/TM modes (Bessel roots, quadrature Q0 and equipartition), field map, coupled-mode energy ledger (DOP853) with a closed-form cross-check, Dürr–Høyer Grover maximum finding within a query budget, Rényi alignment, and pass / fail / inconclusive checks (ADR-46) |
| SDK formula language | `packages/qai-sdk/qai/expr.py` | Closed math-notation grammar for user formulas: recursive-descent parser to a tuple tree, NumPy evaluation, canonical form, limits (2,000 chars, 512 terms, depth 32 checked before recursing). No eval/exec/compile (INV-10, ADR-48) |
| SDK Formula Lab | `packages/qai-sdk/qai/formula.py` | Four modes on a user formula: objective (classical exhaustive vs Dürr–Høyer), physics (formula vs the orb ledger across Q_ext), map (Ψₙ₊₁ = f(Ψₙ, Eₙ, n) with bounds, convergence, fixed point, stability), state (amplitudes or angle formulas on the simulator, sampling TVD bound); checks, outcome and evidence parts |
| SDK evidence protocol | `packages/qai-sdk/qai/protocol.py` | The six-test protocol for the evolution engine (correctness against enumeration, improvement on fresh seeds, baselines at an equal budget, 30-run repeatability, train/validation/test generalization with a lock, governance injections through the real gate). It is seeded by SeedSequence spawn and deterministic. The gate with evidence completeness is in `qai/evolution.py` (`gate`, `evaluate_set`). |
| SDK evolution | `packages/qai-sdk/qai/evolution.py` | Candidate generation (parameter/config sets, never code), V/J evaluation over a fixed seed set, acceptance rule, checkpoint history |
| API app | `services/api/qai_api/main.py` | App factory, lifespan, middleware (request id, JSON logs, security headers, error contract) |
| API config | `services/api/qai_api/config.py` | pydantic-settings, fail-fast |
| API db | `services/api/qai_api/db/` | Async engines (primary + replica), session router, RLS `SET LOCAL app.tenant_id` |
| API models / migrations | `services/api/qai_api/models/`, `services/api/migrations/` | SQLAlchemy 2.1 models, Alembic revisions with downgrade |
| API repositories | `services/api/qai_api/repositories/` | The only module that issues SQL; every query takes `tenant_id` |
| API services | `services/api/qai_api/services/` | Business logic: auth, jobs, governance, evidence, audit, export |
| API routes | `services/api/qai_api/routes/` | Thin HTTP layer, RBAC dependencies, Pydantic schemas |
| Worker | `services/api/qai_api/worker/` | Claims and executes jobs, writes results + evidence, retries, DLQ |
| Scheduler | `services/api/qai_api/scheduler/` (`triggers.py`) | Triggers T1–T3 with an injected clock and per-org entry points |
| Web | `apps/web/` | App shell, screens, design-system primitives, API client |
| Edge | `infra/caddy/Caddyfile` | TLS, redirect, routing, limits |

### 2.3 Data flow (simulation run, W2)

1. **Submit.** The user submits a circuit from `/app/simulator`. The web app calls `POST /api/v1/jobs` (`{kind: "simulate", projectId, circuitId, params}`) with the session cookie and the CSRF header.
2. **Validate and enqueue.** The API checks the payload limits: ≤ 20 qubits, ≤ 2,000 gates, ≤ 100,000 shots, finite params. It then checks RBAC (`member+`) and the per-tenant concurrent-job quota.
   - If the queue is over depth, it answers 429 with `Retry-After`.
   - Otherwise it inserts `jobs(status=queued)` and an `audit_events` row in one transaction and returns `202 {job}`.
3. **Run.** A worker claims the job with `UPDATE jobs SET status='running', claimed_at=now(), lease_until=… WHERE id = (SELECT id … FOR UPDATE SKIP LOCKED LIMIT 1)`. It runs the SDK in a child process under a wall-time limit, writes `run_results` and `evidence_records`, and sets `status='succeeded'`.
4. **Show.** The web app polls `GET /api/v1/jobs/{id}` (replica read, falling back to the primary on a miss) and renders the statevector, histogram and evidence hash.

### 2.4 Job state machine

- `queued → running → succeeded | failed`
- `failed → queued`: retry while attempts < 3.
- `failed → dead`: attempts ≥ 3 (dead-letter queue).
- `queued → cancelled`.
- `running → queued`: the watchdog requeues a job whose lease expired.

Every status is written by `services/api/qai_api/services/jobs.py` or the worker.

### 2.5 Release-readiness state machine (W5)

- `pending → ready`: all required evidence is present and passing.
- `pending → blocked`: any evidence is failing or missing.
- `ready → approved`: approval needs the reviewer capability, and the reviewer cannot be the author.
- `approved → released`: releasing also promotes the release to the project's active Ψ (§2.12).
- Any new failing evidence moves `ready` or `approved` to `blocked`.
- `pending | ready | blocked | approved → held` and `… | held → rejected`: a reviewer decision with a required reason, never by the author (Re-Entry R8, ADR-54). `held → ready | blocked` by evaluating. `rejected` and `released` are terminal.

Status is computed server-side in `services/governance.py`. Clients never supply it.

### 2.6 Evolution decision (W6)

A candidate is a parameter/config set: QAOA layers, initial angles, optimizer budget, simulator strategy. The platform evaluates two quantities:
- `V(S)`: the validation checks pass on every seed.
- `J(S)`: the mean objective cost over the fixed seed protocol (5–30 seeds per run, SPEC §3). Held-out seeds are stored server-side.

The candidate is `accepted` iff `V(S)=1 ∧ J(S) ≤ J(Ψn) − ε`; otherwise it is `rejected`. Decisions are append-only in `evolution_decisions`, and an accepted candidate becomes checkpoint Ψn+1.

### 2.7 Hybrid photonic pipeline (W8, Re-Entry R1)

`POST /jobs {kind: pipeline}` validates the params, enqueues the job and returns 202. A worker child then runs `qai.pipeline.run_pipeline` through four stages:
1. **Classical AI.** `AngleSurrogate.fit` predicts θ[].
2. **Photonics.** A dual-rail `Interferometer` (one BS(θ/2) per qubit) feeds `transition_amplitudes` (permanents). Post-selection gives the photonic distribution.
3. **Quantum operations.** `Circuit(n).ry(θ_q)` is compiled, and `Statevector` gives the gate-model distribution.
4. **Validation.** Five checks run, and `evidence.record("pipeline", …)` writes the record.

The child returns JSON only. The parent writes `run_results` plus one `evidence_records` row (kind `pipeline`) in one tenant transaction, as for every other kind. There is no new table, route module or trust boundary.

Linear optics is limited to single-qubit gates. The photonic model never approximates an entangling gate; it rejects it (KLM, RESEARCH R1).

### 2.8 AI Simulation (W9, Re-Entry R3)

`POST /jobs {kind: surrogate, circuitId}` loads the circuit inside `tenant_session`, so another tenant's id is 404. It validates the params against that circuit with `qai.surrogate.parameter_slots`, enqueues the job and returns 202. A worker child then runs `qai.surrogate.simulate_surrogate`:
1. **Design.** A seeded Latin hypercube over the chosen angles.
2. **Exact.** `Statevector` probabilities per sample. This is the training data.
3. **Training.** `FourierSurrogate.fit` (ridge on the {1, cos, sin}^⊗d basis).
4. **Prediction and exploration.** Held-out TVDs, then the candidates are scored on the target column in 4,096-row chunks and the top-k are re-run exactly.
5. **Validation.** Four checks run, and `evidence.record("surrogate", …)` writes the record.

The child returns JSON only. The parent writes `run_results` plus one `evidence_records` row (kind `surrogate`) in one tenant transaction, as for every other kind. There is no new table, route module, external service or trust boundary: the model is fitted per job, in memory, from that job's own simulations, and nothing is persisted except the result. Cost is bounded by d ≤ 6, n ≤ 10, samples ≤ 4,096 and candidates ≤ 65,536, under the existing job timeout and RLIMIT_CPU.

### 2.9 Orb Lab (W10, Re-Entry R4)

`POST /jobs {kind: orb}` validates the params (no circuit), enqueues the job and returns 202. A worker child runs `qai.orb.run_orb`:
1. **Model.** The cavity's lowest modes from the zeros of j_l and [x·j_l]′, with Q0 and equipartition by quadrature, and the driven mode's field map.
2. **Simulate.** The coupled-mode energy ledger for the baseline and the chosen coupling.
3. **Optimize.** The extracted-energy objective over 2^qubits extraction couplings: classical exhaustive and Brent searches, and Dürr–Høyer on the state-vector Grover model within a query budget (it reports the budget spent and `foundAt`, ADR-47).
4. **Validate.** Eight physics checks (any failure makes the run `fail`) and two comparison checks (a failure makes it `inconclusive`).
5. **Compare and report.** The Rényi alignment against the budget, the success curve, and the outcome with its reason.

The child returns JSON only. The parent writes `run_results` plus one `evidence_records` row (kind `orb`; evidence status pass only for a pass outcome) in one tenant transaction. There is no new table, route module, external service or trust boundary. A run is 2–3 s single-threaded at the defaults (up to ~15 s and ~260 KB at the limits), bounded by modes ≤ 20, qubits ≤ 8 and budget ≤ 2,048 under the existing job timeout and RLIMIT_CPU. The model is simplified: an ideal sphere, linear walls, no plasma and no FDTD.

### 2.10 Formula Lab (W11, Re-Entry R5)

`POST /jobs {kind: formula}` validates the params per mode and parses the formula with `qai.expr` at submit, so a malformed formula returns 422 with its line and column before anything is queued. Parsing is linear in the input and capped at 2,000 characters and 512 terms, so it runs inline rather than through `services/cpu.run_bounded`. State mode with `source: circuit` loads the circuit in the tenant session. The job is enqueued and the API returns 202. A worker child runs `qai.formula.run_formula`:
1. **Parse.** The same grammar again, in the child, so the worker never trusts the stored params.
2. **Run the mode.** objective: grid evaluation, exhaustive search and Dürr–Høyer within the budget. physics: the orb cavity mode and the numerical ledger at each Q_ext. map: the iteration with early stops. state: the normalised amplitudes or the angle sweep on the state-vector simulator, then seeded sampling.
3. **Validate.** Physics checks (any failure makes the run `fail`) and comparison checks (a failure makes it `inconclusive`), per mode as SPEC F-28 lists them.
4. **Report.** The outcome with its reason.

The child returns JSON only. The parent writes `run_results` plus one `evidence_records` row (kind `formula`; evidence status pass only for a pass outcome) in one tenant transaction. There is no new table, route module, external service or trust boundary. The formula is untrusted text: it reaches only a parser that builds tuples and an evaluator that dispatches on a fixed table of NumPy functions. There is no name lookup outside the mode's variables, constants and definitions, no attribute access and no indexing. Evaluation runs in the job child under the existing timeout and RLIMIT_CPU, and its cost is bounded by terms × points (≤ 512 × 1,024 for objective, × 1,000 steps for map). The web app mirrors the parser in TypeScript for instant feedback only; the server's parse decides.

### 2.11 Evidence protocol (W12, Re-Entry R6)

`POST /jobs {kind: protocol}` validates the params (no circuit), refuses a second queued or running protocol job in the tenant with 429 QUEUE_FULL, enqueues the job and returns 202. A worker child runs `qai.protocol.run_protocol`. For each of the `runs` replicates, every one with its own SeedSequence child, it runs these steps:
1. **Draw.** Disjoint train, validation and test MaxCut instances from G(n, p).
2. **Evolve.** Ψ₀ → candidates → `gate` on train × decision seeds. This is the same acceptance rule as the evolution service (§2.6).
3. **Select.** The best checkpoint on validation × fresh seeds. It is adopted only if the paired bootstrap CI against Ψ₀ excludes 0.
4. **Lock.** The sha256 of the selected config, recorded before any test evaluation.
5. **Baselines.** Random search and COBYLA on the same B configuration evaluations.
6. **Test once.** Ψ₀, Ψ* and the baselines on test × fresh seeds. A second test evaluation raises.

It then computes the six tests across the runs (bootstrap CIs, distributions), re-runs replicate 1 to check determinism, and injects the worse, invalid and missing candidates into the real gate against a recorded checkpoint.

The child returns JSON only. The parent writes `run_results` plus one `evidence_records` row (kind `protocol`; evidence status pass only for a pass outcome) in one tenant transaction.
- Protocol decisions live in the result and its evidence. They are not `evolution_decisions` rows, which belong to evolution runs (INV-31..32 unchanged).
- The child runs under `PROTOCOL_TIMEOUT_SEC` (default 900) in place of `JOB_TIMEOUT_SEC`. Leases renew every LEASE_SEC / 3, so a long job is never requeued while its worker lives.
- At the defaults a protocol takes about 3 minutes of one worker. The one-per-tenant cap keeps a tenant from occupying both worker replicas.
- There is no new table, route module, external service or trust boundary.

### 2.12 Validation Lab phase A: active Ψ and the audit chain (W13, Re-Entry R8)

The Validation Lab loop `Ψₙ₊₁ = G(V(Q(L(Ψₙ, Eₙ))))` maps onto existing parts. L and Q are the evolution run (§2.6), V is the worker's recomputation plus the evidence re-verification (§2.5), and G is the governance service with a human reviewer. Phase A adds the versioned state and its history:
- **Ψ snapshot.** A release candidate may carry the best checkpoint of a succeeded evolution run of its project, with the sha256 of its canonical JSON. The snapshot is fixed when the release is created and recomputed before promotion.
- **Engine switches.** `engine_switches` is an append-only list per project: `promoted` (written in the release transaction) and `rolled_back` (admin with the reviewer capability, reason required). The active Ψ is the target of the highest seq. The project row lock plus UNIQUE(tenant_id, project_id, seq) serialise switches, so promotion is exactly once and a lost race changes nothing. Only `services/governance.py` writes switches.
- **Audit chain.** The `audit_events_chain` BEFORE INSERT trigger (SECURITY DEFINER, owned by the migration role, fixed search_path) takes `pg_advisory_xact_lock` per tenant and links each event to the previous one by sha256. The lock serialises audit inserts within a tenant until commit; tenants do not contend. `GET /audit/verify` recomputes the chain with the same SQL function, in a repository query.
- **Lock order.** The audit lock is the last lock a transaction takes: every row lock (release, project) is taken before the first `audit.record` of the transaction, so no two transactions wait on each other in a cycle. Releasing locks the project row before its first audit event, and completing a job locks the project's open releases (by id) before it writes evidence.
- **Limits.** The chain detects an edited, deleted or reordered row, except removal of the newest events unless the reported head is recorded outside the database. It does not stop the database owner from rebuilding the whole chain, and nothing is anchored outside the database. Evidence is hashed, not signed.
- There is no new external service or trust boundary. One table (`engine_switches`), two migrations (0014, 0015) and the routes of SPEC §3 are added.

### 2.13 Validation Lab phase B: baselines, ablations, report bundle (W12, Re-Entry R9)

- **Protocol arms.** `qai.protocol` adds three QAOA ablation arms (noLearner, noValidation, ungated) to the run's single test evaluation, and three classical arms (random bitstring, local search, annealing) on the same frozen test pairs, each with exactly `classicalEvaluations` cut evaluations. The ungated arm is computed inside the protocol only: it never calls `gate`, the release flow or production approval (ADR-49, ADR-55).
- **Statistics.** Each comparison carries a CI, a paired sign-flip permutation p and a Holm-adjusted p over the family, plus d_z. New draws use their own SeedSequence streams, so the phase-A numbers do not move.
- **Scorer.** `qai.algorithms.maxcut.score_cut` rescores the best sampled bitstrings independently of the QAOA cut vector.
- **Report bundle.** `GET /api/v1/evidence/{id}/report` is a pure read in a tenant session: the route loads the evidence row and its job result through repositories, `services/report` renders deterministic `report.md` and the JSON files, and the route writes one `export.evidence_report` audit event in the same transaction. Nothing is stored. The manifest is hashed, not signed.
- **Commit.** The worker passes `GIT_COMMIT` to the child, and `current_env()` records it as `commit`.
- There is no new table, migration, external service or trust boundary.

## 3. Agent and Tool Orchestration

The product contains no LLM agent. "AI proposes" means the evolution engine's candidate generator, a deterministic seeded perturbation search. The governance gate G is code. The build itself is orchestrated by the MDLC kernel: features are dispatched per BUILD-PLAN.md waves, and FOUNDATIONAL/CRITICAL features run in the orchestrator.

## 4. Data Layer (PostgreSQL 18.6)

Every tenant-owned table carries `tenant_id UUID NOT NULL`, indexed, with RLS enabled under the policy `tenant_id = current_setting('app.tenant_id', true)::uuid`. The app sets `SET LOCAL app.tenant_id` per transaction, and repositories also filter explicitly (defence in depth). IDs are UUIDs generated in the application.

| Table | Key columns | Notes |
|---|---|---|
| tenants | id, name, slug UNIQUE, created_at | |
| users | id, email UNIQUE (lower-cased), password_hash (argon2id), created_at, disabled_at | global identity |
| memberships | tenant_id, user_id, role (owner/admin/member/viewer), is_reviewer, UNIQUE(tenant_id,user_id) | RBAC |
| sessions | id, token_hash UNIQUE, user_id, tenant_id, expires_at, idle_expires_at | server-side sessions, cookie `__Host-qai_session` |
| invites | tenant_id, token_hash UNIQUE, role, expires_at, accepted_at | admin-issued |
| api_keys | tenant_id, prefix UNIQUE, key_hash, role, created_by, revoked_at, last_used_at | |
| projects | tenant_id, id, name, description, created_by, archived_at | |
| circuits | tenant_id, id, project_id, name, num_qubits, ops JSONB, version, updated_at | ops validated against the SDK schema |
| jobs | tenant_id, id, project_id, kind (simulate/grover/qaoa/benchmark/evolution), status, payload JSONB, attempts, claimed_at, lease_until, error, created_by, created_at, finished_at | queue + state machine; index (status, created_at) |
| run_results | tenant_id, job_id UNIQUE, result JSONB, metrics JSONB | |
| evidence_records | tenant_id, id, job_id, project_id, kind, status (pass/fail), sha256, env JSONB, created_at | never deleted by retention |
| release_candidates | tenant_id, id, project_id, version, status, author_id, approved_by, requirements JSONB, psi JSONB + psi_sha256, decision_reason, decided_by | Ψ snapshot fixed at creation (§2.12) |
| engine_switches | tenant_id, id, project_id, seq, action (promoted/rolled_back), from_release_id, to_release_id, reason, actor_id, at, UNIQUE(tenant_id, project_id, seq) | append-only for the app role; the active Ψ is the highest seq |
| evolution_runs / evolution_decisions | tenant_id, run_id, iteration, candidate JSONB, v_pass, j_value, j_baseline, epsilon, decision | append-only |
| schedules | tenant_id, id, project_id, kind (benchmark), interval_minutes, timezone (UTC), enabled, next_run_at | |
| scheduled_runs | schedule_id, window_start, job_id, created_at, **UNIQUE(schedule_id, window_start)** | idempotent claim |
| scheduler_runs | id, trigger, started_at, finished_at, processed, skipped, failed | observable outcome |
| worker_heartbeats | worker_id, seen_at | health check |
| audit_events | tenant_id, id, actor_id, actor_kind, action, target_type, target_id, ip, user_agent, request_id, at, seq, prev_hash, hash | the app role has INSERT/SELECT only (REVOKE UPDATE, DELETE); per-tenant hash chain set by the `audit_events_chain` trigger |

**Read/write split.** `read_session()` binds to the replica engine (`DATABASE_REPLICA_URL`) for list/report endpoints, and `write_session()` binds to the primary. A read-after-write inside one request uses the primary. If the replica is unavailable, reads fall back to the primary and the fallback is logged.

**Pooling.** The SQLAlchemy async pool is size 10 with overflow 10 per process. With 2 api, 2 worker and 1 scheduler processes, that stays well under `max_connections=200`.

**Partitioning.** At the remote-deploy tier, `audit_events` and `jobs` move to monthly range partitions. V1 uses plain tables with `(tenant_id, created_at)` indexes (residual, ADR-09).

## 5. Caching, Async, Compute

- **Cache-aside (Valkey).** Each cached item has a TTL and an invalidation trigger:

  | Cached data | TTL | Invalidated when |
  |---|---|---|
  | Overview dashboard aggregates | 30 s | a job completes |
  | Benchmark summaries | 300 s | (TTL only) |
  | Membership/role lookups | 60 s | a role changes |
- **CDN stand-in.** Caddy serves `/_next/static/*` with `Cache-Control: public, max-age=31536000, immutable`. HTML stays dynamic because of the nonce CSP.
- **Async.** A Postgres `jobs` table with a SKIP LOCKED claim.
  - Each worker runs with bounded concurrency (`WORKER_CONCURRENCY`, default 2).
  - Backpressure: the API returns 429 when a tenant's queued jobs exceed `MAX_QUEUED_PER_TENANT` (default 50) or the global queue exceeds `MAX_QUEUE_DEPTH` (default 5,000).
  - Dead-letter queue: `status='dead'` after 3 attempts. Dead jobs are visible in the UI, and an admin can requeue them.
- **Compute isolation.** SDK execution runs in a `ProcessPoolExecutor` child with timeout `JOB_TIMEOUT_SEC` (default 120; `PROTOCOL_TIMEOUT_SEC`, default 900, for protocol jobs). Workers have a Compose `mem_limit`. The API event loop never runs a simulation.
- **Horizontal scaling.** web, api and worker are stateless (sessions in Postgres, cache in Valkey) and scale via `deploy.replicas`.

## 6. Scheduled Work (has_scheduled_work)

| Trigger | Condition | Action | Effect asserted |
|---|---|---|---|
| T1 scheduled benchmark | `schedules.enabled AND next_run_at <= now` | insert `scheduled_runs(schedule_id, window_start)` (UNIQUE claim) + enqueue `jobs(kind=benchmark)`, advance `next_run_at` | a benchmark job row exists for the window; a second run for the same window creates none |
| T2 stuck-job watchdog | `status='running' AND lease_until < now` | `attempts<3` → `status='queued'`, else `status='dead'` | the job returns to the queue, or goes dead with error "lease expired" |
| T3 retention sweep | `jobs.finished_at < now − RETENTION_DAYS` | delete run_results + old terminal jobs in batches of 500; evidence and audit untouched | old rows are gone; evidence/audit rows remain |

How the triggers run:
- **Entry points and records.** Each trigger exposes `run_tX(now, org_id=None)`, a per-tenant entry point, and records `scheduler_runs(processed, skipped, failed)`.
- **Time.** Time is injected through the `now` parameter, and schedules are evaluated in UTC.
- **Startup.** Missed T1 windows are caught up, at most 1 window per schedule, so there is no burst.
- **Shutdown.** An in-flight sweep completes its current batch.
- **Manual trigger.** A non-production endpoint, `POST /api/v1/admin/scheduler/{trigger}/run` (owner role, `SCHEDULER_TRIGGER_ENABLED=true`), lets the Functional Smoke Test force T1.

## 7. Security Architecture

- **TLS.** Caddy terminates TLS: `tls internal` locally, ACME on a remote deploy. `:80` 308-redirects to `:443`. HSTS is emitted only when `HSTS_ENABLED=true` (remote deploys), so the self-signed local deploy never pins HSTS.
- **Headers.**
  - Next `proxy.ts` generates a per-request nonce and sets the CSP on both the request and the response:
    - `default-src 'self'`
    - `script-src 'self' 'nonce-…' 'strict-dynamic'`
    - `style-src 'self' 'nonce-…'`
    - `img-src 'self' data:`
    - `font-src 'self'`
    - `connect-src 'self'`
    - `frame-ancestors 'none'`
    - `base-uri 'self'`
    - `form-action 'self'`
    - `object-src 'none'`
  - It also sets X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy, and COOP same-origin.
  - The API adds the same headers plus `Cache-Control: no-store` on authenticated JSON.
- **Sessions.** Cookie `__Host-qai_session`: HttpOnly, Secure, SameSite=Lax, Path=/. Expiry is 12 h idle and 7 d absolute. The session is rotated on login and on role change.
- **CSRF.** Double-submit: the `__Host-qai_csrf` cookie (non-HttpOnly) must equal the `X-CSRF-Token` header on every unsafe method, plus an Origin check. Requests using an API key (`Authorization: Bearer`) are exempt.
- **AuthN.**
  - Passwords are hashed with Argon2id (argon2-cffi defaults).
  - The password policy follows NIST 800-63B: at least 12 characters, a check against a bundled common-password list, no composition rules.
  - Login is rate limited (5/min per IP+email, 20/min per IP) and returns a generic error.
- **AuthZ.** A FastAPI dependency `require_role(min_role)` guards every tenant route. Governance approval needs the reviewer capability and `approved_by != author_id`. Foreign tenant IDs return 404.
- **Rate limiting.**
  - Edge: Caddy caps request bodies at 1 MiB and sets request timeouts.
  - App: `limits` async moving-window on Valkey:
    - per IP, global: 300/min;
    - per user: 600/min;
    - job submission: cost-based, weighted by 2^qubits / 2^10, capped.
  - Auth routes fail closed if the store is down; everything else fails open to an in-process fallback.
- **Audit.** Every mutation and security event writes an `audit_events` row in the same transaction: login success/failure, role change, key create/revoke, approval, export.
- **Secrets.** Secrets live in env vars only: `.env` is gitignored and `.env.example` documents them. API keys and session tokens are stored as SHA-256 hashes.
- **Payload safety.** No pickle, eval or exec anywhere. The gate set is allow-listed and params must be finite floats. QASM is export only.

### 7.1 Threat model

The RESEARCH.md §5.1 STRIDE model (TB1–TB8) is adopted in full. This architecture adds these deltas:

| Threat | Control in this architecture | Verified by |
|---|---|---|
| Cross-tenant IDOR (TB7) | RLS + repository tenant filter + 404 | `services/api/tests/test_tenant_isolation.py` hits every resource route with a foreign id |
| Job payload DoS (TB5) | schema caps + Caddy body cap + child-process timeout + mem_limit + cost-weighted rate limit | `services/api/tests/test_limits.py` |
| Evolution gaming (TB8) | server computes V/J from stored seeds; the candidate is data only | `packages/qai-sdk/tests/test_evolution.py` |
| Self-approval of a release (TB3) | `approved_by != author_id` + reviewer capability | `services/api/tests/test_governance.py` |
| Audit tampering (repudiation) | REVOKE UPDATE/DELETE on audit_events from the app role | migration + `services/api/tests/test_audit.py` |
| Scheduler double-execution (TB6) | UNIQUE(schedule_id, window_start) | `services/api/tests/test_scheduler.py` |

## 8. Key Decisions and Alternatives

| Decision | Chosen | Alternatives considered | Why |
|---|---|---|---|
| Simulator engine | NumPy tensordot statevector | Numba kernels; Qiskit Aer as the engine | Pure-Python wheels on Windows; Aer stays a reference so validation is independent |
| Bit order | Little-endian (q0 = LSB), like Qiskit | big-endian textbook order | Avoids the R1 differential mismatch |
| Optimizer | SciPy COBYLA | Nelder-Mead; SPSA | Matches the brief; derivative-free; deterministic for a fixed seed and pinned version |
| Queue | Postgres SKIP LOCKED table | Procrastinate; Redis Streams; Celery | Transactional enqueue with audit, one durability source, no framework |
| Cache/rate store | Valkey 9.1.3 | Redis 8.10 | BSD-3 licence (R9) |
| Rate limiter | `limits` async | slowapi | slowapi's redis extra pins redis<4 (R8) |
| Auth | first-party sessions + API keys | OIDC/Keycloak; JWT | No IdP is mandated; server sessions are revocable |
| Tenancy | shared schema + tenant_id + RLS | schema-per-tenant; DB-per-tenant | Scales to many tenants; RLS adds DB-enforced isolation |
| Read scaling | streaming replica + read/write split | single primary | Required at the large tier |
| Web state | React Context (hard constraint) + server components | Redux/Zustand | Constraint; server components keep client state small |
| Styling | Tailwind 4 `@theme` tokens from the template | CSS Modules | Tokens map 1:1 to `@theme` variables |
| Edge | Caddy 2.11.7 | Traefik; nginx | Automatic local TLS, simple load balancing |
| Theme | single dark theme from the template | light per config; dual | The template is binding (ADR-01) |
| Python | 3.14 images; ≥ 3.12 supported | 3.13 only | Research pin; the local host runs 3.13 (ADR-04) |

## 9. Architectural Invariants

Each rule maps 1:1 to `invariants.json` by id.

| ID | Rule | Reference |
|---|---|---|
| INV-1 | The governance ledger is append-only and every post-plan commit has a decision | GOVERN.md |
| INV-2 | The Web Delivery Baseline report passes and is current | FRONTEND-AUDIT.json |
| INV-3 | The pinned web-baseline runner is present | scripts/web-baseline.mjs |
| INV-4 | Every SPEC §6 screen, endpoint and workflow is reachable in the UI and e2e-tested | SPEC.md §6, apps/web |
| INV-5 | The health route exists | services/api/qai_api/routes/health.py |
| INV-6 | The environment is documented | .env.example |
| INV-7 | The env schema module (fail-fast config) exists | services/api/qai_api/config.py |
| INV-8 | Each scheduled trigger T1–T3 has an effect test (manual) | services/api/tests/test_scheduler.py |
| INV-9 | Scheduled work is claimed idempotently | migration UNIQUE(schedule_id, window_start) |
| INV-10 | No pickle/eval/exec on any Python code path | §7 |
| INV-11 | No `dangerouslySetInnerHTML` in the web app | §7 |
| INV-12 | SQLAlchemy is imported only in db/, models/, repositories/, migrations/ and tests | §2.2 |
| INV-13 | RLS is enabled on tenant tables | §4 |
| INV-14 | The audit table is append-only for the app role | §4 |
| INV-15 | The login route is rate limited | §7 |
| INV-16 | Passwords are hashed with Argon2id | §7 |
| INV-17 | Qiskit is imported only by the reference adapter, benchmarks and tests | §2.2 |
| INV-18 | The CSP nonce is generated in proxy.ts | §7 |
| INV-19 | Caddy terminates TLS | infra/caddy/Caddyfile |
| INV-20 | The template's design tokens are the source of colour | apps/web/app/globals.css |
| INV-21..26 | Every job status the state machine admits (queued, running, succeeded, failed, dead, cancelled) is written by the writer surface | §2.4 |
| INV-27..30 | Every release status (ready, blocked, approved, released) is written by the governance service | §2.5 |
| INV-31..32 | Every evolution decision (accepted, rejected) is written by the evolution service | §2.6 |
| INV-33 | The CI workflow exists | .github/workflows/ci.yml |
| INV-34 | Release version agrees across manifests, lockfile and version defaults | ADR-33 |
| INV-35..36 | Every release decision status (held, rejected) is written by the governance service | §2.5, ADR-54 |
| INV-37 | Engine switches are append-only for the app role | §2.12 |
| INV-38 | The audit hash-chain trigger exists | §2.12 |

## 10. Dependencies (pinned)

- **Python:** numpy 2.5.4, scipy 1.18.1, fastapi 0.162.0, uvicorn 0.54.0, sqlalchemy 2.1.6, asyncpg 0.32.0, alembic 1.20.1, pydantic 2.14.2, pydantic-settings 2.16.2, argon2-cffi 25.1.0, limits 5.8.0, redis 8.1.0, structlog 26.1.0. Optional extra `reference`: qiskit 2.6.1 + qiskit-aer 0.18.2.
- **Python tests:** pytest 9.2.0, hypothesis 6.170.14, httpx 0.28.1, pytest-cov.
- **Web:** next 16.4.3, react/react-dom 19.3.1, tailwindcss 4.3.3, typescript 5.9.x, eslint 9 flat config + typescript-eslint + jsx-a11y + react-hooks, prettier, vitest, @playwright/test.
- **Images:** python:3.14-slim, node:24-slim, postgres:18.6, caddy:2.11.7, valkey/valkey:9.1.3-alpine.

If a pinned version cannot be installed in the build environment, the nearest available release is used and recorded as an ADR (dependency reconciliation).

## 11. Implementation Notes and Risks

- **R1 endianness:** the SDK documents little-endian order; boundary tests use X on q0 and CX direction.
- **R2 reproducibility:** evidence records capture the numpy/scipy versions, platform and seed. Cross-platform comparisons use tolerances.
- **R3/R6 resource caps:** enforced in Pydantic before any allocation.
- **R4 scale:** 50k concurrent applies to API/session traffic. Simulation throughput is bounded by workers, as documented in QUICKSTART. A reduced-scale load test runs locally, with results in REPORT.md.
- **R10 local TLS trust:** the Caddy root CA is exported to `artifacts/caddy-root.crt` for the smoke test (`curl --cacert`). Playwright sets `ignoreHTTPSErrors` for the local origin only.
- **Observability:**
  - JSON logs (structlog) carry the request_id and redact `password|token|authorization|cookie|secret`.
  - `/api/health` reports db, cache, queue and worker checks. The worker check passes when a heartbeat row was updated less than 30 s ago.
