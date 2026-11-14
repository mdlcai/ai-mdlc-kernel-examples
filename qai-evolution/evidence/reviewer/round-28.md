Commits reviewed: 4f24501 (F-39, active Ψ with promote on release and rollback)
Previous round reviewed: round 27 (F-38 PASS, migration 0014 PASS)

# Reviewer Gate: Review Round 28 (F-39)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show 4f24501` and checked it against SPEC §3 rows POST /releases (`psiRunId`), GET /projects/{id}/engine and POST /projects/{id}/engine/rollback, plus the `psiRunId` note (SPEC.md:169). It also checked §4 S7 / S8, §5 F-39 and its ACs (SPEC.md:820-834), §7, ADR-54, ARCHITECTURE §2.12 and the §4 rows for release_candidates / engine_switches.
  - It ran `node scripts/invariant-lint.mjs`: `38 total / 37 machine (34 pass, 3 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`.
  - The 3 failures are all expected carries:
    - INV-2 `artifacts` web-baseline report: stale.
    - INV-4 ui-coverage: `/app/governance/engine` and W13 are not built (F-41).
    - INV-38: no `CREATE TRIGGER audit_events_chain` in `services/api/migrations/**`. The lint inspected 16 files (F-40).
  - INV-37 passes (`0014_decisions_engine.py:80`).
  - `ruff check` reports "All checks passed!" and `ruff format --check` reports "115 files already formatted".
  - It ran `pytest tests/test_engine.py tests/test_governance_decisions.py` against qai-test-pg / qai-test-valkey several times.
    - The first two runs failed because of the environment. Postgres reported `AdminShutdownError: terminating connection due to administrator command`, most likely another python process using the same test database at the same time.
    - Later runs hit 429 RATE_LIMITED on signup, because the repeated runs used up the signup limit.
    - Every test passed in at least one clean run: `test_engine.py` alone gave 6 passed, and a combined run gave all 9 in `test_governance_decisions.py`. No failure pointed to a defect in the code.
  - It confirmed that `with_for_update(key_share=True)` compiles to `FOR NO KEY UPDATE` on the PG dialect.
  - It edited no repo file, because it has no write tools.

## Verdict table

| Invariant / SPEC ref | File:line | Verdict | Notes |
|---|---|---|---|
| SQL only in repositories | qai_api/repositories/engine.py; services/governance.py; routes/engine.py | OK | All engine SQL is in `repositories/engine.py`: `lock_project`, `latest`, `history`, `insert_switch` and `releases_by_id`. The service and routes call only repository functions and `audit.record`. |
| tenant_session + RLS | governance.py:764-797 (`engine_state`, `rollback`), `create`, `release` | OK | Every path runs in `tenant_session(tenant_id, user_id)`; GET uses `read=True`. `engine_switches` has RLS from 0014. |
| Tenant filter in every repository query | repositories/engine.py:29-106; repositories/evolution.py `get_run` | OK | Every new query filters `tenant_id` explicitly. `display_names` (existing) reads global `users` by ids that come from tenant rows. |
| Cross-tenant id -> 404 | governance.py:339, 786; routes/engine.py `_parse_project` | OK | A foreign or unknown project gives 404 on GET and rollback, and a foreign or unknown run gives 404 on create. A release of another project or tenant gives 404. All are tested. |
| RBAC S7 (rollback: admin + reviewer, session only; GET viewer S/K) | routes/engine.py:39-51 (`Viewer`, `Admin`); governance.py:777-780 | OK | `require_role("admin")`, then `kind == "user"`, then `is_reviewer`. A viewer, a reviewer member, an admin without the reviewer capability and an admin API key each get 403 FORBIDDEN (tested). See N-R28-2 for precedence. |
| Release still admin, session only | governance.py:541-543 | OK | Unchanged. Promotion inherits it. |
| SPEC §7 error codes | governance.py, routes/engine.py, routes/releases.py | OK | Only NOT_FOUND, FORBIDDEN, INVALID_STATE, VERSION_CONFLICT, NOT_APPROVED and VALIDATION_FAILED appear. The field codes are INVALID (named by SPEC.md:169), REQUIRED and TOO_LONG. No new codes. |
| No pickle / eval / exec | all files | OK | None present. |
| CPU-bound work via run_bounded | governance.py `psi_digest` | OK | Hashing a bounded checkpoint (canonical_json has max_bytes / max_depth) is not compile or QASM work. |
| Design tokens / pinned scripts | n/a | OK | No web, `scripts/*.mjs` or governance-policy.json change. |
| §3 POST /releases `psiRunId?`; SPEC.md:169 | routes/releases.py:48, 136; governance.py:333-347 | OK | Optional. It must be a run of the same project with status succeeded and an object best checkpoint. Unfinished, failed or no-checkpoint runs give 422 `psiRunId` INVALID; unknown or foreign runs give 404. A malformed id gives 422 `psiRunId`. See N-R28-1 for a run of another project in the same tenant. |
| §5 F-39 snapshot fixed at creation, canonical sha (ADR-16) | governance.py:86-98, 366-385; models/releases.py:73 | OK | `psi_digest = qai.validation.evidence.digest` (sha256 of `canonical_json`). Nothing updates `psi` after insert. `JSONB(none_as_null=True)` stores SQL NULL, which keeps the `psi_snapshot` CHECK satisfiable. |
| §5 F-39 promote in the same transaction, exactly once | governance.py:563-578 | OK | `_switch(PROMOTED)` runs inside the same `tenant_session` as `set_fields(status=released)` and the `release.released` audit. Any raise rolls back the whole transaction (db/scope.py `_scoped`). The release row is locked FOR UPDATE, so a duplicate (sequential or concurrent) re-reads `released` and gets 409 INVALID_STATE, with no second switch. The sequential duplicate is tested. |
| §5 F-39 sha recompute before promote; mismatch writes nothing | governance.py:552-553 | OK | `psi_verifies` runs before `assess`, `set_fields`, any audit and the switch. A mismatch gives 409 INVALID_STATE, and the row stays approved with the active Ψ unchanged (tested). |
| §5 F-39 / ARCH §2.12 lock: project FOR NO KEY UPDATE, then the seq read, then the insert | governance.py:675-698; repositories/engine.py:29-37 | OK | The order is lock, then `latest`, then seq+1, then `insert_switch`. NO KEY UPDATE does not conflict with the KEY SHARE taken by FK inserts (jobs, evidence), so the evidence hook (release FOR UPDATE + project KEY SHARE) cannot deadlock with release(). UNIQUE(tenant, project, seq) is the backstop and maps to VERSION_CONFLICT (line 699). |
| §5 F-39 rollback reads the seq before the lock and compares after it | governance.py:784, 679-680 | OK | `seen` is read before `_switch`. After the lock, `latest` is read again (READ COMMITTED, so it sees the new snapshot) and compared. A mismatch gives 409 VERSION_CONFLICT, with nothing written. |
| §5 F-39 rollback 404 / 409 choices | governance.py:785-791, 681-685 | OK | An unknown project or a release of another project or tenant gives 404. A target that is not released, the active target, or nothing active gives 409 INVALID_STATE. A target whose Ψ fails verification also gives 409 INVALID_STATE, a stricter reading consistent with §2.12. |
| §5 F-39 reason 1–500 required | routes/engine.py:20-28, 49-50 | OK | Trimmed before `max_length`. Missing or blank gives REQUIRED, and 501 characters give TOO_LONG (tested). |
| §3 GET engine shape | governance.py:721-761 | OK | `{projectId, active{seq, releaseId, version, psiRunId, psi, psiSha256, since}\|null, history[...]}`, newest first, at most 50 (`HISTORY_LIMIT`). The field names match the SPEC row exactly. |
| §3 rollback response | governance.py:797 | OK | Returns the engine state from the same (primary) transaction. |
| §4 S8 / §5 F-39 audit detail | governance.py:701-718 | OK | `engine.promoted` / `engine.rolled_back`, target_type `project`, target_id the project, detail `{seq, fromReleaseId, toReleaseId, version, psiSha256, reason}`. Written in the same transaction. `release.created` detail adds `psiRunId` and `psiSha256`. |
| §5 F-39 only governance writes switches; append-only | repositories/engine.py (no update/delete function); 0014:80-81 | OK | The only `insert_switch` caller is `governance._switch`. The grants are SELECT/INSERT (INV-37). See N-R28-3 for the FK cascade. |
| ORM mirrors 0014 CHECKs (N-R27-1) | models/releases.py:87-97, 121-124 | OK | The CHECK texts match migration 0014. |
| Test coverage against F-39 ACs | tests/test_engine.py | OK (gaps low) | All five ACs are exercised. See N-R28-4. |

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-39 | 4f24501 | **PASS** (0 blockers, 0 major, 6 low notes) | Promote happens once and only once, in the release transaction, after the sha re-verification. A mismatch writes nothing. Switches are serialised under the project row lock (NO KEY UPDATE), then the seq read, then the insert, with UNIQUE as the backstop. Rollback detects a lost race through the seq read before the lock and re-checked after it. Status codes match the SPEC rows and §5. RBAC matches S7, and an API key gets 403. The audit detail matches §5. Tenant filters are explicit in every new query. Codes stay within §7. |

## Findings

- **N-R28-1 (low, SPEC wording):** a `psiRunId` that names a succeeded run of another project in the same tenant gives 422 `psiRunId` INVALID (governance.py:340-341; tested at test_engine.py:97-99).
  - SPEC.md:169 says the run must be "of the same project", and "an unknown or foreign run gives 404".
  - The code reads "foreign" as cross-tenant, which matches the 404 rule. The run is visible to the caller, so 422 is defensible.
  - Recommend that SPEC.md:169 say explicitly "a run of another project gives 422 `psiRunId` INVALID". Accepted.
- **N-R28-2 (low, precedence):** routes/engine.py:47-50 parses `project_id` (404), `releaseId` (422) and a blank `reason` (422 REQUIRED) before `governance.rollback` checks `kind == "user"` and `is_reviewer`.
  - So an admin without the reviewer capability, or an admin-role API key, sending a blank reason gets 422, not 403.
  - Viewers and members still get 403 first, from the `Admin` dependency.
  - No data is exposed. Recommend a `require_session_admin_reviewer` dependency, like `require_session_member` in routes/releases.py. Accepted.
- **N-R28-3 (low, concurrency; the code path predates this commit but F-39 makes it matter more):** routes/projects.py:289-303 `delete_project` runs the unlocked `has_released_release` check and then the DELETE.
  - A concurrent release() can commit between the two. Its DELETE then cascades `release_candidates` and `engine_switches` (`project_id ... ondelete="CASCADE"`, 0014:63). Postgres runs RI cascades as the table owner, so they bypass the SELECT/INSERT-only grant.
  - The result is that the append-only switch history of a released project is removed, against the HAS_RELEASES rule.
  - Separately, release() locks the release row and then the project row, while DELETE locks the project row and then cascades to the release rows. Postgres detects the deadlock and aborts one side, which then shows as a 500.
  - Recommend that `delete_project` lock the project row FOR UPDATE before the has-released check, as a follow-up outside F-39.
- **N-R28-4 (low, coverage), tests/test_engine.py:**
  - `test_concurrent_rollbacks...` (line 237) also passes on `[200, 200]`. So neither the `check_expected` VERSION_CONFLICT branch (governance.py:679) nor the `SwitchConflict` mapping (line 699) is exercised deterministically. Recommend forcing the interleaving, for example by holding the project row lock from `owner_conn` while two rollbacks start.
  - The "nothing has been promoted" branch (governance.py:682) is untested. It is reachable only for a release that was released before F-39 and has no switch.
  - Untested: rollback to a released target whose Ψ was tampered, and a tampered `psi_sha256` (as opposed to a tampered `psi`).
  - The `engine.promoted` audit detail (`fromReleaseId`, `version`, `psiSha256`) is not asserted.
  - The mismatch test does not assert that no `release.released` audit row was written.
  - A concurrent duplicate release request is not tested.
  - The SPEC §8 test map lists "append-only" under test_engine.py, but that assertion lives in test_migrations.py.
  - The Ψ-only downgrade refusal (carried from N-R27-3) is still not separately tested.
- **N-R28-5 (low, payload):** `release_out` (governance.py:215-217) now returns the full `psi` on every item of GET /releases. The §3 row does not fix the release shape, and checkpoints are small (generation, objective, params). Accepted.
- **N-R28-6 (low, note):** GET /projects/{id}/engine reads with `read=True` (governance.py:766), so a read right after a switch may lag on a replica. The rollback response itself comes from the primary transaction. Accepted.
- **Carried:** INV-2 (stale web baseline), INV-4 (F-41 screens and W13 not built), INV-38 (F-40 not built). All expected.

**Verdict: PASS**

## Recorded by the build session (not part of the reviewer's verdict)

- The reviewer has no write tools. The build session saved this report verbatim from its hand-back.
- The environmental failures the reviewer saw (AdminShutdownError, then 429) came from the build session running F-40 tests on the same `qai_test` database at the same time. Later build-session runs use `TEST_DB_NAME=qai_test_f40`.
- Before 4f24501, the build session ran test_engine.py, test_governance_decisions.py, test_governance.py and test_migrations.py: 84 passed, plus the known clock-drift flake in test_evolution_api.py, which passes 3/3 in isolation.
- Follow-ups: N-R28-3 (lock the project row in `delete_project`) and N-R28-4 (a deterministic rollback race and the other coverage gaps) are carried. They are low and outside F-39.
