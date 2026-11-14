Commit reviewed: dff13cf99d25cbd89127ad8434a88e33e5f6aae7
Previous round reviewed: 9b9a0cd064f6ed139e05fdcdb4d468b7614fa51c

# Reviewer Gate: Review Round 5 (RB-5, C-7, C-8, F-20)

- **Mechanism:** a fresh-context subagent. It did a static review of the diff and ran the targeted SDK tests, the invariant lint, ruff, one Vitest file and the web typecheck. It ran no e2e, docker, web baseline or API integration tests, and touched no containers. It did not read FRONTEND-AUDIT.json. It edited no source files and staged nothing.
- **Inputs:**
  - artifacts/reviewer/round-4.md (the previous round) and artifacts/design/score-4.md;
  - the diff `git diff 9b9a0cd..HEAD -- . ':!GOVERNANCE-DECISIONS.jsonl' ':!.claude' ':!BUILD-STATE.md'` (41 files). The focus commits are RB-5 e77e485, C-7 bd71622, C-8 46aa25a, and F-20 b198686 + 061d159;
  - SPEC.md (§3 pipeline params, §5 F-17, §6 Design System and Shell, §7), ARCHITECTURE.md §9, invariants.json, DECISIONS.md ADR-35, ADR-36 and ADR-37, the BUILD-PLAN.md wave 10 F-20 `feature:` block, and the CLAUDE.md conventions;
  - the e2e specs in apps/web/e2e (auth, smoke, w4, w5, w8).
- **Re-run here:**
  - `python -m pytest packages/qai-sdk/tests/test_pipeline.py packages/qai-sdk/tests/test_photonics.py -q -p no:randomly`: **59 passed** in 6 s.
  - `npm run lint:invariants`: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web baseline, `files=0`), because FRONTEND-AUDIT.json is absent. This is the known environment blocker, not a code defect. INV-20 (accent `#00D9FF`) and INV-34 (0.3.3 agrees across 10 files) pass.
  - `PATH=/c/CODE/.qai-venv/Scripts:$PATH npm run lint:py`: "All checks passed!" and "150 files already formatted".
  - `cd apps/web && npx vitest run components/shell/nav.test.tsx --maxWorkers=1`: **9 passed**.
  - `npm run typecheck` (`next typegen && tsc --noEmit`): exit 0.
  - **Not run:** the API integration tests (test_jobs_pipeline.py, test_export). No test Postgres or Valkey was provisioned for this review. The new API cases were verified by reading the code and the test only. The Playwright e2e, the web baseline and the smoke script were also not run, because they need the stack.
  - Token comparison: a script parsed the DESIGN-TEMPLATE.html `:root` and every custom property in apps/web/app/globals.css. It normalised case, `var(--rgb-*)` expansion and `0.10`/`0.1`. Every template value is present and equal. The font stacks are the only differences (see the note in the F-20 table).

## RB-5 (e77e485): Review Round 4 remediation (ADR-35)

| Item | Verdict | Evidence |
|---|---|---|
| Huge ints → `OUT_OF_RANGE` (P5-1 / N-R4-1) | PASS | schemas/jobs.py `_bounded` checks the type first, then `lo <= x <= hi`. Python compares an int with a float exactly, so `10**400` fails the range check without converting to float and never raises `OverflowError`. NaN and ±inf also fail the chained comparison. `float()` runs only after both checks pass (jobs.py:139-140). `targets` is `list[Any]` and `tolerance` is `Any`, so pydantic does no float coercion first. In the SDK, `pipeline._targets` and `photonics._finite` (`not abs(x) <= 1e6`) use the same comparison, and the SDK tolerance check (pipeline.py:78) was already comparison-only. Tests: test_pipeline.py adds `10**400` for `targets[0]` and `tolerance`, and test_photonics.py adds `PhaseShifter(0, 10**400)`. Both pass here. The API test adds 400-digit ints at `params.targets[0]`, `params.targets[1]` (negative) and `params.tolerance` (read, not run). |
| Wrong types → `INVALID` (P5-3) | PASS | `_bounded` raises `INVALID` for bool, non-number, `None` and lists before the range check. `INVALID` is the existing field-code default (errors.py:62), so no new code is invented. The API test checks the exact codes: string, bool → `INVALID`; 1.5, huge → `OUT_OF_RANGE`. It also checks `tolerance: null` at `params.tolerance`. |
| `kind=pipeline` export (N-R4-2) | PASS | routes/export_evidence.py:32 `Kind` Literal now includes `pipeline`, matching the evidence kinds. test_jobs_pipeline.py exports `format=json&kind=pipeline` and asserts the new evidence id is present (read, not run). |
| Web kind lists (N-R4-3) | PASS | jobs/_lib/types.ts `JobKind`, jobs/_lib/format.ts, validation/_lib/format.ts, governance/_lib/types.ts and governance/_lib/format.ts all add `pipeline`. Each label map is a `Record<Kind, string>`, so typecheck enforces completeness, and typecheck passes. Each has a Vitest case. |
| Postselection value (N-R4-4) | PASS | pipeline.py stores `abs(1.0 - post)` as the check value with detail `"|1 - P(valid dual-rail)|"`. Pass/fail is unchanged (`abs(1-post) <= EXACT_TOLERANCE`). The stage output still reports the probability, as ADR-35 states. The test asserts `value == |1 - P| <= 1e-10`. This is a chained comparison; it is correct, but it is terse (N-R5-4). |
| Docs (N-R4-5) and ADR-35 | PASS | SPEC §5 F-17 now names `transition_amplitudes(interferometer, occupation)`, matching the code. ADR-35 records each fix and accepts P5-2 and P5-4 with their bounds. The CHANGELOG has an RB-5 line. |

## C-7 (bd71622): wordmark `I AM // QAI` (ADR-36)

| Item | Verdict | Evidence |
|---|---|---|
| Scope matches ADR-36 | PASS | The rename covers the Brand.tsx wordmark (full: `Ψ ◈ I AM // QAI`; compact: `Ψ ◈ QAI`), the root and app layout titles and metadata, manifest.ts, opengraph-image.tsx, SiteFooter, global-error.tsx, the landing title and the template's `<title>` and wordmark. The `qai` package, the API title and the repository docs are untouched (smoke-test.sh still says QAI // QUANTUM SDK), as ADR-36 says. No tokens change in this commit. |
| Accessibility | PASS | Brand keeps `aria-label="I AM // QAI home"`. The decorative glyphs, `//` and the SVG wave stay `aria-hidden`. Below 420 px, site.css hides `.brand-lead` and `.brand-slash`, so the visible text reads `Ψ ◈ QAI`. No e2e asserts the old wordmark or titles (checked with grep). |

## C-8 (46aa25a): ADR-37 Observatory docs and F-20 manifest

| Item | Verdict | Evidence |
|---|---|---|
| Doc-first ordering | PASS | ADR-37 and the SPEC §6 Design System and Shell rows (Space Grotesk, radii 4/8/14, grouped tabs and pill sub-nav at ≥ 1024 px, drawer below) land before the F-20 code, as the CLAUDE.md guardrail requires. The accent stays `#00D9FF` (INV-20). ADR-37 lists what was not adopted (raster art, demo data, the Google Fonts link). |
| Template values | PASS (note) | The DESIGN-TEMPLATE.html `:root` changes to the Observatory values and adds `--color-fg-2`, `--color-flare` (+ soft/line) and `--rgb-flare`. Note: the template's own Google Fonts `<link>` (line 8) still loads Manrope, not Space Grotesk, while its `--font-display` names Space Grotesk. This affects only how the template renders when opened on its own; the app uses next/font (N-R5-3). |
| F-20 manifest | PASS | Wave 10 row plus `feature: F-20` with files_write covering the template, globals.css, layout.tsx, page.tsx, styles/**, components/shell/**, components/site/**, PageHeader.tsx and opengraph-image.tsx. |

## F-20 (b198686, 061d159): Observatory redesign of apps/web

| Item | Verdict | Evidence |
|---|---|---|
| Manifest adherence | PASS (note) | All code changes (globals.css, layout.tsx, page.tsx, styles/{base,ui,shell,observatory}.css, shell/AppFrame.tsx, nav.ts and nav.test.tsx) are inside files_write. Outside it: BUILD-PLAN.md (a one-word correction of the manifest's own `tests:` path, `nav.test.ts` → `nav.test.tsx`) and artifacts/design/score-4.md (a gate artifact). Neither changes behaviour (N-R5-1). |
| Tokens verbatim (ADR-01/30/37, INV-20) | PASS | All 80 template `:root` properties are in globals.css with equal values. globals.css lowercases hex, expands `rgba(var(--rgb-cyan), 0.10)` to `rgba(0, 217, 255, 0.1)`, and prefixes the font stacks with next/font variables (`var(--font-space-grotesk), 'Space Grotesk', system-ui, sans-serif`). This is the same convention the file used before 9b9a0cd. `--color-accent: #00D9FF` stays as a literal under `prettier-ignore`, and INV-20 passes. `--color-error-soft` is extra (it existed before). `--sidebar-w` is now unused (info). |
| Font | PASS | layout.tsx swaps Manrope for `next/font/google` Space_Grotesk (self-hosted, so the nonce CSP is unchanged). The heading weight goes from 800 to 700, because 700 is Space Grotesk's heaviest weight. |
| Styling locality | PASS | No new raw `<button>`, `<input>` or `<select>` in TSX. Buttons use `ButtonLink`, and status uses `StatusPill`. page.tsx uses only semantic markup (`h1`–`h3`, `dl`, `ul`, `Link`), styled through class-scoped descendant selectors in observatory.css (`.hero h1`, `.door strong`), the same pattern site.css already uses (`.site-links a`). Primitive restyles (pill buttons, glassy `.panel`, `.page-header`) are in ui.css, the primitives' stylesheet. Hard-coded `rgba(255, 107, 122, 0.5)` borders match the new `--color-error` channels. The orbit animations are stopped by the global `prefers-reduced-motion` rule (base.css:224). |
| Shell: tenant switcher rendered once | PASS | AppFrame.tsx builds one `switcher` element. It renders it in the drawer when `!desktop` and in the sub-nav bar when `desktop`, never both, so the fixed field id is never duplicated. The SSR snapshot is `desktop=false`, so the server places it in the drawer, which is `display:none` at ≥ 1024 px, and it moves after hydration. That is a brief invisible switcher on desktop before hydration, not a duplicate (info). |
| Shell: aria-current and landmarks | PASS | Group tabs use `aria-current="true"` (a set member, not the page). The sub-nav and drawer links use `aria-current="page"`. Three `nav` landmarks are labelled: "Application" (drawer), "Sections" (group tabs) and "`<Group>` pages" (sub-nav). At ≥ 1024 px the drawer is `display:none`. Below 1024 px the group tabs and sub-nav bar are `display:none` and the closed drawer is `visibility:hidden`. So only one copy of each link is ever in the accessibility tree. The drawer keeps its modal dialog role, label, focus trap, Escape handling and focus return. Single `main#main` and skip link unchanged. 061d159 fixes the design-4 major (the `.topbar .topbar-brand` hide rule now outranks `.brand`). |
| nav.ts groups | PASS | `NAV_GROUPS` is built with `byHref` from `APP_NAV`, which throws on an unknown href. The test asserts that every nav page sits in exactly one group in nav order, and checks `groupFor` for nested and unknown paths. 9 tests pass. Every group tab links to its first item (Workspace → /app, Execution → /app/simulator, and so on). |
| e2e still match (static) | PASS (not run) | auth.spec and w4: `heading level 1 'Overview'` still comes from the page header; the top bar adds no `h1`. w8: on `/app/pipeline` (desktop), the Execution sub-nav renders a visible link `Pipeline` with `href="/app/pipeline"`, which satisfies `.first()`. On the mobile project, link visibility is the same as before F-20 (the closed drawer was already `visibility:hidden`). smoke.spec: the landing page still has one `h1` (`#hero-title`), and its primary CTA `ButtonLink variant=primary href=/signup` carries `data-primary`. No spec asserts the old wordmark, the old landing copy or the sidebar. |
| SPEC §6 type scale | PASS (note) | `.page-title` is 34 px (mobile) and 48 px (≥ 1024 px). SPEC §6 lists h1 40 and display 50, so 48 is off-scale (N-R5-2). |
| Design gate | PASS | artifacts/design/score-4.md: 3/5, rising to 4/5 after the 061d159 fix. The remaining minors and nits (sub-nav tab order after the header, the tall evidence aside, Theory card spacing, the jobs table cutoff at 375, the Refresh baseline) were accepted for V1. |

## Findings

No blocking findings. All are low or info.

- **N-R5-1 (info):** F-20 b198686 edits BUILD-PLAN.md (the F-20 manifest's own `tests:` path) and adds artifacts/design/score-4.md. Neither is in F-20 files_write. Correcting a manifest belongs in a docs (C-) commit.
- **N-R5-2 (low):** `.page-title` is 48 px at ≥ 1024 px (ui.css), which is not on the SPEC §6 type scale (h1 40 / display 50). Use `--text-h1` or `--text-display`, or record the size in SPEC §6.
- **N-R5-3 (info):** DESIGN-TEMPLATE.html line 8 still loads Manrope from Google Fonts, while its `--font-display` is Space Grotesk. The template no longer renders in its own display font. The app is not affected.
- **N-R5-4 (info):** test_pipeline.py `assert v["checks"][1]["value"] == abs(1.0 - p["postselection"]) <= 1e-10` relies on comparison chaining. It is correct, but it reads like a precedence slip. Splitting it into two asserts would be clearer.
- **N-R5-5 (info):** `--sidebar-w: 224px` remains in globals.css, but no rule uses it since F-20. The design-4 minor on tab order (group tabs → Sign out → sub-nav) is still open. It was accepted for V1.
- **Not verified here:** the API integration tests for RB-5 (no test Postgres was provisioned), the Playwright e2e on both projects, and the web baseline (INV-2 has no input).

## Verdict

| Focus ID | Verdict |
|---|---|
| RB-5 | PASS |
| C-7 | PASS |
| C-8 | PASS |
| F-20 | PASS |

**Overall: PASS.** There are no blocking findings: one low (N-R5-2) and four info (N-R5-1, N-R5-3, N-R5-4, N-R5-5).
