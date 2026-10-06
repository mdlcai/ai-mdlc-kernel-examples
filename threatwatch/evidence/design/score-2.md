# Design Quality Score: round 2 (re-score after RB-1)

Scored commit: a9ee654. The running `web` image was rebuilt at 01:31, after the commit.

- **Archetype and system:** unchanged from score-1: `saas`, dark-first, Audiowide / IBM Plex Sans / JetBrains Mono, 14px x 1.2 scale, 4px unit.
- **Scorer:** fresh-context subagent. I captured new screenshots with the pinned runner (`scripts/web-baseline.mjs --screen ...`, partial runs, no `--full`):
  - Run 1 covered the 23 screens score-1 flagged, at concurrency 4.
  - Run 2 re-ran 7 screens serially (events, map, search, dashboard, integration-detail, alert-detail, correlation-detail), because run 1 had page-load timeouts and LCP failures that came from contention.
  - Run 3 covered account, rule-new, notifications, organization, login and register. All 29 screens now have screenshots from a9ee654.
- **Viewing:** I read every PNG at 1280 light, 375 light and 768. I checked 1280 dark and 375 dark on alerts, map, events, account and dashboard. Full-page captures were cropped to the first viewport, with a second crop where the flagged item sat below the fold.
- **Measured audit:** I kept copies of the summaries in the scratchpad (`score2-run{1,2}.summary.json`). The live file is `artifacts/web-baseline/FRONTEND-AUDIT.partial.summary.json`, which run 3 overwrote last.
  - **Target-size failure on every app screen.** Every authenticated screen fails axe `target-size` (WCAG 2.5.8, serious) at 1280 in light and dark. The node is the sidebar wordmark `aside [aria-label="ThreatWatch home"]`. All three runs show it, including the serial one, so it is deterministic.
  - **Correlations overflows at 375.** `scrollWidth 390 > viewport 375` in light and dark.
  - **Not design failures:**
    - The LCP failures in run 1 (alert-detail, correlation-detail, integration-detail) cleared on the serial re-run.
    - `site buildFreshness` compares against the local `web/.next/BUILD_ID`, not the running container. That is infra.
- **Root cause of the target-size failure.** I confirmed it with a scratch axe probe (`scratchpad/ts-probe.mjs`). The runner presses Tab before axe runs, to test the skip link. The focused skip link (`fixed left-3 top-3`, 34px tall, y 10–44) then covers the wordmark (y 10–54), which leaves a 10px slice. RB-1 (D1-14) tightened the wordmark wrapper from `mb-6` to `mb-3`. That brought the first nav row to within 10px, so the obscured wordmark now also fails the 24px spacing exception. Without the Tab there is no violation.

## Per-screen scores

"Design" is my visual judgement. "Gate" applies the rubric: any `summary.failures[]` entry on a screen makes it an automatic ✗.

| Screen | Design | Gate | Reason |
|---|---|---|---|
| landing | ✓ | ✓ | The ghost CTA is now a text link, so only 2 of the 5 §8.11 markers remain (accent clause, muted paragraph). The 768 step grid's 5th cell spans both columns, so the orphan is gone. Pipeline chips are legible at 375. |
| login | ✓ | ✓ | Unchanged and clean. |
| register | ✓ | ✓ | Unchanged and clean. |
| dashboard | ✓ | ✗ | Lead tile (Open critical, 22) above a 3-up supporting row. Integration health shows a summary ("Silent 230 · Awaiting 23 · Healthy 0"), 8 tiles and "View all". Page height dropped from 4,764px to 1,836px. Top-source IPs are untruncated. At 375 the filters sit behind a disclosure. Gate ✗: target-size. |
| alerts | ⚠ | ✗ | IPs no longer break (`whitespace-nowrap min-w-[15ch]`). But `[overflow-wrap:anywhere]` on Asset still splits hostnames mid-label ("…corp.ex/ample" at 1280, "w3-/muuvdih1clm" at 768). The 10-control filter grid fills the 1280 first viewport (about 3 rows visible). The 768 table is cropped behind horizontal scroll (Assignee column clipped). The 375 card list is good. |
| alert-detail | ✓ | ✗ | Tabs scroll with a right-edge mask fade at 375. The MITRE value wraps right-aligned on purpose (`ml-auto`). |
| correlations | ✗ | ✗ | Table at 1280 (good). At 375 the mobile card's Span `<dd>` is `whitespace-nowrap`, so the 2-timestamp span is 390px wide and the page scrolls horizontally (measured overflow, light and dark). The card label "SOURCE IP" is uppercase while the table says "Source IP". |
| correlation-detail | ⚠ | ✗ | Timeline times no longer wrap, "#1" stays inline, and it uses the shared Stat. Outcome "failure" is still rendered in sev-high orange in the timeline (D1-6 residual). The Pivots group labels are UPPERCASE tracked. |
| assets | ✓ | ✗ | Now a DataTable at 1280 and cards at 375. Shares the minor row-alignment drift (D2-3). |
| asset-detail | ⚠ | ✗ | The H1 respects the gutter, but `[overflow-wrap:anywhere]` breaks mid-word ("corp.exampl/e" with an orphan "e") at 375. The timeline "failure" is orange (D1-6 residual). |
| search | ✓ | ✗ | No `break-all`. Alert titles and IPs render whole. |
| source-profile | ⚠ | ✗ | The chart caption now sits under the axis labels at 375 (D1-10 done). New: the "Related alerts" table is squeezed at 375 ("TW-/1", a 4-line title, and a wrapped "Last/event" header). |
| events | ✓ | ✗ | Times are neutral and "Failure" uses a neutral dot. Filters sit behind a disclosure. Stacked cards at 375 in light and dark. |
| intel | ✓ | ✗ | Full-width value cards at 375 and a proper table at 1280. Type cells sit slightly high (D2-3). |
| map | ✓ | ✗ | At 375 the map is in the first viewport behind a Filters disclosure. Good in both themes at 1280. |
| integrations | ✓ | ✗ | Sentence-case headers. Status is now a pill, matching health. |
| integration-detail | ✓ | ✗ | Shared `Stat bare` grid. The ingest URL has a Copy action and no clipping. |
| rules | ⚠ | ✗ | Now a table. The State column is still a primary-red checkbox form control in a read surface, so it reads as a form and puts accent colour on a status (score-1 note carried). |
| rule-new | ✓ | ✗ | Unchanged and good. |
| rule-detail | ✓ | ✗ | The chart uses a neutral bar. Shared Stat row. |
| notifications | ✓ | ✗ | Unchanged and good. |
| notification-log | ⚠ | ✗ | Sentence-case headers and an underlined "Open alert" link. But the link's `min-h-[44px] items-center` sits about 8px below the row baseline and makes those rows taller than the others. The rhythm is uneven. |
| reports | ✓ | ✗ | `md:pt-[26px]` is gone and the card title matches the card-title style. |
| report-detail | ✓ | ✗ | One lead metric above a 5-up supporting row. Asset meta is unclipped. The false legend is gone. Neutral alerts bar. 10 rows plus "Show all 50". |
| team | ✓ | ✗ | Uniform bordered actions. Save role appears only when dirty. No orphaned action at 375. |
| organization | ✓ | ✗ | Unchanged and good. |
| audit-log | ⚠ | ✗ | Sentence-case headers, nowrap Actor, and a CopyId on Target. The Target cell (`py-0` and a 44px CopyId) sits about 10px below its siblings, so the row baselines are ragged. |
| jobs | ✓ | ✗ | "Recent runs" is a neutral underlined disclosure. |
| account | ✓ | ✗ | Unchanged and good. |

**Design judgement: 21 ✓ / 7 ⚠ / 1 ✗.**
**Gate posture (rubric): 3 ✓ / 0 ⚠ / 26 ✗ across 29 screens.** The 26 ✗ come from one shared AppShell defect (D2-1), plus Correlations' own overflow (D2-2).

## Part I Universal floor

| § | Item | Result |
|---|---|---|
| 1 | Display face plus one scale | ✓ Card-title drift fixed (Reports, Team). |
| 2 | Committed palette, decisive accent | ⚠ Mostly fixed: Events, charts and the Jobs disclosure are neutral. Residuals: orange "failure" in the CorrelationDetail and AssetDetail timelines, and the primary-red checkboxes in the Rules State column. |
| 3 | Intentional backgrounds | ✓ |
| 4 | 4/8px spacing, consistent radii | ✓ `pt-[26px]` removed, and modals moved to `rounded-lg`. |
| 5 | Control states | ✓ |
| 6 | Empty, loading, error and success states | ✓ |
| 7 | Motion | ✓ |
| 8 | WCAG 2.2 AA | **✗ measured:** axe `target-size` (2.5.8) fails on all 26 app screens at 1280 in light and dark (D2-1). This regressed from round 1's 0 failing cells. |
| 9 | Three compositions | ⚠ Intel, Events, Alerts and Integrations now have mobile compositions. Remaining: the Correlations 375 overflow (D2-2), the squeezed Source-profile Related alerts table at 375 (D2-5), and the Alerts table cropped at 768 (D2-4). |
| 10 | Mobile nav | ✓ The sidebar now fits all 17 items at 1280x820 (rows render about 28px). |

## §8 anti-slop

| # | Item | Result |
|---|---|---|
| 1–3, 5–9 | | ✓ absent |
| 4 | Off-unit spacing / mixed radii | ✓ resolved |
| 6 | Focus-visible / AA failures | ✗ present (target-size, D2-1). It is already counted under Part I §8. |
| 10 | Cross-screen drift | ✓ largely resolved: one Th style, one Stat primitive (`components/Stat.tsx`, with `lead` and `bare`) and one FilterBar. Residual (⚠, not ✗): uppercase sub-labels in the CorrelationDetail Pivots and on the Correlations mobile card, and the cell-baseline drift (D2-3). |
| 11 | Template landing skeleton | ✓ 2 of 5 markers |

## Template Conformance

The tokens and font families did not change in RB-1 (`globals.css` and `tokens.css` are untouched). Still ✓ conformant.

## D1 fix status

| ID | Status | Evidence |
|---|---|---|
| D1-1 Dashboard hierarchy and health wall | RESOLVED | dashboard@1280: lead tile, health summary, 8 of 253 plus "View all", page height 1,836px. dashboard@375: Filters disclosure. |
| D1-2 `break-all` | PARTIAL | IPs are nowrap everywhere (alerts, search, intel, events). Hostnames still split mid-label through `[overflow-wrap:anywhere]`: alerts@1280 Asset ("corp.ex/ample"), alerts@768 ("w3-/muuv…"), asset-detail@375 H1 ("exampl/e"). |
| D1-3 DataTable / Stat / FilterBar primitives | RESOLVED | integrations, notification-log and audit-log use sentence-case Th. Stat is shared across dashboard, reports, rule-detail, asset-detail, correlation-detail and integration-detail. |
| D1-4 Intel and Events mobile compositions | RESOLVED | intel@375 and events@375 (light and dark) are stacked cards with full-width values. |
| D1-5 ReportDetail | RESOLVED | report-detail@1280: lead metric, unclipped meta, no false legend, "1 alert", neutral bar, "Show all 50". |
| D1-6 Accent vs critical | PARTIAL | Events, rule-detail and report charts, and the Jobs disclosure are fixed. The CorrelationDetail and AssetDetail timeline "failure" is still sev-high orange. |
| D1-7 Log and audit cells | PARTIAL | The link affordance, nowrap Actor and CopyId are in place. "Open alert" and the Target cell still sit off the row baseline. |
| D1-8 Mobile filters | RESOLVED | Alerts, Dashboard, Map, Assets, Integrations and Correlations at 375 all use a Filters disclosure. |
| D1-9 Timeline time column | RESOLVED | correlation-detail@1280 and asset-detail@1280 show times on one line. |
| D1-10 H1, chart caption, ingest block | PARTIAL | The caption and ingest block are fixed. The H1 stays inside the gutter but breaks mid-word (see D1-2). |
| D1-11 AlertDetail tabs and MITRE | RESOLVED | Mask fade on the tab strip. MITRE is right-aligned by design. |
| D1-12 Landing | RESOLVED | Text-link secondary CTA, 768 grid span fixed, chips legible. |
| D1-13 Team and Reports | RESOLVED | team@1280 and team@375, reports@1280. |
| D1-14 Compact sidebar | RESOLVED, but caused D2-1 | All items fit at 1280x820, but the `mb-6` to `mb-3` change produced the target-size regression. |
| D1-15 Cleanup | RESOLVED (one residual) | Card titles, `rounded-lg` modals and Integrations status pills are done, and Correlations, Assets and Rules are DataTables. The Rules State checkbox remains (D2-6). |

## New issues

- **D2-1 (blocking, all 26 app screens, 1280, light and dark)** in `web/src/components/AppShell.tsx` (the skip link at about line 300 and the aside wordmark wrapper at about line 304).
  - **Problem:** the focused skip link covers the sidebar wordmark, and the first nav row is now only 10px away.
  - **Fix (preferred):** keep the compact nav and move the skip link off the sidebar at `lg`. Add `lg:left-[calc(var(--sidebar-width)+theme(spacing.3))]` to the skip link, or a token-based `lg:left-[calc(var(--sidebar-width)+0.75rem)]`, so it lands over the top bar or main column.
  - **Fix (alternative):** restore `mb-6` on the wordmark wrapper and drop the group margin to `mb-1`, so the nav still fits.
  - **Check:** run `web:baseline --screen dashboard` after the change.
- **D2-2 (blocking, correlations, 375)** in `web/src/app/(app)/correlations/CorrelationsView.tsx:177`.
  - **Problem:** the mobile Span value can't wrap, so the page overflows to 390px.
  - **Fix:** remove `whitespace-nowrap` from the `<dd>`. Wrap each timestamp in its own `<span className="whitespace-nowrap">` with the arrow between them, so the span wraps at the arrow. Add `min-w-0` to the card grid. While there, make the card's entity label sentence case to match the table (drop `uppercase tracking-wide`, line 158).
- **D2-3 (⚠, cross-screen: integrations, assets, intel, events, correlations, audit-log, notification-log; 1280 and 768)** in `web/src/components/DataTable.tsx` and the callers.
  - **Problem:** cells use mixed `py-0` (44px link or CopyId) and `pt-3` offsets, so the first column, Target or "Open alert" sits 8–10px off the row baseline.
  - **Fix:**
    - Give `Td` one vertical rhythm: `py-2 align-top` with no per-cell `pt-3` or `py-0`.
    - Inside tables, give links and CopyId `inline-flex min-h-6 items-start`. WCAG 2.5.8 AA needs 24px, and the 44px tier applies to primary nav and buttons per the runner.
    - Alternatively, switch these tables to `align-middle` throughout.
- **D2-4 (⚠, alerts, 1280 and 768)** in `web/src/app/(app)/alerts/AlertsView.tsx` and `web/src/components/FilterBar.tsx`.
  - **Problem:** the filter grid takes about 300px of the first viewport at 1280, and the 768 table clips the Assignee column.
  - **Fix:**
    - Keep Search, Status, Severity, Assignee and Rule visible. Put Source, Asset, Created, From/To and Sort behind a "More filters (n)" disclosure at every breakpoint (FilterBar already has the disclosure).
    - Below `lg`, use the card list (`lg:` instead of `md:` for the table switch), or hide Assignee and Created at `md`.
    - Drop `[overflow-wrap:anywhere]` from the Asset cell (line 437). Use `whitespace-nowrap max-w-[24ch] truncate` with `title` and the full hostname, or insert `<wbr/>` after `.` and `-` with a small `breakableHost()` helper so breaks only fall at label boundaries.
- **D2-5 (⚠, source-profile, 375)** in `web/src/app/(app)/sources/[value]/SourceProfile.tsx`, Related alerts.
  - **Problem:** a desktop table is squeezed into a phone.
  - **Fix:** below `md`, render the same stacked card used on Alerts (id and title on full-width lines, then a severity/status/risk/time key-value list), or hide the Status and Risk columns and make the id `whitespace-nowrap`.
- **D2-6 (⚠, rules, 1280)** in `web/src/app/(app)/rules/RulesView.tsx`, State column.
  - **Problem:** the primary-red checkbox reads as a form and puts accent colour on a status.
  - **Fix:** render a status pill (Enabled uses telemetry, Disabled is neutral) and move the toggle into the row's detail page or a neutral switch with `accent-[var(--color-text)]`.
- **D2-7 (⚠, correlation-detail and asset-detail timelines)** in `web/src/app/(app)/correlations/[id]/CorrelationDetail.tsx` and `web/src/app/(app)/assets/[id]/AssetDetail.tsx`.
  - **Problem:** this is the D1-6 residual: the event outcome "failure" uses sev-high orange.
  - **Fix:** use the same neutral dot plus `text-text` treatment as EventsView. Also make the Pivots group headings (CorrelationDetail.tsx:302) sentence case `text-xs font-medium text-text-secondary`, without `uppercase tracking-wide`.
- **D2-8 (⚠, asset-detail H1, 375)** in the `PageHeader` title in `web/src/components/ui.tsx`.
  - **Problem:** `[overflow-wrap:anywhere]` breaks the hostname mid-word.
  - **Fix:** pass the hostname through the D2-4 `breakableHost()` (`<wbr/>` after `.` and `-`) and keep `overflow-wrap:anywhere` only as a last-resort fallback.

## Verdict

D2-1 is one AppShell line. It accounts for 25 of the 26 gate ✗, and fixing it together with D2-2 clears all of them. On visual judgement alone the posture would be 21 ✓ / 7 ⚠ / 1 ✗. After fixing, rebuild the `web` image, run `npm run web:baseline -- --screen dashboard,correlations,alerts,audit-log,notification-log`, then re-score.

Verdict: FAIL (3✓/0⚠/26✗)
