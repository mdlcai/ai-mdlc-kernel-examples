Commit reviewed: 3a9cb5c2294900a1d9c82ec02884f969ef470113
Previous round reviewed: c019a8ea84f285a79a1d0fbb714fc2c84f2df44f

# Reviewer Gate: Review Round 2 (diff-scoped)

- **Mechanism:** a fresh-context Task-tool subagent. Static review, plus two targeted pytest files. No e2e, docker or web baseline was run, and no containers were touched.
- **Inputs:**
  - artifacts/reviewer/round-1.md (previous round; its FAIL item texts);
  - the diff c019a8e..3a9cb5c. The code is all in fix batch 442f4af; 3a9cb5c only touches GOVERNANCE-DECISIONS.jsonl;
  - SPEC.md, ARCHITECTURE.md and invariants.json;
  - the kernel Reviewer Gate section;
  - every source file the diff references. DECISIONS.md ADR-30 was read only to locate the batch's intent; the verdicts are derived from the code.
- **Re-run here:**
  - `pytest tests/test_logging.py tests/test_members.py -k "reset or redact"`: 4 passed;
  - `ruff check qai_api tests`: clean;
  - `ruff format --check`: 104 files formatted.
- **Taken as established:**
  - narrow API suite 122/122 on the changed modules;
  - web tsc, eslint and prettier clean;
  - vitest 109/109.

## Round-1 FAIL items re-derived

| Round-1 item | Verdict | Evidence |
|---|---|---|
| Item 6: cold read, no obvious bugs, dead code or broken wiring. (a) a duplicate, divergent T2 watchdog (`services/jobs.py recover_expired_leases`) that production never calls | **PASS (fixed)** | `recover_expired_leases` and `repositories/jobs.py expired_leases` are deleted. A repository-wide grep finds no remaining caller or reference outside DECISIONS.md. The false-coverage test `test_expired_leases_are_recovered` is removed from tests/test_worker.py. The module docstring now points to `scheduler/triggers.py` (services/jobs.py:16). The only T2 is `scheduler/triggers.py:222` `_t2_tenant`, which runs per tenant through `repositories/schedules.py:203 lock_expired_leases`. T2 coverage remains in tests/test_scheduler.py:230 and :251. ruff is clean, so the deletion left no orphaned imports. |
| Item 6: (b) the overview cache is not invalidated on the T2 dead path or on requeue | **PASS (fixed)** | In scheduler/triggers.py:244-245, `if rows: await overview_cache.invalidate(tenant_id)` runs after the `async with tenant_session` block exits. That is after commit, and it covers both the queued and the dead branch. requeue invalidates at services/jobs.py:323, also after its session block. `overview_cache.invalidate` never raises (services/overview_cache.py:67-72), so neither path can 500 or abort T2 after commit. Tests assert the generation bump: tests/test_jobs.py:296-297 for requeue from both failed and dead, and tests/test_scheduler.py:236-239 for T2. |

## Regression scrutiny of fix batch 442f4af

- **`_issuer_still_scoped` (routes/auth.py:288-302) and its RLS GUC changes:** sound.
  - Only `memberships` has RLS among the tables this path touches, with the policy `TENANT_PREDICATE OR OWN_USER` (migrations/versions/0002_auth.py:73-75). `users`, `sessions` and `password_resets` have no policy.
  - The first `rescope(tenant=None, user=target)` lets `tenants_for_user` see the target's own rows. The second `rescope(tenant=t, user=target)` lets `membership(t, issuer)` see the issuer's row through the tenant predicate.
  - The code that follows does not depend on the GUCs: `set_password_hash` (users) and `sessions.revoke_user` (sessions) run on tables without RLS.
  - `password_reset` then explicitly re-runs `rescope(None, user)` and re-scopes per tenant before each `audit.record` (auth.py:327-339). The audit loop therefore starts from a clean GUC state and is unaffected.
  - On a False result, the raised `ApiError` rolls back the transaction (db/scope.py:67-69), so the claim is undone. The token stays unused but can never pass, and member removal now voids it (routes/members.py:171).
  - The role logic matches the issue-time rule at routes/members.py:211-212: the issuer must be admin or owner, and an owner target needs an owner issuer. Self-issued tokens and tokens with a null issuer pass.
  - Tests cover both paths. The positive path is tests/test_members.py:135, which still returns 204 and the login works. The cross-workspace path and the removal path are tests/test_members.py:173-193.
- **Invalidation on removal and invite accept:** sound.
  - Both call `repositories/access.py:163 invalidate_password_resets`, a no-RLS update keyed only by user_id: routes/members.py:171 and routes/invites.py:234.
  - The invite call sits inside the claim transaction, after `add_membership`.
- **Compile offload (routes/circuits.py:129-141):** thread-safe.
  - `_load` commits and closes its session before the thread starts. The sessionmaker uses `expire_on_commit=False` (db/engine.py:41), so `circuit.num_qubits` and `circuit.ops` (column and JSONB values, no lazy relationships) are already-loaded plain attributes. The thread only reads them.
  - The ORM session is never touched off the loop.
  - An `ApiError` from `to_sdk` propagates through `asyncio.to_thread` unchanged.
  - `enforce_compile` uses `Rule("compile:principal", "30/minute")`, which fails open to the in-process limiter (services/ratelimit.py:70-94,119-120). It runs before `_load`, so requests for an unknown circuit also consume budget. That is acceptable.
- **SSR X-Forwarded-For (apps/web/lib/api-server.ts:19-45, app/invite/[token]/page.tsx:28):** correct.
  - Only Caddy publishes ports (compose.yaml:46-48). web and api are internal, so an attacker cannot reach web:3000 directly with a forged header.
  - Caddy 2.11.7 (compose.yaml:41) sets no `trusted_proxies` (infra/caddy/Caddyfile). It therefore replaces any client-supplied XFF with the peer address, and the value web forwards is the real client IP.
  - uvicorn's `--forwarded-allow-ips "*"` (services/api/Dockerfile:30) then sets `request.client`, which `deps.client_ip` reads (deps.py:64-65). The ADR-30 residual about exposing the API port is noted.
  - The caller's `options.headers` are merged after the forwarded header (`api.ts:157-159` spreads the options), so nothing is clobbered.
- **Log redaction (qai_api/logging.py):** effective.
  - `redact` runs after `format_exc_info` (logging.py:71-72), so the rendered tracebacks are strings when they are scrubbed.
  - The `_API_KEY` pattern matches the real key format `qai_<8 alnum>_<token_urlsafe(32)>` (routes/api_keys.py:47-48,88,118).
  - The DSN pattern can over-redact a URL whose path contains `@`, which only costs log detail. That is harmless.
- **CSS and TSX:** presentational only: tokens, uppercase headings, a `min-width: 0` brand, and an email cell with a `title` attribute. No logic or wiring changed.

## Checklist (current tree)

| # | Item | Verdict | Rationale and evidence |
|---|------|---------|------------------------|
| 1 | Every RESEARCH requirement is implemented and traceable, with no stubs | PASS | The batch removed no capability. The simulator, compile, validation, algorithms, queue, scheduler, governance and exports are unchanged since round 1. Compile still runs the real SDK pipeline (routes/circuits.py:138), now in a thread. |
| 2 | Every SPEC feature has code and behavioural tests | PASS | The changed behaviours have DB- or cache-effect tests: requeue and T2 invalidation (test_jobs.py:296, test_scheduler.py:238), reset scope at claim time (test_members.py:173), and redaction (test_logging.py:8). T1, T2 and T3 coverage is intact. Minor gap, non-blocking: the new compile 30/min limit has no 429 test. It is a hardening knob under SPEC S5 ("any limit … 429"), not a SPEC feature. |
| 3 | Implementation matches ARCHITECTURE | PASS | No new module or layer was added. The diff consolidates T2 into `scheduler/`, which removes a shadow path. The round-1 doc note N-1 about the `jobs/worker.py` path is unchanged and stays a doc note only. |
| 4 | Scope discipline | PASS | The batch's additions are security hardening of existing SPEC surfaces (S5 rate limits, S13 redaction, reset scoping) and design-token alignment. No user-visible feature was invented. |
| 5 | Security invariants upheld | PASS | See the regression section. The reset scope is now enforced both at issue time and at claim time. Compile no longer blocks the event loop and is rate limited. The SSR per-IP bucket is correct. Secrets are scrubbed from values and tracebacks. No hardcoded secrets were introduced. |
| 6 | Cold read: no obvious bugs, dead code or broken wiring | PASS | Both round-1 findings are fixed (table above). No regression was found in the batch, and ruff is clean. |
| 7 | Runnable (README/QUICKSTART) | PASS | The batch did not touch README or QUICKSTART. The postcss pin bump is reflected in package-lock.json, which the batch also updated. |
| 8 | Every manual invariant upheld | PASS | INV-8 is the only manual invariant. Verdict below. |
| 9 | Nothing declared is unreachable | PASS | Every job state keeps a production writer. The deleted function was a duplicate writer of queued and dead, and T2 still writes both (triggers.py:234-242). No state, field or config was added without a writer. `COMPILE_PRINCIPAL` is read by `enforce_compile`, which routes/circuits.py:133 calls. |

## Manual invariants

| Invariant | Verdict | Evidence |
|-----------|---------|----------|
| INV-8: each scheduled trigger has a business-effect test | PASS | Each test asserts real DB state:<br>- T1: `test_t1_enqueues_benchmark_job` at services/api/tests/test_scheduler.py:70 (job row enqueued, next_run_at advanced) and `test_t1_idempotent_same_window` at :100 (UNIQUE window claim).<br>- T2: `test_t2_requeues_expired_lease` at :230 (status queued, "lease expired", LEASE_EXPIRED, and now the cache generation bump) and `test_t2_dead_after_max_attempts` at :251.<br>- T3: `test_t3_deletes_old_results_keeps_evidence_and_audit` at :298. |

## Findings

None blocking.

- **N-R2-1 (test gap, LOW, non-blocking):** POST /circuits/{id}/compile has no test that reaches the new `compile:principal` 429. Consider one test that patches `COMPILE_PRINCIPAL` to "1/minute".
- **N-R2-2 (doc):** the round-1 notes N-1 and N-2 (ARCHITECTURE path and seed-protocol wording) are still open. They are documentation only.

## Overall verdict

**PASS.** Both round-1 FAILs (item 6: F-R1-1 and F-R1-2) are fixed at the root, and fix batch 442f4af introduced no regression. All nine items and INV-8 are PASS.
