# Review Round 4: RB-5..RB-7

- **Commits reviewed:** `3fee33a..5fc6924`: RB-5 `826c475` (live pulse: reduced motion and outline ring), RB-6 `c7d8d3f` (mobile drawer panel mounts only while open), RB-7 `5fc6924` (role="status" on 26 loading skeletons). HEAD reviewed: `5fc6924`.
- **Scope:** diff-scoped (`git diff 3fee33a..5fc6924 -- web server compose.yaml Caddyfile SPEC.md ARCHITECTURE.md`; 25 files, +60/-49, all under `web/`). Focus item from governance decision GD-0023: is RB-7's ARIA fix correct?
- **Date:** 2026-10-05
- **Reviewer:** independent Reviewer (read-only, Task subagent)
- **Inputs used:** RESEARCH.md, ARCHITECTURE.md §9, SPEC.md, the diff and the code around it, invariants.json and `npm run lint:invariants`, FRONTEND-AUDIT.summary.json (gitSha `c7d8d3f`, which is before RB-7), artifacts/web-baseline/*.png, docs/previews/*.png. I did not read DECISIONS.md, CHANGELOG.md, REPORT.md, DOGFOOD.md, BUILD-STATE.md or earlier reviewer rounds.

## Checks run

| Check | Result |
|---|---|
| `npm run typecheck` | exit 0 |
| `npm run lint` (eslint, `--max-warnings 0`) | exit 0 |
| `cd web && npx vitest run` | 8 files, 30 tests passed |
| `npm run lint:invariants` | exit 1: `43 total / 42 machine (41 pass, 1 fail, 0 zero-files) / 1 manual`. The one failure is INV-2 (web baseline must pass and be fresh). |
| FRONTEND-AUDIT.summary.json @ `c7d8d3f` | `fail`. Failing cells: dashboard@375 axe `aria-prohibited-attr` on `.lg:grid-cols-[minmax(0,1fr)_minmax(0,2fr)]` (the RB-7 target); 7 `ERR_TIMED_OUT` load aborts (account ×4, integrations, rules, rule-new); asset-detail@1280 CWV LCP 2900 ms; dashboard Lighthouse performance 75 (< 80). There are no mobile-nav or reduced-motion failures. |
| Static scan of every `web/src/**/*.tsx` (not tests) for `aria-label` on div/span/p/i/b/em/strong/small/code/pre with no `role`, multi-line tags included | 1 hit: `web/src/app/(app)/search/SearchView.tsx:230` (R4-2) |
| Search for `getByRole('status')`, `aria-busy` and `Loading …` selectors in `web/e2e` and `*.test.tsx` | 2 hits, neither ambiguous (see below) |

INV-2 context: the baseline on record is from `c7d8d3f`, before RB-7, and it fails. A fresh run is in progress separately. Nothing in this diff causes a baseline failure. RB-7 targets the one axe failure, and the other failures are load timeouts or performance measurements the diff does not touch (but see R4-3).

## Focus item: RB-7 `role="status"` on loading skeletons (GD-0023)

**Is `role="status"` the right fix?** Yes, it is a valid and proportionate fix.
- axe's `aria-prohibited-attr` fires because `aria-label` is prohibited on the `generic` role, which is what a role-less `div` maps to.
- `status` allows an author-supplied name, and `aria-busy` is a global attribute, so `<div role="status" aria-busy="true" aria-label="Loading …">` is conformant.
- The axe selector in the summary is `LEAD_GRID` (`web/src/features/dashboard/DashboardView.tsx:162`), which is exactly the node changed at `DashboardView.tsx:169`.
- I counted 26 call sites in the diff, matching the commit message: AlertsView 1, AlertDetail 2, AssetsView 1, AssetDetail 2, CorrelationsView 1, CorrelationDetail 1, EventsView 2, IntegrationsView 1, IntegrationDetail 1, IntelView 1, NotificationsView 1, LogView 1, ReportsView 1, ReportDetail 1, RulesView 1, RuleDetail 1, SearchView 1, AuditView 1, JobsView 1, OrganizationView 1, TeamView 1, SourceProfile 1, DashboardView 1.

**Alternatives considered:**
- Dropping `aria-label` and keeping `aria-busy` would also satisfy axe, which is the pattern `MapView.tsx:84` uses. But the loading container would then have no name in the accessibility tree.
- `role="progressbar"` (indeterminate) with a label is also valid. It is not a live region, though, so it gains nothing over `status`.

`status` keeps the name and is the conventional loading pattern, so it is the better of the three.

**Announcement noise:** none.
- Every child is `Skeleton`, which renders `aria-hidden` (`web/src/components/ui.tsx:96`), so the live region has no text content.
- Live regions announce changes to content, not the region's name, and `aria-busy="true"` (which is never cleared, because the node is removed) suppresses announcements anyway.
- In practice the skeletons are silent. A screen reader user can find them by name in browse mode, but they will not hear them announced.
- The commit message claims the change "announces the busy region". That overstates it, but it is harmless (R4-1).

**Nesting:** none of the 26 sits inside another live region, `role=status` or `role=alert`.
- The nearby live regions are siblings, not ancestors: `JobsView.tsx:119` (`aria-live` result banner, sibling of the skeleton at `:135`), and `IntegrationDetail.tsx:422` and `:506` (sub-panel regions, while the skeleton at `:85` is an early `return`).
- `rules/_parts.tsx:624` is likewise a sibling, because `RuleDetail.tsx:66` is an early return.
- The `EventsView.tsx:569` skeleton sits inside `role="region"` within a dialog. That region is not live.
- The `AlertDetail.tsx:752` `TabState` renders inside a tab panel. That panel is not live.
- On the dashboard, the skeleton at `:169` and `LiveControl`'s `role="status"` (`widgets.tsx:102`) are separate, independent regions.
- In SearchView, the "Searching" skeleton (`:195`) and the results `<p role="status">` (`:220`) are mutually exclusive branches, so nothing is announced twice.

**Test and e2e selectors:**
- `web/e2e/w10-search.spec.ts:69` uses `getByRole('status').filter({ hasText: 'results for' })`. The skeleton has no text, so it never matches, and it has already unmounted by then.
- `web/src/components/ui.test.tsx:44` renders `Alert` in isolation.
- No test selects `[aria-busy]` or a "Loading …" label.
- No selector becomes ambiguous.

**Is the same pattern left anywhere?** Once, outside the skeletons:
- `web/src/app/(app)/search/SearchView.tsx:230` is a `<span aria-label="{n} results">` with no role (R4-2).
- axe-core reports this as "needs review" rather than a violation because the span has text content, which is why the search screen passes the baseline. The label is still discarded, so assistive technology reads just "1" or "5+" under each group heading (confirmed in `artifacts/web-baseline/search@375.png`).
- No other span, section, ul, li or p is affected: `section` and `ul`/`li` with `aria-label` map to region, list and listitem, which all allow a name.

**Maintainability:** the 26 identical skeleton wrappers are copy-pasted. A shared `LoadingRegion` primitive in `ui.tsx` would have made this one edit and would stop the pattern from regressing (R4-4).

## RB-5 / RB-6 assessment

**RB-5** (`web/src/app/globals.css:96` and `:137-140`): correct.
- The reduced-motion override `.tw-live::after { animation: none }` is not in a layer, so it beats the `@layer components` declaration at `:99`.
- The global `*::after` `!important` rule sets only duration and iteration count, so `animation-name: none` holds and the ring stays at `opacity: 0`.
- This meets SPEC.md:578 ("everything is disabled under `prefers-reduced-motion`").
- The outline ring (`border: 1.5px solid var(--color-pulse)`) uses only a colour token.

**RB-6** (`web/src/components/AppShell.tsx:113-145`): correct, and I found no regressions.
- The root `#tw-nav-drawer` always renders (`:116`), so the toggle's `aria-controls` (`:328`) always resolves to a real element.
- The panel mounts in the same commit that sets `open`, so the focus effect (`:70-79`) finds `panelRef.current` and focuses the first link.
- Escape and the Tab trap (`:87-109`) are unchanged, and `close()` still returns focus to the toggle.
- Closing on a route change (`:302`) now unmounts the panel. Before, the panel was hidden with `display:none`, and focus ends up in the same place either way.
- Server and client rendering now agree: neither renders the closed panel.
- `web/e2e/auth.spec.ts:11` uses `navigation "Primary" .first()`, which still resolves now that only the sidebar copy exists.
- The baseline's `checkMobileNav` (`scripts/web-baseline.mjs:331-355`: open, every destination reachable, Escape closes and restores focus to the toggle) has no failures at `c7d8d3f`, which is a run after RB-6.
- The stated purpose, dashboard Lighthouse performance at 80 or above, did not happen in the measurement (R4-3).

## Checklist

| # | Item | Verdict | Rationale |
|---|---|---|---|
| 1 | RESEARCH.md requirements implemented, not stubbed | PASS | Scoped: the diff touches only presentation and a11y (`web/src/**`, `globals.css`) and changes no capability, server path or data flow, so the earlier rounds' verdict on this item still holds. |
| 2 | SPEC features have code and behaviour tests | PASS | SPEC.md:585 (drawer: toggle, focus trap, Escape) is covered behaviourally by `scripts/web-baseline.mjs:331-355`, which passes at `c7d8d3f`; SPEC.md:578 (reduced motion) is enforced by the baseline cells, with no reduced-motion failure at `c7d8d3f`. There is no vitest or e2e test of the drawer itself, and none asserts skeleton semantics (R4-4, advisory). |
| 3 | Matches ARCHITECTURE.md (no shadow modules or layers) | PASS | All changes stay inside the existing `AppShell.tsx` component, the view files and `globals.css`, with no new module or dependency (diffstat: 25 files under `web/src`). |
| 4 | No invented features (scope discipline) | PASS | RB-5 to RB-7 change only rendering, motion and ARIA semantics of existing SPEC screens (SPEC.md:578, :585) and add no new behaviour. |
| 5 | Security invariants upheld | PASS | The diff adds no secrets, inputs, endpoints, headers or CSP changes, and the drawer change only removes markup (`AppShell.tsx:116-144`); lint:invariants passes every security INV (INV-41, INV-42 and the rest). |
| 6 | Cold read: no obvious bugs or broken wiring | PASS | The drawer focus, Escape and Tab-trap effects still find their refs when open (`AppShell.tsx:70-109`), the reduced-motion cascade resolves to `animation:none` (`globals.css:137-140`), and none of the 26 `role=status` wrappers is nested in another live region. |
| 7 | Runnable and Verification Gate commands reproduce | PASS | `npm run typecheck` (0), `npm run lint` (0) and `npx vitest run` (30/30) reproduce on `5fc6924`; INV-2 fails only because the baseline is stale or failing, which is recorded as context and not caused by this diff. |
| 8 | Manual invariant INV-11 (one effect test per trigger T1-T7, with injected `now` and business effect plus `job_runs` counts) | PASS | `server/test/jobs/` has one test per trigger, each calling `run*ForOrg` with an injected `now` and asserting a row-level effect plus counts: detection-sweep.test.ts:68-80, email-drain.test.ts:73-80, delivery-drain.test.ts:89-96, escalation-check.test.ts:32-45, ingestion-health.test.ts:91-95, retention-sweep.test.ts:39-56, report-generate.test.ts:63-73 (the diff does not touch `server/`). |
| 9 | Nothing declared is unreachable | PASS | RB-6 keeps the `aria-controls` target `#tw-nav-drawer` mounted (`AppShell.tsx:116`) and the dialog is reachable through the toggle (`:327-330`); the diff declares no new state, field or control. |

## Findings

| ID | Severity | Location | Finding |
|---|---|---|---|
| R4-1 | LOW (advisory) | `web/src/components/ui.tsx:96`; the 26 RB-7 sites, e.g. `DashboardView.tsx:169` | The fix is valid for axe and causes no announcement noise. But the skeleton regions are effectively silent: every child is `aria-hidden`, the region mounts already populated, and `aria-busy="true"` is never cleared. The RB-7 commit message claim that it "announces the busy region" is therefore overstated. If an audible loading cue is wanted, put visually hidden text (`<span className="sr-only">Loading …</span>`) inside the region. No change is required for conformance. |
| R4-2 | LOW | `web/src/app/(app)/search/SearchView.tsx:230` | This is the last label-without-role case: `<span aria-label="{n} results">` on a generic element. axe reports it as "needs review" rather than a violation because the span has text, so the baseline passes, but the label is discarded and assistive technology reads only "1" or "5+". Remove `aria-label` and add visually hidden " results" text, or give the span a role that allows a name. |
| R4-3 | LOW (info) | `web/src/components/AppShell.tsx:113-145`; FRONTEND-AUDIT.summary.json `lighthouse[dashboard]` | RB-6 is correct as code, but its stated aim of lifting dashboard Lighthouse performance from 79 to 80 or above did not show up in measurement: the run at `c7d8d3f` records 75, which still fails INV-2. Do not record the performance failure as fixed by RB-6 until a fresh `--full` run confirms it. This is not a defect of the diff. |
| R4-4 | LOW (advisory) | 26 copy-pasted wrappers under `web/src/app/(app)/**`; `web/src/components/` | Loading-region semantics are duplicated 26 times, and no component test asserts them, so this regression class (generic element with a label) was caught only when the axe baseline happened to snapshot a loading state. A shared `LoadingRegion` primitive in `ui.tsx` with one vitest assertion would close it. Separately, the AppShell drawer has no vitest or e2e test of its own and relies on the baseline's `checkMobileNav`. |

No finding is correctness-class: none fails items 1-6, 8 or 9.

Verdict: PASS
