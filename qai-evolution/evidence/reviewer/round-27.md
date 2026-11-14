Commits reviewed: 5b51720 (F-38, release hold and reject with a reason, approve rationale)
Previous round reviewed: round 26 (F-37 PASS)

# Reviewer Gate: Review Round 27 (F-38)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show 5b51720` and checked it against SPEC §2 rows release_candidates / engine_switches, the §3 approve / hold / reject / release rows, §4 S7 / S8, §5 F-38, §7, ADR-54 and ARCHITECTURE §2.5.
  - It ran `node scripts/invariant-lint.mjs`: `38 total / 37 machine (34 pass, 3 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`.
  - The 3 failures are all expected carries:
    - INV-2 `artifacts` web-baseline report: status fail, stale since 3f35306, covers 2 of 30 screens.
    - INV-4 ui-coverage: `/app/governance/engine` has no page.tsx, and there is no W13 spec (F-41).
    - INV-38: no `CREATE TRIGGER audit_events_chain` in `services/api/migrations/**` (F-40).
  - Ruff check and format pass. The reviewer could not run the integration tests, because qai-test-pg was not up in its environment.
  - It edited no repo file. It had no write tools, so the build session saved this report verbatim from its hand-back (see "Recorded by the build session" below).

## Verdict table

| Invariant / SPEC ref | File:line | Verdict | Notes |
|---|---|---|---|
| SQL only in repositories | qai_api/services/governance.py, routes/releases.py | OK | Every write goes through `repo.get` / `repo.set_fields` / `audit.record`. Raw SQL appears only in migration 0014 and tests. |
| tenant_session + RLS | governance.py `_decide`; 0014:83-87 | OK | `_decide` runs in `tenant_session` with `repo.get(..., lock=True)`. `engine_switches` gets `enable_rls_sql` (USING + WITH CHECK). |
| Cross-tenant id -> 404 | governance.py `_decide` | OK | A missing or foreign row gives `not_found()`; the test covers an unknown id. |
| RBAC S7 (approve/hold/reject: member + reviewer, not author; session only) | routes/releases.py `require_session_member`; governance.py `_decide` | OK | `require_role("member")`, then the `kind=="user"` check, then the author check (SELF_APPROVAL), then the reviewer check (FORBIDDEN). An API key gives 403 FORBIDDEN. |
| SPEC §7 error codes | governance.py, routes/releases.py | OK | Only FORBIDDEN, SELF_APPROVAL, NOT_FOUND, NOT_READY, NOT_APPROVED, INVALID_STATE and VALIDATION_FAILED (field codes REQUIRED / TOO_LONG). No new codes. |
| No pickle / eval / exec | all files | OK | None present. |
| CPU-bound work via run_bounded | n/a | OK | No compile or QASM path is touched. |
| Design tokens / pinned scripts | n/a | OK | No web, `scripts/*.mjs` or governance-policy.json change. |
| §5 F-38 hold `pending\|ready\|blocked\|approved -> held`, drops approval | governance.py HOLDABLE, `_decide` | OK | HOLDABLE = OPEN. `approved_by` and `approved_at` are set to None; held -> held gives 409 INVALID_STATE. |
| §5 F-38 resume `held -> ready\|blocked` clears the hold | governance.py `_apply` | OK | A held row clears `_CLEAR_DECISION`, and `_outcome(held, …)` gives READY or BLOCKED. |
| §5 F-38 reject from open or held; rejected terminal (5 actions -> 409 INVALID_STATE) | governance.py REJECTABLE, TERMINAL, `evaluate` / `approve` / `release` / `_decide` | OK | Evaluate (not in EVALUABLE), approve (TERMINAL), hold, reject and release (TERMINAL) each give INVALID_STATE. Released also stays terminal for hold and reject. |
| §5 F-38 approve on held -> 409 NOT_READY | governance.py approve | OK | Checked before the generic NOT_READY. |
| §5 F-38 evidence hook leaves held / rejected | governance.py `on_evidence_written` (OPEN_FOR_REVERT) | OK | Only ready and approved rows are selected. Tested. |
| §5 F-38 decision_* set by approve/hold/reject, cleared by block or resume, kept by approved -> approved | governance.py approve, `_decide`, `_apply` | OK | A block clears the decision in the hook and recheck paths too. An approved row that re-evaluates as passing keeps the approval's decision. |
| §5 F-38 reason 1–500 trimmed; approve rationale ≤ 500 optional | routes/releases.py DecisionIn / ApproveIn / `_reason` | OK | The reason is trimmed before `max_length`. A blank reason gives REQUIRED, 501 characters give TOO_LONG, and a blank rationale becomes None. |
| §4 S8 audit hold/reject/approve reason, from/to | governance.py `_decide`, approve | OK | `release.held` and `release.rejected` carry {version, from, to, reason}. `release.approved` carries the reason. |
| §2 release_candidates CHECKs | 0014:31-37 | OK | psi all-or-none, reason 1–500, reason required while held or rejected, `decided_by <> author_id`. |
| §2 engine_switches | 0014:59-87 | OK | seq ≥ 1, the action enum, reason 1–500 and required for rolled_back, UNIQUE(tenant, project, seq), SELECT/INSERT only (INV-37 passes; the test asserts the exact grant set). |
| Migration downgrade | 0014:90-104 | OK | Refuses while any row is held or rejected, carries a Ψ, or any engine switch exists (SPEC wording). Otherwise drops RLS, the table, the CHECKs and the columns, then restores the old status CHECK. |
| ORM mirrors the DB CHECKs | models/releases.py | DRIFT (low) | See N-R27-1. |
| F-38 AC test coverage | tests/test_governance_decisions.py, test_migrations.py | OK (gaps low) | See N-R27-3. |

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-38 | 5b51720 | **PASS** (0 blockers, 0 major, 5 low notes) | The state machine matches §5 F-38 and ARCHITECTURE §2.5 exactly. Decision fields are set, cleared and kept as specified. Separation of duties holds for the author, viewer, non-reviewer and API key. Codes are within §7. The DB CHECKs match §2, and the downgrade matches its SPEC wording. |

## Findings

- **N-R27-1 (low, model drift):** `services/api/qai_api/models/releases.py` does not mirror the 0014 CHECKs.
  - `ReleaseCandidate.__table_args__` lacks `psi_snapshot`, `decision_reason_length`, `decision_has_reason` and `no_self_decision`, although it does mirror `no_self_approval`.
  - `EngineSwitch` lacks `seq_positive`, `reason_length` and `rollback_has_reason`.
  - The migration is authoritative and no autogenerate or compare-metadata test exists, so this is harmless today. Recommend mirroring them in F-39.
- **N-R27-2 (low, wording):** an API key on hold or reject gets the message "Approval needs a signed-in reviewer" from `require_session_member` (routes/releases.py:97-101). The code (403 FORBIDDEN) is correct. Accepted.
- **N-R27-3 (low, coverage):**
  - A viewer is tested only on hold, not reject.
  - Reject from blocked or approved, and hold from pending, are not exercised directly. Pending -> rejected and held -> rejected are.
  - The 0014 downgrade refusal is tested for a held row and an engine switch, but not for a rejected row or a row with only a Ψ snapshot.
  - The code paths are shared, so the risk is low. Recommend adding these with F-39.
- **N-R27-4 (low, ordering):** in `_decide` (and in the existing approve), the author and reviewer checks run before the state check. So the author acting on a rejected release gets 403 SELF_APPROVAL, not 409 INVALID_STATE. The AC "every action on a rejected release gives 409" is met for an eligible reviewer and admin; 403-before-409 is the existing precedence. Accepted.
- **N-R27-5 (low, note):** the 0014 downgrade drops `decision_reason` / `decided_by` / `decided_at` on approved rows, losing approve rationales, without refusing. This matches SPEC's refusal list exactly (held / rejected / Ψ / switches). Accepted.
- **Carried:** INV-2 (stale web baseline), INV-4 (F-41 screens and W13 not built), INV-38 (F-40 not built). All expected.

## Recorded by the build session (not part of the reviewer's verdict)

- Before 5b51720, the build session ran `pytest tests/test_governance_decisions.py tests/test_governance.py tests/test_migrations.py` (excluding the older 0009–0012 migration tests and downgrade_to_base) against qai-test-pg / qai-test-valkey: 29 passed.
- N-R27-1 and N-R27-3 are addressed in the F-39 commit:
  - the ORM mirrors the CHECKs;
  - there are new tests for the viewer on reject, hold from pending, reject from blocked and approved, and the downgrade refusal with a rejected row.
  - The Ψ-only downgrade refusal shares the same `EXISTS` clause and is still not separately tested.
