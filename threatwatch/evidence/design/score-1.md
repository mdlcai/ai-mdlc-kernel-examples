# Design Quality Score: round 1 (pre-score)

Scored commit: d700300

- **Archetype:** `saas` (RESEARCH.md §Design Language, `Archetype: saas`). Brand voice: authoritative, technical, calm. Art direction is dark-first.
- **Design system:** tokens derived from DESIGN-TEMPLATE.html (ADR-002), with extra derived tokens from ADR-003. Faces: Audiowide (display and wordmark), IBM Plex Sans (body), JetBrains Mono (data). Type scale is 14px x 1.2. Spacing unit is 4px. Radii are 4/8/12px. Default theme is dark, with a light theme behind a runtime toggle.
- **Scorer:** fresh-context subagent. I opened every screen at 1280 light, 375 light and 1280 dark. I also opened 375 dark and 768 where they mattered. Full-page captures were cropped to the first viewport for detail views.
- **Evidence freshness:** the screenshots and `FRONTEND-AUDIT.nolh/partial.summary.json` were built at 8d09fef. The 8d09fef..d700300 diff under `web/` is strict-lint, Prettier and hook refactors. `git diff -w` shows no className, style or token-value changes; hex values were only lower-cased. The visuals therefore count as current for d700300.
- **Measured audit:** the nolh run covered 29 screens (145 cells) and the partial run covered landing and dashboard. Every screen shows `failing: 0` for a11y, overflow, targets and contrast. There are no per-screen `summary.failures[]`. The only failure is site-level: no HSTS header. That is infra, not design. Site-level checks and Lighthouse are deferred to the final `--full` run.

## Per-screen scores

| Screen | Score | Reason |
|---|---|---|
| landing | ⚠ | Shows 3 of the 5 §8.11 markers: accent-coloured final clause, muted paragraph, filled+ghost CTA pair. That is under the ✗ threshold, but it is the template shape. At 768, the 5-item step grid leaves an empty tinted orphan cell. At 375, the pipeline-chip text is very small. |
| login | ✓ | Centred auth card on the tw-grid backdrop, with labelled fields. Clean in both themes and at every viewport. |
| register | ✓ | Same auth composition. The password hint sits under its field. |
| dashboard | ✗ | The Integration health tile dumps every integration with no cap. It is a wall of hundreds of equal tiles: page height is 4,764px at 1280 and 14,708px at 375. That breaks saas "lead metric -> supporting -> detail, never an equal tile grid". The KPI row is 4 equal tiles with no lead metric. Top-source IPs are truncated ("198.18.118.…") at 1280. At 375, three stacked filters fill the first viewport. Drift: the KPI tile is a one-off variant (§8.10). |
| alerts | ✗ | `break-all` on the Source and Asset cells splits IPs mid-octet and hostnames mid-word ("198.18.253./54", "corp.ex/ample") at 1280. At 768 it is worse ("198.1/8.253./54"). This fails engineered tables on the primary triage screen. Rows are about 90px because the title is repeated as a subtitle. At 375, 8 stacked filters push the first alert below the fold. |
| alert-detail | ⚠ | Good two-column triage layout with designed empty states. At 375, the tab strip clips "Notificatio…" with no scroll affordance. The MITRE key-value pair stacks while its siblings align right. |
| correlations | ⚠ | Readable card list, but lower density than the saas/SOC brief wants. No table, sort or bulk action. |
| correlation-detail | ⚠ | At 1280, timeline timestamps wrap ("8:18/PM"). An orphan "#1" sits on its own line at 375. It uses a local Stat variant (sentence-case label). |
| assets | ⚠ | Card list at 1280 where a table belongs. At 375, the inline filter selects have ragged widths. |
| asset-detail | ⚠ | At 375, the H1 hostname runs to the viewport edge and ignores the 16px gutter. Activity timestamps wrap ("9:38/PM"). It uses a local Stat and KV variant. |
| search | ⚠ | Grouped results are good. The alert title is mono with `break-all`, so it splits mid-word ("authentica/tion", "20/3.0.113.7"). |
| source-profile | ⚠ | Solid profile layout. At 375, the activity-chart axis labels collide ("2026-09-failures shown in the darker 2026-10-"). Stat values are mono here but sans elsewhere. |
| events | ⚠ | Dense, engineered table at 1280. Every timestamp is rendered in primary red, which is also the critical-severity colour. "Failure" uses sev-high orange. At 375 it is the desktop table cropped behind horizontal scroll, not a designed composition. |
| intel | ✗ | At 375 the desktop table is squeezed. `break-all` chops indicator values into roughly 10-character vertical stacks, which makes them unreadable. There is no mobile composition, which fails the Part I three-compositions floor. Filter labels are bold here but regular on Alerts. |
| map | ⚠ | The 1280 composition (map plus country rail) is good in both themes. At 375, the filter stack and controls push the map below the first viewport. This is the shared mobile-filter issue. |
| integrations | ✗ | Cross-screen drift (§8.10): UPPERCASE tracked table headers, where Alerts, Events and Intel use sentence case. Status is shown as uppercase coloured text while health uses pills. |
| integration-detail | ⚠ | Good page with a designed "Silent" error banner. Its stat grid is a third local Metric variant (joined cells). The ingest URL and curl block clip at the right edge. |
| rules | ⚠ | A card list instead of a table makes it low density. The enable checkbox reads as a form control, not a status. |
| rule-new | ✓ | Well-structured sectioned form with hints under fields and a segmented match control. Good in both themes. |
| rule-detail | ⚠ | The performance chart bar is a large primary/critical-red block for a neutral alert count. Colour should mean status or severity only. |
| notifications | ✓ | Designed empty states with a clear CTA and the tw-grid treatment. Good in both themes. |
| notification-log | ✗ | Drift (§8.10): UPPERCASE table headers. Cells are vertically misaligned: timestamps are top-aligned and "Open alert" is mid-aligned. "Open alert" has no link affordance. |
| reports | ⚠ | The "Generate a report" card title uses about 20px where other card titles use 14px semibold. The form uses an off-unit `md:pt-[26px]` alignment hack. |
| report-detail | ✗ | 8 equal KPI tiles with no lead metric (saas avoid). Targeted-assets meta is clipped to "27 ev · 2 al". The legend says "dashed: previous", but no dashed series is drawn. "1 alerts" pluralisation. Neutral counts use the critical-red bar. About 50 critical-activity rows are unpaginated. |
| team | ⚠ | Each row has 4 actions in mixed styles: bordered selects and buttons next to a borderless "Reset password", which is orphaned at 375. "Save role" is always visible. Uses `md:pt-[26px]`. |
| organization | ✓ | Compact settings form. Clean in both themes. |
| audit-log | ✗ | Drift (§8.10): UPPERCASE table headers. The Actor column wraps ("E2E/Analyst"). Target IDs are truncated with no way to reveal them. |
| jobs | ⚠ | Clear job cards. The "Recent runs (5)" disclosure is in critical red, which collides accent with severity. |
| account | ✓ | Two-card profile and password layout with hints under fields. Fine at 375 dark. |

("/" in quoted strings marks where the rendered text line-breaks.)

**Posture: 6 ✓ / 16 ⚠ / 7 ✗ across 29 screens.**

## Part I Universal floor

| § | Item | Result |
|---|---|---|
| 1 | Characterful display face plus one modular scale | ✓ Audiowide, IBM Plex Sans and JetBrains Mono, all self-hosted. No Inter, Roboto or Arial. One 1.2 scale in `tokens.css`. Minor drift: a ~20px h2 inside some cards (Reports, Team). |
| 2 | Committed palette, decisive accent | ⚠ Committed and decisive (#FF4D6A / #D61F45). But ADR-003 sets `sev-critical == primary`, so the accent and critical severity collide: Events timestamps, the Rule-detail chart, the Jobs disclosure, and the Danger buttons that look like the accent. |
| 3 | Intentional backgrounds | ✓ The template's 44px tw-grid sits behind landing, auth and empty states. App surfaces stay flat with 1px borders. |
| 4 | 4/8px spacing, consistent radii | ⚠ The 4px scale is followed almost everywhere. Two off-unit `md:pt-[26px]` (ReportsView.tsx:206, TeamView.tsx:390). Modals use `rounded-xl` (the Tailwind default) instead of the `--radius-lg` token. No hard-coded hex values in tsx. |
| 5 | Designed control states | ✓ The Button primitive has hover, `focus-visible` (global ring), disabled (opacity plus not-allowed) and busy states. Inputs show a focus border and an error border. |
| 6 | Designed empty, loading, error and success states, with field errors attached | ✓ EmptyState, Skeleton and Alert primitives. Empty states are designed (Notifications, Target asset, Threat intel). The silent-integration error banner is designed. Fields carry `aria-invalid` with the hint or error under the control. |
| 7 | Motion 120-320ms eased, reduced motion honoured | ✓ `--dur-fast` 120 / `--dur-panel` 200 / `--dur-stagger` 320ms with `--ease`. Global `prefers-reduced-motion` kill-switch. The 1.8s twpulse is the template-mandated live indicator, which is purposeful. |
| 8 | WCAG 2.2 AA in light and dark | ✓ Measured: 0 failing a11y, contrast or target cells across 29 screens x 5 cells. Lighthouse is deferred to `--full`. |
| 9 | Three compositions at 375/768/1280 | ✗ Intel at 375, Events at 375 and Alerts at 768 are squeezed or cropped desktop tables, not compositions. The mobile filter stacks on Alerts, Dashboard and Map bury content. |
| 10 | Mobile nav | ✓ A 44px toggle with `aria-expanded` and `aria-controls` opens a focus-trapped drawer. Escape closes it and focus returns to the toggle. A skip link comes first. Note: at 1280x820 the fixed sidebar puts Team, Organization, Audit log and Jobs below the fold, because every row is 44px. |

## §8 anti-slop list

| # | Item | Result |
|---|---|---|
| 1 | Default font | ✓ absent |
| 2 | Purple-blue gradient / template look | ✓ absent (ember-red on near-black, technical grid) |
| 3 | Stock component-library theme | ✓ absent (custom primitives) |
| 4 | Off-unit spacing / mixed radii | ⚠ minor: 2x `pt-[26px]`, plus `rounded-xl` modals outside the token scale. Not judged ✗. |
| 5 | Missing or undesigned states | ✓ absent (empty, loading, error and success are designed) |
| 6 | No focus-visible / AA failures | ✓ absent (measured) |
| 7 | Purposeless motion | ✓ absent |
| 8 | Centred single form as the app | ✓ absent (auth screens only) |
| 9 | Lorem / TODO / placeholder copy | ✓ absent (grep finds none; the `muu…` and `E2E` strings are e2e seed data, not product copy) |
| 10 | Cross-screen drift | **✗ present.** There are three table-header styles: UPPERCASE on Integrations, Notification log and Audit log, sentence case elsewhere. Stat tiles have five local variants: Dashboard KPI, `AssetDetail.Stat`, `CorrelationDetail.Stat`, `IntegrationDetail.Metric` and `rules/_parts.Metric`, plus the mono values on Source profile and ReportDetail. Filter bars have three layouts: labelled grid, inline label-left, and bold labels. Card titles use two sizes. |
| 11 | Template landing skeleton | ⚠ 3 of 5 markers: accent final clause, muted paragraph, filled+ghost CTA. No eyebrow pill and no 3-stat / 3-icon-card row. Below the 4-of-5 ✗ threshold. |

## Template Conformance

- **Token diff:** N/A in strict form. The template has no `:root` block (ADR-002), so the build derived its tokens from the template's own colours. I checked those derived values against the template HTML and they match `tokens.css` exactly: `#0E090B`, `#F4ECEE`, `#FF4D6A`, `#FF7A90`, `#8C787E`, `#21141A`, `rgba(244,236,238,0.08)`, `rgba(255,77,106,0.06)`, and a 44px grid. The only difference is lower-case hex from Prettier. ADR-003 tokens sit alongside the template values; none replace them. **Conformant.**
- **Typography:** the template's families are Audiowide, IBM Plex Sans and JetBrains Mono. The build loads and applies exactly those. **Conformant.**
- **Landing scaffold:** the template export has no component markup (three `<dc-import>` stubs), so the scaffold cannot be ported. This is logged in ADR-002 and DOGFOOD.md. **N/A.**
- **Result:** ✓ conformant (no token or typography ✗).

## Prioritized fix list

Group by primitive first. D1-2 and D1-3 each clear several screens.

- **D1-1 · `features/dashboard/DashboardView.tsx` (Integration health, KPI row).** Replace the unbounded tile wall with a health summary (Silent N · Awaiting N · Healthy N). List at most 8 unhealthy integrations, then a "View all" link to `/integrations?health=silent`. Make Open Critical a larger lead tile, with the other three KPIs as a supporting row. Stop truncating IPs: mono, `whitespace-nowrap`, wider column. *Why:* the saas composed hierarchy, and it clears the dashboard ✗.
- **D1-2 · the `break-all` primitive in `alerts/AlertsView.tsx:448-449,486`, Search results, `intel/IntelView.tsx:269,320`, `AlertDetail` (345,374,409,447,861,872), `AssetsView:154`, `AssetDetail`, `CorrelationsView:144`, `CorrelationDetail`, `EventsView:545`.** Use `whitespace-nowrap` with `min-w-[15ch]` for IPs. Use `[overflow-wrap:anywhere]` or `break-words` only for hostnames, emails, URLs and hashes, and only when they sit on their own full-width line. Set Source and Asset column minimum widths. Drop the repeated rule-name subtitle when it matches the title prefix, and tighten rows to `py-2`. *Why:* clears the Alerts ✗ and the Search ⚠, and fixes engineered tables.
- **D1-3 · `components/ui.tsx`: add `Th`/`DataTable`, `Stat` (with a `lead` prop) and `FilterBar`.** `Th` uses sentence case, `text-xs font-medium text-text-secondary`. `Stat` has one label style, a sans value with tabular numerals, an optional hint and an optional tone. `FilterBar` puts labels above the controls. Replace the local versions in Integrations, Notification log, Audit log, AssetDetail, CorrelationDetail, IntegrationDetail, `rules/_parts`, ReportDetail, SourceProfile, Dashboard, Assets, Rules, Correlations and Intel. *Why:* §8.10 drift. Clears the Integrations, Notification log and Audit log ✗ and several ⚠.
- **D1-4 · Intel and Events below `md`.** Render stacked cards, like Alerts already does: value on its own full-width line, then type, threat, confidence and scope as a key-value list. *Why:* the three-compositions floor. Clears the Intel ✗ and the Events ⚠.
- **D1-5 · `reports/[id]/ReportDetail.tsx`.** Use one lead metric (Critical/high alerts with severity split) above a supporting row, instead of 8 equal tiles. Unclip the targeted-asset meta ("27 events · 2 alerts", allowed to wrap). Remove the "dashed: previous" legend unless a previous series is drawn. Show 10 critical-activity rows plus "Show all". Fix "1 alerts" -> "1 alert" pluralisation in Top sources. Use a neutral bar colour for counts. *Why:* clears the report-detail ✗.
- **D1-6 · Accent vs critical collision.** In Events, render the time link in `text-text` with an underline on hover; give Outcome "Failure" a neutral or status style, not sev-high. In RuleDetail and the report "Alerts per day" chart, use a neutral or telemetry bar colour. Make the Jobs "Recent runs" disclosure `text-text` with an underline. Keep primary for actions only. Do not re-theme the tokens, because ADR-002/003 fix them; this is a usage fix. *Why:* saas restrained palette, and colour only for status or severity.
- **D1-7 · `notifications/log/LogView.tsx`, `settings/audit`.** Apply `align-top` to all cells. Style "Open alert" as a link (underline or primary text). Actor gets `whitespace-nowrap`. Target IDs get a `title` with the full value and copy-on-click. *Why:* finishes the Notification log and Audit log after D1-3.
- **D1-8 · Mobile filters (Alerts, Dashboard, Map, Assets, Audit, Notification log).** Below `md`, collapse into a "Filters (n active)" disclosure, as Events already does, with full-width selects. *Why:* the 375 composition. Content should be in the first viewport.
- **D1-9 · Timeline time column (CorrelationDetail, AssetDetail).** Apply `whitespace-nowrap` with a fixed `19ch` width. Keep the "#n" alert ref inline. *Why:* stops the wrapped "8:18/PM".
- **D1-10 · `PageHeader` title, SourceProfile chart, IntegrationDetail ingest block.** Add `[overflow-wrap:anywhere]` to the H1 so long hostnames respect the gutter. Below `sm`, move the chart caption under the axis labels so they don't collide. Make the ingest URL and curl block `overflow-x-auto` with a copy button, so they are not clipped.
- **D1-11 · `AlertDetail` tabs and key-value pairs.** Add an edge-fade scroll affordance or shorter labels at 375. Align MITRE right like its siblings.
- **D1-12 · `app/page.tsx` (landing).** Break the template skeleton further: drop the accent colour on "Act on the right one." or demote the ghost CTA to a text link. At `md`, fix the orphan 6th cell: make the 5th step span 2 columns or use a 3+2 layout. Raise the pipeline-chip text to at least `--text-xs` at 375.
- **D1-13 · `settings/team/TeamView.tsx`.** Show "Save role" only when the select is dirty. Move Disable and Reset password into one row overflow menu so every action shares a style. Replace `md:pt-[26px]` with `items-end`. Apply the same `items-end` fix in `ReportsView.tsx:206`.
- **D1-14 · `components/AppShell.tsx` nav.** At `lg` (pointer), use compact rows (`min-h-9`) so all 17 items fit at 1280x800, keeping 44px rows in the drawer. Settings items are below the fold today.
- **D1-15 · Cleanup.** Bring card h2 sizes into line (Reports and Team "Generate…" / "Add a member" to the card-title style). Change modals (`alerts/_parts.tsx:289`, `integrations/_parts.tsx:202`) from `rounded-xl` to the `rounded-lg` token. Show Integrations status as a pill, matching health. Correlations, Assets and Rules: move to the D1-3 `DataTable` at `lg`.

## Verdict

**FAIL.** 7 screens are ✗ and the gate blocks on any ✗: dashboard, alerts, intel, integrations, notification-log, report-detail, audit-log. Run the auto-remediate pass on D1-1 to D1-8 first; those clear all 7 ✗. Then fix the mechanical ⚠ items (D1-9 to D1-15), rebuild, run `npm run web:baseline -- --screen <affected>`, and re-score. This is a pre-score: the gate closes only at the wide re-score against the definitive run's screenshots.
