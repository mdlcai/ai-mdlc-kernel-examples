**Round 33 review of F-42 (GD-0211, 20c20b5), F-43 (GD-0212, d2f9cb1), F-44 (GD-0213, 1745f66), F-45 (GD-0214, 134d190) and C-43 (93cd4fc): PASS with conditions**

The four ADR-55 features match ARCHITECTURE §9, invariants.json and SPEC §5 F-42..F-45. The condition is the final `--full` web baseline. It must clear INV-2 (Lighthouse and staleness), and its cells must show the F-45 Design Quality Gate passing at 375, 768 and 1280 for `/app/protocol`, `/app/validation/[id]` and `/app/jobs/[id]`. No finding is medium or high.

**Invariant lint** (`node scripts/invariant-lint.mjs`), summary line verbatim:
`38 total / 37 machine (36 pass, 1 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- One entry is not passing: INV-2 web-baseline, `C:\CODE\quantum\FRONTEND-AUDIT.json`.
  - Detail: `summary.status is "fail" with 2 unaccepted failure(s): landing/lighthouse, app/lighthouse`.
  - The report is stale since gitSha 08cc4ef (28 files).
  - `carried: site/headers accepted per ADR-29`.
- This is the expected state before the final `--full` baseline, so it is not counted against these features.

**Pinned runners:** `git diff 88f92fb HEAD -- scripts/` is empty. `apps/web/app/globals.css` and `DESIGN-TEMPLATE.html` are unchanged.

**Tests:**
- Reviewer:
  - SDK `tests/test_maxcut.py tests/test_protocol.py -m "not slow"`: 79 passed, 1 deselected.
  - Vitest `components/results components/shell`: 66 passed.
- Orchestrator, at HEAD 93cd4fc:
  - full SDK suite: 623 passed, 9 skipped;
  - full API integration suite: 366 passed;
  - web Vitest: 391 passed;
  - `npm run build`, typecheck and lint: clean.

| Invariant / SPEC ref | file:line | Verdict | Note |
|---|---|---|---|
| INV-2 web baseline | `C:\CODE\quantum\FRONTEND-AUDIT.json` | UNCERTAIN (pending full baseline) | Stale; the landing and app Lighthouse cells are unaccepted. Not counted against F-42..F-45. |
| C-43 INV-2 `accepted` site/headers | `C:\CODE\quantum\invariants.json:3` | OK | Accepts one cell and cites ADR-29, which already names it a residual (DECISIONS.md:191, 366). invariants.json is not a pinned runner. |
| Pinned `scripts/*` | `C:\CODE\quantum\scripts\` | OK | No diff in 88f92fb..HEAD. |
| SQL only in repositories | `C:\CODE\quantum\services\api\qai_api\repositories\evidence.py:58-69` | OK | `get_with_result` is the only new query, and it is tenant-filtered. |
| tenant_session / RLS | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:44-46` | OK | The read runs inside tenant_session, with explicit tenant filters on both tables. |
| Cross-tenant 404 | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:40-47` | OK | Foreign, unknown and malformed ids all return 404 NOT_FOUND. |
| RBAC S7 member session | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:36` | OK | `require_member_session`: viewers and API keys get 403 FORBIDDEN, per SPEC §3. |
| Export audit | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:48-57` | OK | `export.evidence_report` with evidenceId and format, in the same transaction. |
| SPEC §7 codes | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:42,47` | OK | Only NOT_FOUND, FORBIDDEN and VALIDATION_FAILED. |
| No-store | `C:\CODE\quantum\services\api\qai_api\services\export.py:47-53` | OK | `attachment_headers` sets no-store and nosniff. |
| cpu.run_bounded | `C:\CODE\quantum\services\api\qai_api\routes\evidence_report.py:58-65` | OK | Rendering is small linear work, the same as `export_evidence`. |
| No pickle/eval/exec, no SDK network, Qiskit only in the adapter | added lines | OK | None found. |
| Size limits | `C:\CODE\quantum\packages\qai-sdk\qai\algorithms\maxcut.py:51`; `C:\CODE\quantum\services\api\qai_api\schemas\jobs.py:549` | OK | ≤ 16 nodes; classicalEvaluations 8–1024; protocol stays at 6–10 nodes. |
| F-42 score_cut independent | `C:\CODE\quantum\packages\qai-sdk\qai\algorithms\maxcut.py:22-32` | OK | Sums edges from the characters and never calls cut_value; errors carry paths. |
| F-42 rescoring | `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:476-479` | OK | Every adopted test record is rescored, and a mismatch fails correctness. |
| F-43 Holm | `C:\CODE\quantum\packages\qai-sdk\qai\protocol_stats.py:31-39` | OK | Step-down with a running max, capped at 1, over 8 comparisons. |
| F-43 verdict | `C:\CODE\quantum\packages\qai-sdk\qai\protocol_stats.py:52-59` | OK | The CI must exclude 0 AND pAdjusted must be < 0.05. Zero diffs give p 1, d_z null and "indistinguishable". |
| F-43 budgets | `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:286-337, 700-706` | OK | Exact `classicalEvaluations` per pair, checked by the baselines test. |
| F-43 -G offline | `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:405-418` | OK | No release or adoption path; used only in the single test evaluation. |
| F-43 AC "never calls gate for ungated" | `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:480-482` | UNCERTAIN | A validity-check `gate()` call runs on the ungated test evaluation (N-R33-2). |
| Phase-A unchanged | `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:641-642, 714-757` | OK | Separate streams and the same phase-A draw order, pinned by a test. The verdict rule changes, as ADR-55 intends. |
| No fabricated metrics / advantage / signed / tamper-proof | added lines; `C:\CODE\quantum\services\api\tests\test_evidence_report.py:22` | OK | None found, and the test asserts it. |
| F-44 determinism / manifest | `C:\CODE\quantum\services\api\qai_api\services\report.py:156-248` | OK | The text is a pure function of the record; generatedAt appears only in the manifest; hashes are over the same UTF-8 text. |
| F-44 commit | `C:\CODE\quantum\packages\qai-sdk\qai\validation\evidence.py:136-141`; `C:\CODE\quantum\services\api\qai_api\worker\loop.py:59` | OK | Matches the regex or becomes `unknown`; the worker passes GIT_COMMIT to the child. |
| F-44 "hash chain verified" | `C:\CODE\quantum\services\api\qai_api\services\report.py:174` | UNCERTAIN | A single-record recompute; the protocol tables are not covered by it (N-R33-1). |
| F-45 INV-11 | `C:\CODE\quantum\apps\web\components\results\protocol-panels.tsx`, `execution.tsx` | OK | Text nodes only. |
| F-45 RBAC downloads | `C:\CODE\quantum\apps\web\app\app\validation\[id]\EvidenceDetailView.tsx:171-193` | OK | Gated by `canWrite(role)`, and the server enforces it again. |
| F-45 nav | `C:\CODE\quantum\apps\web\components\shell\nav.ts:86-90` | OK | Validation Lab has 5 items; no page added. |
| ADR-30 tokens | `C:\CODE\quantum\apps\web\components\results\execution.css`; `protocol-panels.css:20-28` | OK | Only existing tokens are used. The raw `<a class="btn">` download follows existing code. |
| F-45 Design Quality Gate | `C:\CODE\quantum\FRONTEND-AUDIT.json` | UNCERTAIN (pending) | Waits on the `--full` baseline. |

**Findings**
- **N-R33-1 (low):** `C:\CODE\quantum\services\api\qai_api\services\report.py:174`.
  - "Hash chain verified" is a single-record sha256 recompute.
  - The protocol tables and the four protocol JSON files come from RunResult, which the evidence hash does not cover.
  - Add a Limits line saying what the hash covers.
- **N-R33-2 (low):** `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:480-482`.
  - A validity-check `gate()` call runs on the ungated test evaluation, which contradicts the literal F-43 AC.
  - Either reword the AC or use a separate helper.
- **N-R33-3 (low):** `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:493`.
  - The classical stream key has no instance index, so draws are correlated across instances.
  - Fixing it changes the numbers and needs an ADR note.
- **N-R33-4 (info):** `C:\CODE\quantum\packages\qai-sdk\qai\protocol.py:411`. The comment "never passes the gate" is inaccurate.
- **N-R33-5 (info):** `C:\CODE\quantum\packages\qai-sdk\qai\algorithms\maxcut.py:56`. The `while True` loop has no attempt cap, and `_connected` is duplicated.
- **N-R33-6 (pending, not counted):** INV-2 (Lighthouse and staleness) and the F-45 Design Quality Gate AC wait on the final `--full` baseline. This is the pass condition.

**Overall verdict: PASS with conditions** (N-R33-6). N-R33-1..3 can go to a follow-up RB.
