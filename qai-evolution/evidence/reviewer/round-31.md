**Round 31 review of F-41 (GD-0199, 929f3b4) as amended by RB-30 (GD-0200, 874f40a): FAIL**

RB-30 fixes N-R30-1, N-R30-2 and N-R30-4 correctly. Its N-R30-3 fix does reload the screen, but that reload breaks something round 30 had marked OK: when a rollback is refused for any reason other than a reason-field 422, the error banner never appears (N-R31-1). Separately, the F-41 acceptance criterion "the new screen passes the Design Quality Gate at 375, 768 and 1280" can't be met yet, because the baseline config doesn't include the Active Ψ screen (N-R31-3).

**Invariant lint** (`node scripts/invariant-lint.mjs`), summary line verbatim:
`38 total / 37 machine (36 pass, 1 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- The only failure is **FAIL INV-2 web-baseline**, from `FRONTEND-AUDIT.json`:
  - summary status is "fail" with 5 unaccepted failures: site/buildFreshness, validation/overflow and site/headers;
  - the report covers 20 screens, but the SPEC UI Surface lists 30;
  - the report is stale against gitSha c25b243, because RB-30 changed ReleaseDetailView.tsx, ConfirmStep.tsx and EngineView.tsx after it.

**INV-2 residual question**
- **validation@375 overflow:** I agree it is a residual. The detail reads "path right=377px clipped by svg" (twice), so the clipping element is the icon's own SVG, not a page container. `/app/validation` was untouched in phase A. I did not inspect the live DOM to confirm the `.table-wrap` ancestor.
- **HSTS (ADR-29) and Lighthouse (ADR-45):** I agree both are residuals. The `app` Lighthouse score of 0 is a host-bound sample.
- **Two INV-2 items are not residuals:**
  - `buildFreshness`: the report predates the build and now also predates RB-30. A rerun is needed.
  - Screen coverage: the report has 20 of 30 screens, and the Active Ψ screen is one of the missing ones (see N-R31-3).

| Invariant / SPEC ref | file:line | Verdict | Note |
|---|---|---|---|
| INV-2 web baseline | `C:\CODE\quantum\FRONTEND-AUDIT.json` | DRIFT | The report is stale since RB-30, 20 of 30 screens are covered, and the three named failures are acceptable residuals. |
| ARCH §9 SQL, tenant_session/RLS, cross-tenant 404, RBAC, pickle/eval/exec, cpu.run_bounded | (no API files in RB-30) | OK | RB-30 changes only web files and the round-30 report. |
| ADR-30 tokens, `components/ui`, pinned scripts and policy | RB-30 diff | OK | No CSS, globals.css, DESIGN-TEMPLATE.html, `scripts/*.mjs` or governance-policy.json changes, and no new raw-element styling. |
| F-39 / §3 POST /releases/{id}/release "a release without a Ψ still promotes" (N-R30-1) | ReleaseDetailView.tsx:354, :571; EngineView.tsx:58 | OK | The copy now says releasing makes the version active with no Ψ, which matches governance.py:582 calling `_switch(PROMOTED)` unconditionally. |
| F-41 rollback targets (N-R30-2) | EngineView.tsx:153-156 | OK | Targets are any released version other than the active one; this matches §3 row 141 and governance.py:793-795 (`psi_verifies` is true when `psi` and `psi_sha256` are both null). |
| N-R30-3 reload and stale target | EngineView.tsx:158, :184-186, :317-320 | OK | A target that is no longer offered falls back to `options[0]`, and the engine and targets reload. |
| §7 error display for a refused rollback (round 30 OK row, now touched) | EngineView.tsx:183-186, :214, :294-296 | DRIFT | The banner is unmounted in the same render that sets it; see N-R31-1. |
| N-R30-4 stale server reason error | ConfirmStep.tsx:51, :80, :90, :110-111 | OK | The server error shows only while the text equals the refused text, Cancel clears it, and Escape keeps both text and error, which is consistent. |
| F-41 Active Ψ copy for no-Ψ targets | EngineView.tsx:209 (also :129, :269) | DRIFT | The rollback confirm still says "the active Ψ" and that "its Ψ hash is checked again" for targets that have no Ψ; see N-R31-2. |
| F-41 AC Design Quality Gate for the new screen | `C:\CODE\quantum\artifacts\web-baseline.config.json` | DRIFT | `/app/governance/engine` is not in the 20 configured screens, so the AC has no evidence; see N-R31-3. |
| INV-11 strings as text | RB-30 diff | OK | All new strings are JSX text nodes. |

**Findings**

- **N-R31-1 (medium), introduced by RB-30:** `C:\CODE\quantum\apps\web\app\app\governance\engine\EngineView.tsx:183-186`, together with `:317-320` and `:294-296`.
  - After any refused rollback other than a reason-field 422, `confirm` calls `setProblem(...)`, `setTarget('')` and `onRefresh()` in one batch.
  - `onRefresh` clears `override` and bumps `nonce`. `useLoad` then returns `loading` for the new key, so `current` becomes undefined and the whole grid is replaced by the Skeleton. `<Rollback>` unmounts, and its `problem` state is lost with it.
  - When the engine loads again, a fresh `<Rollback>` mounts with no problem set, so "The rollback did not happen" (line 214) never renders.
  - Failure scenario: admin B rolls back first, and admin A's rollback returns 409 VERSION_CONFLICT. A's screen flashes a skeleton and then shows B's switch with no message, so A may believe their own rollback happened. The same silent result follows a 403 when the reviewer capability was revoked, a 409 INVALID_STATE when the Ψ no longer verifies, a 404, or a 500.
  - Nothing in e2e or Vitest covers this path.
  - Possible fixes: keep the last ready engine visible during a refresh (don't blank `current`), or move `problem` up into `EngineView`.

- **N-R31-2 (low):** `C:\CODE\quantum\apps\web\app\app\governance\engine\EngineView.tsx:209`.
  - Now that no-Ψ versions are offered as targets, the confirm message "Make version 1.0.0 the active Ψ? Its Ψ hash is checked again…" is inaccurate for them. There is no Ψ to activate or check, and the result is the version becoming active with no Ψ. This is the same class of issue as N-R30-1.
  - Two stale copies remain: the docstring at :129 ("to another released version with a Ψ") and the empty state at :269 ("release a version with a Ψ").
  - Failure scenario: an admin picks "1.0.0 (No Ψ)" and is told its Ψ hash will be checked and become active.

- **N-R31-3 (medium, F-41 AC):** `C:\CODE\quantum\artifacts\web-baseline.config.json`.
  - The baseline config lists 20 screens and does not include Active Ψ (`/app/governance/engine`).
  - SPEC F-41 AC requires "the new screen passes the Design Quality Gate at 375, 768 and 1280". The planned INV-2 rerun cannot produce that evidence however it goes.
  - Failure scenario: INV-2 is closed on the three accepted residuals while the one screen F-41 adds has never been checked at any viewport.
  - Fix: add the screen to the config, which is an artifact and not a pinned script, then rerun the baseline.

- **Carried from round 30:** N-R30-5 (single-page run, project and target lists, now also covering the wider rollback target list), N-R30-6 (Verify chain has no abort), and the UNCERTAIN author-hidden-actions row. All are still low, and RB-30 does not change them.

**Overall verdict: FAIL.**
- To pass, fix N-R31-1 (the refused-rollback feedback must survive the reload) and get Design Quality Gate evidence for `/app/governance/engine` (N-R31-3) in the INV-2 rerun.
- N-R31-2 is low and is a copy fix.
- N-R30-1, N-R30-2 and N-R30-4 are resolved against SPEC F-39, F-41, §3 rows 139 and 141, and governance.py.
- No new drift against ARCHITECTURE.md §9 or ADR-30.

## Recorded by the build session (not part of the reviewer's verdict)

- The invariant-check reviewer has no write tools, so the build session saved its hand-back report above verbatim. The verdict is the reviewer's.
- F-41 and RB-30 stay pending REVIEW. RB-31 (a2a753b) fixes N-R31-1 (the refused-rollback banner moves into EngineView, keyed to the project, so it survives the reload) and N-R31-2 (no-Ψ copy). It also fixes N-R31-3: web-baseline.config.json now lists every SPEC §6 screen, including Active Ψ, and baseline-prep seeds the project-detail, circuit-builder and governance-engine routes. The user approved the baseline-prep edit on 2026-11-09. Round 32 reviews F-41, RB-30 and RB-31 together.
- Carried, all low: N-R30-5, N-R30-6, and the author-hidden-actions row.
