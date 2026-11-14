**Round 30 review of F-41 (GD-0199, commit 929f3b4): FAIL**

The verdict is FAIL because of two medium SPEC drifts. Both are copy or filter fixes of about one line each in the web code. There is no security or architecture drift.

**Invariant lint** (`node scripts/invariant-lint.mjs`), summary line verbatim:
`38 total / 37 machine (36 pass, 1 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- FAIL INV-2 web-baseline, `artifacts/` web baseline report. The status is "fail" with 8 unaccepted failures, it covers 2 of the 30 screens, and it is stale against gitSha 3f35306. This is the known failure; your rerun is in progress.

**What I checked that holds:**
- **Request shapes match the API:**
  - hold and reject send `{reason}` and approve sends `{reason?}`, against `DecisionIn` and `ApproveIn` with `extra=forbid`, strip and ≤500;
  - create sends `psiRunId`;
  - rollback sends `{releaseId, reason}`, against `RollbackIn`.
- **Response shapes match:** `decision{reason,decidedBy,decidedByName,decidedAt}`, `psiRunId`, `psiSha256`, `psi`, the hold and reject permissions, the engine `active` and `history` fields, and verify `{verified,events,head{seq,hash},brokenAt{seq,id,reason},checkedAt}`. All list limits (50 and 100) are within `le=100`.
- **Which statuses allow hold or reject:**
  - the UI follows the server's `permissions.hold` and `permissions.reject` (`allowed`, `eligible`, `reason`);
  - a held release shows Hold disabled with "This release is already held" and Reject available, which matches HOLDABLE = OPEN and REJECTABLE = OPEN plus held;
  - the section is hidden for rejected and released releases.
- **Rollback gate:** `(admin||owner) && isReviewer` (EngineView.tsx), and the server enforces the same in governance.py `rollback`. Everyone else sees the reason text, as SPEC F-41 requires.
- **Reasons render as text:** all reason and Ψ strings are JSX text nodes, there is no `dangerouslySetInnerHTML`, and the Vitest case covers `<b>` (INV-11).
- **Wording:** "Hash chain verified…" only. The only use of "signed" is the disclaimer "the chain is not signed" (AuditClient.tsx:81), which F-40 and ADR-54 allow.
- **Styling:** only `components/ui` primitives and existing tokens. The tokens used (`--space-1/2/3`, `--color-muted/error/border`, `--radius-md`, `--text-xs/sm`) each exist exactly once in globals.css. globals.css, DESIGN-TEMPLATE.html, `scripts/*.mjs` and `governance-policy.json` are untouched. The Badge `held` entry is inside the primitive.
- **Loaders:** `useLoad` and `usePsiRuns` use AbortController, ignore errors after an abort, and treat a result keyed to a different path or nonce as loading.
- **API-side rules:** there is no SQL, pickle, eval or exec, and no new error codes; the UI only consumes SPEC §7 shapes through `fieldErrors().reason` and `toProblem`.

| Invariant / SPEC ref | file:line | Verdict | Note |
|---|---|---|---|
| INV-2 web baseline | `artifacts/` baseline report | DRIFT | Known: the report is stale and failing, and the rerun is in progress. |
| ARCH §9 SQL, tenant_session, RLS, cross-tenant 404, pickle/eval/exec, cpu.run_bounded | (no API files changed) | OK | The commit is web-only. |
| ADR-30 tokens and `components/ui` | governance.css:463-526, settings.css:361-381, Badge.tsx:36 | OK | Only existing tokens are used, and styling stays in primitives or page CSS. |
| Pinned scripts and policy | `scripts/` | OK | Not touched. |
| SPEC §7 error contract | ReleaseDetailView.tsx:425-433, EngineView.tsx:173-180 | OK | A 422 `reason` field error stays in the confirm step; everything else goes to the problem banner. |
| §3 endpoint shapes (releases, engine, verify, runs) | types.ts, settings/_lib/types.ts | OK | Names and nullability match `release_out`, `_engine_out` and `verify_chain`. |
| F-38 reason limits 1–500 | format.ts `reasonState`, ConfirmStep.tsx | OK | The client is stricter than the server or equal to it (it trims, and counts UTF-16 units against the server's code points). |
| F-38/F-41 RBAC: hold/reject come from server permissions | ReleaseDetailView.tsx ReasonedDecision | OK | `allowed`, `eligible` and `reason` come from the server. |
| F-41 "Disabled actions show the server's reason" (author) | governance.py:274 | UNCERTAIN | The server gives authors `eligible:false` for hold and reject, so the UI hides those actions instead of showing "Authors cannot decide on their own release". This is consistent with the server, but arguably short of the F-41 text. |
| F-39/F-41 rollback (admin + reviewer) | EngineView.tsx (`canRollback`) | OK | Matches governance.py `rollback`. |
| F-39 "a release without a Ψ still promotes" | ReleaseDetailView.tsx:571, :354; EngineView.tsx:58 | DRIFT | The copy says releasing a no-Ψ release does not change the active Ψ; see N-R30-1. |
| F-41 rollback targets | EngineView.tsx:150-152 | DRIFT | Released versions without a Ψ are filtered out; see N-R30-2. |
| F-40/F-41 wording | settings/_lib/format.ts `chainMessage`, AuditClient.tsx | OK | "Hash chain verified: n events, head seq k" and the first break. |
| INV-11 API strings as text | LatestDecision, History, psiLines, trailNote | OK | Text nodes only. |
| Stale and abort handling | EngineView.tsx `useLoad`, GovernanceView.tsx `usePsiRuns` | OK | Keyed results and aborts are handled; see N-R30-3 and N-R30-5 for gaps in other places. |

**Findings**

- **N-R30-1 (medium):** `apps/web/app/app/governance/[id]/ReleaseDetailView.tsx:571` (also :354 and `apps/web/app/app/governance/engine/EngineView.tsx:58`).
  - The panel says: "This release carries no Ψ, so releasing it does not change the active Ψ."
  - SPEC F-39 and the §3 row for POST /releases/{id}/release say a release without a Ψ still promotes, with `psi` null. governance.py:582 calls `_switch(... PROMOTED ...)` unconditionally.
  - Failure scenario: 2.0.0 (with a Ψ) is active. An admin releases 2.0.1 (no Ψ), trusting this text. 2.0.1 becomes active at the next seq with Ψ null, and the Engine screen shows "No Ψ".
  - The no-Ψ release confirm (:354) also leaves out that the version becomes active. The empty state (:58) implies only a release that carries a Ψ promotes.

- **N-R30-2 (medium):** `apps/web/app/app/governance/engine/EngineView.tsx:150-152`.
  - The rollback target list filters on `r.psiSha256`.
  - SPEC F-41 says the targets are "released versions other than the active one", and the API accepts a no-Ψ release (`psi_verifies` is true when both are null).
  - Failure scenario: a project has released 1.0.0 (no Ψ) and has 2.0.0 active. An admin with the reviewer capability is told "No other released version with a Ψ to roll back to" and cannot do a rollback the API allows.

- **N-R30-3 (low):** `apps/web/app/app/governance/engine/EngineView.tsx:179` (Rollback `confirm`).
  - A non-422 failure sets the banner but does not reload the engine or the releases.
  - Failure scenario: another admin switches first, so the server returns 409 VERSION_CONFLICT ("Reload it") or INVALID_STATE. The screen keeps showing the old active version and history, and nothing on the page reloads them.
  - Also, `target` stays set to an id that may no longer be in `options`, so the message reads "Make version  the active Ψ?" with an empty version and resubmits the stale id.

- **N-R30-4 (low):** `apps/web/app/app/governance/_lib/ConfirmStep.tsx:106` and `apps/web/app/app/governance/[id]/ReleaseDetailView.tsx:378`.
  - `fieldError` from the server takes precedence over the live validation and is only cleared on the next submit. Cancel (`close`, :74) does not clear it in the parent.
  - Failure scenario: a 422 at `reason`, then the user corrects the text (or cancels and reopens with an empty box). The old server error is still shown under the field.

- **N-R30-5 (low):** `apps/web/app/app/governance/GovernanceView.tsx:72`.
  - The page fetches `limit=50` runs and filters `succeeded && bestCheckpoint` on the client, and it does not page.
  - Failure scenario: a project whose 50 newest runs are not succeeded (failed or running) shows "This project has no succeeded evolution run yet", although older succeeded runs exist.
  - The same single-page cap applies to projects and released versions (`limit=100`, EngineView.tsx:222 and the Rollback loader).

- **N-R30-6 (low):** `apps/web/app/app/settings/audit/AuditClient.tsx:46` (`verify`).
  - The request has no AbortController and no guard against unmount or a stale response.
  - The Button `loading` state does disable repeat clicks, so the only effect is a state update after navigating away. This is minor.

**Overall verdict: FAIL.** To pass, fix N-R30-1 and N-R30-2:
- correct the no-Ψ wording to say that releasing makes the version active with no Ψ;
- either include no-Ψ released versions as rollback targets, or amend SPEC F-41 with an ADR.

N-R30-3 to N-R30-6 are low and can be deferred. INV-2 stays open until the baseline rerun lands.

## Recorded by the build session (not part of the reviewer's verdict)

- The invariant-check reviewer has no write tools, so the build session saved its hand-back report above verbatim. The verdict is the reviewer's.
- F-41 stays pending REVIEW. RB-30 fixes N-R30-1 (no-Ψ release copy), N-R30-2 (no-Ψ rollback targets), N-R30-3 (reload after a refused rollback; a stale target falls back) and N-R30-4 (a server reason error shows only while the refused text is unchanged). Round 31 reviews F-41 together with RB-30.
- N-R30-5 (single-page run, project and target lists) and N-R30-6 (no abort on Verify chain) are carried. So is the UNCERTAIN author row: the server marks authors ineligible, so the UI hides hold and reject for them, the same as approve's eligibility rule.
