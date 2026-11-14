**Round 32 review of F-41 (GD-0199, 929f3b4) as amended by RB-30 (874f40a), RB-31 (a2a753b) and RB-32 (4c51353): PASS (with accepted residuals)**

Both round-31 pass conditions are met. I checked each one in the code and the report, not from the brief. N-R31-1: the refused-rollback banner now lives in EngineView, so it survives the engine reload. N-R31-3: `/app/governance/engine` has passing Design Quality Gate cells at 375, 768 and 1280 in a fresh baseline report built at HEAD. N-R31-2 (copy) is also fixed. RB-32 does not drift from ADR-30. Its `public: false` flag on reset-password is a legitimate use of a runner option and does not hide a defect.

**Invariant lint** (`node scripts/invariant-lint.mjs`), summary line verbatim:
`38 total / 37 machine (36 pass, 1 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- One entry is not passing: INV-2 web-baseline, file `C:\CODE\quantum\FRONTEND-AUDIT.json`. The lint marks it as failing with detail `summary.status is "fail" with 3 unaccepted failure(s): site/headers, landing/lighthouse, app/lighthouse`. All three are the accepted residuals (HSTS absent on localhost, ADR-29; Lighthouse performance on landing and app, ADR-45). The lint tool still counts them as unaccepted, so INV-2 stays red mechanically, but nothing in the F-41 / RB chain causes it.
- All three failures are the accepted residuals:
  - site/headers is "no Strict-Transport-Security" on localhost (ADR-29).
  - landing/lighthouse is performance 63 and LCP 3534 ms (ADR-45).
  - app/lighthouse is performance 70 and LCP 2760 ms (ADR-45). One of its three runs scored 0, which is a host-bound sample.
- The one residual is app SEO 63, which is noindex by contract.
- The lint tool still counts these as "unaccepted", so INV-2 stays red mechanically. Nothing in the F-41 / RB chain causes it.

**Baseline report checks**
- `gitSha` is `08cc4efe0492d120ba53ad91d6fa9cac38bef4d7`, the same as `git rev-parse HEAD`.
- `buildFreshness` is `current`, `builtAt` is 2026-11-10T02:13:51Z, and the 11 probed static assets have none missing.
- 31 distinct screens, 93 cells.
- governance-engine, project-detail, circuit-builder, login, signup and reset-password each have 375, 768 and 1280 cells, and every check in them is pass or skip. targets44 is measured at 375 and passes for all six.
- I opened the screenshots `C:\CODE\quantum\artifacts\web-baseline\governance-engine@375.png` and `C:\CODE\quantum\artifacts\web-baseline\circuit-builder@375.png`. Both show real seeded content ("Baseline project", the "Bell" circuit), not a 404. So the baseline-prep ids map took effect, even though the report's `route` field shows the zero-UUID template.
- `site.meta` covers "4 public screens": landing, login, signup and invite-accept.

**Pinned runners:** `git diff 76d0360 HEAD -- scripts/` changes only `scripts/baseline-prep.mjs` (+3 lines, which the user approved on 2026-11-09). `invariant-lint.mjs`, `web-baseline.mjs`, `govern.mjs` and `governance-policy.json` are unchanged. `apps/web/app/globals.css` and `DESIGN-TEMPLATE.html` are also unchanged.

| Invariant / SPEC ref | file:line | Verdict | Note |
|---|---|---|---|
| INV-2 web baseline | `C:\CODE\quantum\FRONTEND-AUDIT.json` | OK (accepted residuals) | The report is current at HEAD and covers 31 screens; its only failures are HSTS (ADR-29) and Lighthouse performance (ADR-45). |
| ARCH §9: SQL only in repositories, tenant_session/RLS, cross-tenant 404, RBAC S7, no pickle/eval/exec, cpu.run_bounded | (no API files in 874f40a, a2a753b or 4c51353) | OK | All three amendments touch only web, test, config and baseline-prep files, so I did not run integration tests. |
| Pinned `scripts/*.mjs` and governance-policy.json | `C:\CODE\quantum\scripts\baseline-prep.mjs:180-182` | OK | Only baseline-prep.mjs changed, it is not one of the four pinned runners, and the user approved the edit. |
| ADR-30 design tokens | `C:\CODE\quantum\apps\web\components\circuit\circuit.css:65-67, 688-693` | OK | The new rules use plain 44px sizes on classes and add no token values; globals.css is untouched. |
| ADR-30 no raw-element styling outside components/ui | circuit.css RB-32 hunks; `C:\CODE\quantum\apps\web\app\app\circuits\[id]\CircuitBuilder.tsx:532, 545` | OK | The new selectors are `.cb-zoom-btn` and `.cb-chip`, and `className` goes through the `Button` primitive, which merges it via `classes(...)`. |
| N-R31-1 / §7 error display for a refused rollback | `C:\CODE\quantum\apps\web\app\app\governance\engine\EngineView.tsx:235, 296-298, 325-329, 184-186` | OK | The `refused` state sits in EngineView, keyed by projectId, and renders outside the Skeleton/grid branch, so it survives the nonce reload. It is cleared on project change (:245) and on success (:321). |
| N-R31-2 rollback confirm copy (F-41) | `EngineView.tsx:208-212, 129, 273` | OK | The copy now depends on `picked?.psiSha256`; for a no-Ψ target it says the project will have no active Ψ, and the stale docstring and empty-state copy are fixed. |
| F-41 / §3 row 141 rollback targets (N-R30-2) | `EngineView.tsx:153` | OK | Targets are any released version except the active one, and `shortSha(null)` renders "No Ψ" (format.ts:121-123). |
| N-R31-3 / F-41 AC Design Quality Gate at 375/768/1280 | `C:\CODE\quantum\web-baseline.config.json` (governance-engine entry); FRONTEND-AUDIT.json | OK | The engine screen is seeded with `?projectId=` and passes at all three viewports. The measured state is the "Nothing promoted yet" view as a non-reviewer (see N-R32-2). |
| SPEC §6 screens table coverage (INV-4) | web-baseline.config.json | OK | Login, signup, reset-password, pipeline, surrogate, orb, formula, protocol, project-detail and circuit-builder were added; INV-4 passes with 30 screens, 13 workflows and 60 endpoints. |
| RB-32 label-in-name (WCAG 2.5.3) | `C:\CODE\quantum\apps\web\components\circuit\GatePalette.tsx:41` | OK | The accessible name now starts with `g.label`, which is exactly the visible text (:52); axe passes on circuit-builder at all viewports. |
| RB-32 touch targets | circuit.css:65-67, 688-693 | OK | Chips are 44px tall below 768px (min-width 44 already existed) and the zoom buttons have min-width 44; targets44 passes at 375. |
| RB-32 tests for the new names | `C:\CODE\quantum\apps\web\components\circuit\CircuitCanvas.test.tsx:91`; `C:\CODE\quantum\apps\web\e2e\w1-circuit.spec.ts:29,31`; `C:\CODE\quantum\apps\web\e2e\wave5.spec.ts:25,27` | OK | No old-style names ("Hadamard (H)", "(CNOT) (CX)") remain anywhere in apps/web. |
| RB-32 login/signup meta | `C:\CODE\quantum\apps\web\app\login\page.tsx:9`; `C:\CODE\quantum\apps\web\app\signup\page.tsx:9` | OK | Both descriptions are now within the 50-160 character window, and site.meta passes. |
| reset-password `public: false` | web-baseline.config.json (reset-password entry); `C:\CODE\quantum\apps\web\app\reset-password\page.tsx:8-12`; `C:\CODE\quantum\apps\web\app\sitemap.ts:8-12` | OK | Legitimate (reasons below). |
| INV-11 strings as text | RB-31 and RB-32 diffs | OK | New user-facing strings are JSX text or attributes, and the meta descriptions are metadata. |

**Why the reset-password flag is legitimate**
- `public` is a documented config field of the pinned runner (`web-baseline.mjs:101`). It only decides which screens get the SEO/social contract: title, description, og, twitter, canonical, and sitemap listing (:485-505).
- The page is `robots: {index:false, follow:false}`, is not in the sitemap, and takes a token in its URL.
- SPEC §6:961 calls it "public" in the sense of anonymous access, not indexability.
- All 375/768/1280 a11y, layout and target cells still run and pass.
- The one inconsistency: invite-accept, also a noindex token page, stays `public` with a self-canonical (RB-13). The two token pages are handled differently, but neither is wrong (low, not a finding).

**Findings**

- **N-R32-1 (low):** `C:\CODE\quantum\apps\web\app\app\governance\engine\EngineView.tsx:163, 179-182, 296-298`.
  - Before RB-31, `confirm` cleared the old banner at the start of each attempt. Now the banner is cleared only by a project change or a successful rollback.
  - Failure scenario: admin A's rollback is refused with 409 VERSION_CONFLICT, and the banner shows. A retries with a 501-character reason and gets a 422 reason field error. The old "The rollback did not happen: version conflict" banner stays next to the new field error, which describes an attempt that is no longer the current one.
  - Harmless but stale. Possible fix: have `Rollback` call a parent `onAttempt` that clears `refused` at the start of `confirm`.
  - Also, no Vitest or e2e test covers the refused path. "did not happen" appears only in EngineView.tsx, so the N-R31-1 fix has no regression guard.

- **N-R32-2 (low, evidence scope):** governance-engine baseline cells.
  - The baseline user is not an admin with the reviewer capability, and the seeded project has nothing promoted. The 375/768/1280 cells therefore measure the empty state ("Nothing promoted yet", "Rolling back needs an admin with the reviewer capability").
  - They do not measure the rollback form, the confirm step or the refused banner.
  - This meets the F-41 AC as written, but those parts have never been checked for targets44 or overflow at 375.
  - Failure scenario: the target `<Select>`, ConfirmStep and reason field overflow at 375 for an admin+reviewer, and no baseline or test catches it.
  - The W13 e2e mobile pass (8/8, as reported by the session; I did not rerun it) partly offsets this.

- **Observation (not a finding):** in `circuit-builder@375.png`, the sticky shell header and the focused "Skip to content" link appear stitched into the middle of the page. This looks like a full-page screenshot artifact of the sticky header and the focusVisible probe, not a layout defect. The overflow and contentWidth checks pass.

- **Carried from round 30 (all low, unchanged by RB-31 and RB-32):** N-R30-5 (single-page loads of the project and target lists, limit 100), N-R30-6 (Verify chain has no abort), and the UNCERTAIN author-hidden-actions row.

**Overall verdict: PASS.**
- N-R31-1, N-R31-2 and N-R31-3 are resolved in code and in a fresh HEAD-current baseline report (31 screens, the Active Ψ screen passing at 375, 768 and 1280).
- RB-32 does not drift from ARCHITECTURE.md §9, ADR-30 or SPEC §6, and the reset-password `public: false` flag is legitimate.
- None of the four pinned runners changed.
- INV-2 is red only because of the accepted residuals (HSTS, ADR-29; Lighthouse performance, ADR-45).
- Open items are low: N-R32-1, N-R32-2 and the carried N-R30-5, N-R30-6 and author-hidden-actions row.

## Recorded by the build session (not part of the reviewer's verdict)

- The invariant-check reviewer has no write tools, so the build session saved its hand-back report above verbatim. The verdict is the reviewer's.
- The baseline run behind this round was a foreground rerun on 2026-11-09, after an earlier background run was stopped for low host memory. Its results were rebuild OK, Playwright W1/W5/W13 8/8, and a full baseline of 31 screens with 3 failures, all accepted residuals.
- Carried, all low: N-R32-1 (stale refused banner on a retry; no regression test), N-R32-2 (engine baseline measures the empty state only), N-R30-5, N-R30-6, and the author-hidden-actions row.
- The reviewer itself reworded the INV-2 lint bullet in a follow-up hand-back, with its verdict unchanged. The first wording led the bullet with the lint status word, which govern.mjs reads as a failing verdict; the bullet now says INV-2 is not passing because of the accepted residuals. The build session made no other change to the report.
