# Design Quality Score: round 3 (closure re-score after RB-2)

Scored commit: c230639 (`RB-2: Review Round 2 revert-and-refix ...`). The running `threatwatch-mdlc-web-1` container was rebuilt at 02:18:52, after the commit (02:17:25). Every audit row records `gitSha c2306393...` and served `buildId ddfbc61fc2bd`.

- **Archetype and system:** unchanged from score-1 and score-2: `saas`, dark-first, Audiowide / IBM Plex Sans / JetBrains Mono, 14px x 1.2 scale, 4px unit. RB-2 did not touch `globals.css` or `tokens.css`.
- **Scorer:** fresh-context subagent.
- **Closure rule (BUILD.md, Design Quality Gate):** every screenshot used here was captured after the last remediation (RB-2), so this posture does not predate it. I captured them with the pinned runner (`scripts/web-baseline.mjs --screen`, unmodified, never `--full`), in three serial batches at `--concurrency 2`:
  - Batch 1: landing, login, register, dashboard, account, integrations, integration-detail, rules, rule-new, rule-detail.
  - Batch 2: correlations, correlation-detail, alerts, alert-detail, assets, asset-detail, search, source-profile, events, intel.
  - Batch 3: map, notifications, notification-log, reports, report-detail, team, organization, audit-log, jobs.
  - All 29 screens x 5 cells (375, 375 dark, 768, 1280, 1280 dark) were recaptured between 02:20 and 02:32. No PNG in `artifacts/web-baseline/` predates 02:19.
- **Measured audit:** I kept copies of the batch summaries at `scratchpad/score3-{landing,,correlat,map,noti}.summary.json`.
  - **0 failing cells on all 29 screens (145 cells).** The axe `target-size` failure (D2-1) is gone from every app screen in light and dark, including the runner's Tab-to-skip-link step. The Correlations 375 overflow (D2-2) is gone.
  - The only `summary.failures[]` entry is `site buildFreshness` in batch 1. It compares against the local `web/.next/BUILD_ID`, not the served container, so it is infra and not a design failure. Batches 2 and 3 report `buildFreshness: current` with no failures.
- **Viewing:** I read the PNGs, cropped to the first viewport (with a second crop where the flagged item sat below the fold), at:
  - 1280: dashboard, alerts, correlation-detail, asset-detail, rules, audit-log, notification-log, report-detail, landing.
  - 768: alerts.
  - 375: correlations (light and dark), asset-detail, correlation-detail, source-profile, team.
  - Dark: dashboard@375.dark, alerts@375.dark.
  - Unchanged screens were spot-checked against score-2.

## Per-screen scores

"Gate" applies the rubric: any `summary.failures[]` entry on a screen makes it an automatic ✗. "Score" is the final posture: gate ✗ wins, otherwise the visual judgement.

| Screen | Gate | Score | Reason |
|---|---|---|---|
| landing | ✓ | ✓ | Unchanged. Text-link secondary CTA, 2 of 5 template markers, legible pipeline. |
| login | ✓ | ✓ | Unchanged and clean. |
| register | ✓ | ✓ | Unchanged and clean. |
| dashboard | ✓ | ✓ | No target-size failure. Wordmark spacing `mb-6` is restored at 1280, and the 17 nav items still fit in 820px (the last item, Jobs, is at about y 661). The lead tile and Filters disclosure at 375 are fine in dark. |
| alerts | ✓ | ⚠ | Hostnames now break only at label boundaries ("w3-/muuvdih1clm" at 768). But the 10-control filter grid still fills about 300px of the 1280 first viewport, and the 768 table still clips the Assignee column behind horizontal scroll (D2-4 remainder). The 375 card list is good in light and dark. |
| alert-detail | ✓ | ✓ | Unchanged. |
| correlations | ✓ | ✓ | At 375 the Span wraps at the arrow ("Oct 5, 2026, 1:29 AM →" / "Oct 5, 2026, 1:29 AM"), with no horizontal scroll in light or dark. The card label is now sentence-case "Source IP", matching the table. |
| correlation-detail | ✓ | ✓ | The timeline outcome "failure" is neutral `font-medium text-text`. The Pivots group headings ("Common sources", "Users") are sentence case. |
| assets | ✓ | ✓ | Unchanged. The minor D2-3 drift is shared. |
| asset-detail | ✓ | ✓ | The 375 H1 breaks at the label boundary ("e2e-muun5drjn22.corp." / "example"), with no orphan letter. The timeline "failure" is neutral at 1280 and 375. |
| search | ✓ | ✓ | Unchanged. |
| source-profile | ✓ | ⚠ | Related alerts at 375 is still a squeezed desktop table: "TW-/1" split, a 4-line title, and a wrapped "Last/event" header (D2-5, not addressed in RB-2). |
| events | ✓ | ✓ | Unchanged. |
| intel | ✓ | ✓ | Unchanged. |
| map | ✓ | ✓ | Unchanged. |
| integrations | ✓ | ✓ | Unchanged. |
| integration-detail | ✓ | ✓ | Unchanged. |
| rules | ✓ | ⚠ | The State column is still a primary-red checkbox in a read surface (D2-6, not addressed). Technique and number cells also sit about 8px above the rule-name baseline (D2-3). |
| rule-new | ✓ | ✓ | Unchanged. |
| rule-detail | ✓ | ✓ | Unchanged. |
| notifications | ✓ | ✓ | Unchanged. |
| notification-log | ✓ | ⚠ | The "Open alert" link (44px min-height) still sits about 8px below the row baseline, so those rows are taller than the "—" rows and the rhythm is uneven (D2-3). |
| reports | ✓ | ✓ | Unchanged. |
| report-detail | ✓ | ✓ | Unchanged. Lead metric, 5-up supporting row, neutral bars. |
| team | ✓ | ✓ | Unchanged. Clean at 375. |
| organization | ✓ | ✓ | Unchanged. |
| audit-log | ✓ | ⚠ | The Target cell ("user 01a10983…") still sits about 10px below Time, Action and Details, so the row baselines are ragged (D2-3). |
| jobs | ✓ | ✓ | Unchanged. |
| account | ✓ | ✓ | Unchanged. |

**Gate posture (measured): 29 ✓ / 0 ⚠ / 0 ✗.**
**Final posture: 24 ✓ / 5 ⚠ / 0 ✗ across 29 screens.**

## Part I Universal floor

| § | Item | Result |
|---|---|---|
| 1 | Display face plus one scale | ✓ |
| 2 | Committed palette, decisive accent | ⚠ The timeline "failure" residual is fixed. One residual remains: the primary-red checkboxes in the Rules State column (D2-6). |
| 3 | Intentional backgrounds | ✓ |
| 4 | 4/8px spacing, consistent radii | ✓ |
| 5 | Control states | ✓ |
| 6 | Empty, loading, error and success states | ✓ |
| 7 | Motion | ✓ |
| 8 | WCAG 2.2 AA | ✓ measured: 0 failing cells across 145. `target-size` (2.5.8) is cleared on all 26 app screens. |
| 9 | Three compositions | ⚠ The Correlations 375 overflow is fixed. Remaining: the Source-profile Related alerts table at 375 (D2-5) and the Alerts table cropped at 768 (D2-4). |
| 10 | Mobile nav | ✓ The sidebar fits at 1280x820 with `mb-6` restored. The skip link now lands over the main column at `lg`. |

## §8 anti-slop

| # | Item | Result |
|---|---|---|
| 1–5, 7–9 | | ✓ absent |
| 6 | Focus-visible / AA failures | ✓ absent (measured) |
| 10 | Cross-screen drift | ✓ The uppercase sub-labels are gone (Correlations card, CorrelationDetail Pivots). The cell-baseline drift remains as D2-3. It is a polish gap, not a ✗. |
| 11 | Template landing skeleton | ✓ 2 of 5 markers |

## Template Conformance

The tokens and font families are unchanged. Still ✓ conformant.

## D2 fix status

| ID | Status | Evidence |
|---|---|---|
| D2-1 Skip link vs wordmark (target-size) | RESOLVED | 0 `target-size` failures on all 26 app screens at 1280 in light and dark (all three batches). dashboard@1280 shows the `mb-6` wordmark gap. `AppShell.tsx` has the `lg:left-[calc(var(--sidebar-width)+0.75rem)]` skip link. |
| D2-2 Correlations 375 overflow | RESOLVED | No overflow failure on correlations@375 in light or dark. The Span wraps at the arrow. The label is sentence case. |
| D2-3 Table cell vertical rhythm | OPEN | audit-log@1280 Target about 10px low. notification-log@1280 "Open alert" about 8px low. rules@1280 Technique and numbers sit high against the rule name. |
| D2-4 Alerts filters, 768 table, hostname breaks | PARTIAL | Hostname breaks are fixed (Breakable). The filter grid still takes about 300px at 1280, and Assignee is still clipped at 768. |
| D2-5 Source-profile Related alerts at 375 | OPEN | source-profile@375: "TW-/1", a 4-line title, and a "Last/event" header. |
| D2-6 Rules State checkbox | OPEN | rules@1280: red checked checkboxes with "Enabled". |
| D2-7 Timeline failure colour, Pivots headings | RESOLVED | correlation-detail@1280 and @375, asset-detail@1280 and @375. |
| D2-8 PageHeader H1 hostname break | RESOLVED | asset-detail@375 breaks at "corp." / "example". |

Minor note, not counted: the asset-detail "Internet-sourced" series and the correlation and source-profile timeline bars use a sev-high-like orange. Those bars encode a subset or severity, not an outcome. Check them against DESIGN.md when D2-3 to D2-6 are fixed, and consider a neutral or telemetry shade unless the bar is severity-coded on purpose.

## Residual fixes (⚠, non-blocking)

- **D2-3**: `web/src/components/DataTable.tsx` (`Td`) and its callers.
  - **Where:**
    - `web/src/app/(app)/settings/audit/AuditView.tsx` (Target cell)
    - `web/src/app/(app)/notifications/log/LogView.tsx` ("Open alert")
    - `web/src/app/(app)/rules/RulesView.tsx`
  - **Fix:** give every Td one rhythm (`py-2 align-top`, or `align-middle` throughout) and remove per-cell `pt-3` and `py-0`. In-table links and CopyId get `inline-flex min-h-6 items-start` (24px, which meets WCAG 2.5.8) instead of 44px.
- **D2-4 (remainder)**: `web/src/app/(app)/alerts/AlertsView.tsx` and `web/src/components/FilterBar.tsx`.
  - **Filters:** keep Search, Status, Severity, Assignee and Rule visible. Put Source, Asset, Created, From/To and Sort behind "More filters (n)" at every breakpoint.
  - **768 table:** switch from the table to cards at `lg:` instead of `md:`, or hide Assignee and Created below `lg`.
- **D2-5**: `web/src/app/(app)/sources/[value]/SourceProfile.tsx`, Related alerts. Below `md`, render the Alerts stacked card (id and title on full-width lines, then a severity/status/risk/time key-value list). At minimum, make the id `whitespace-nowrap` and hide the Status and Risk columns below `md`.
- **D2-6**: `web/src/app/(app)/rules/RulesView.tsx`, State column. Render a status pill (Enabled uses telemetry, Disabled is neutral). Move the toggle to rule-detail, or make it a neutral switch (`accent-[var(--color-text)]`).

## Verdict

D2-1 and D2-2, the two blocking items, are resolved on fresh post-RB-2 screenshots. The measured audit has 0 failing cells across all 29 screens, and no screen fails the floor or shows a §8 item. Per the Closure rule, this round returns `0 ✗`, so the gate closes. The 5 ⚠ (alerts, source-profile, rules, notification-log, audit-log) are residual polish to log in DECISIONS.md and REPORT.md under `review_gates: auto`.

Verdict: PASS (24✓/5⚠/0✗)
