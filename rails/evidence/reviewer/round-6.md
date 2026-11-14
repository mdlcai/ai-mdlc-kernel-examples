Commit reviewed: 2a05154
Previous round reviewed: d4c6ec6
Mechanism: Task tool general-purpose subagent
Verdict: PASS

Inputs: `git show 37a9af9` (RB-19), `git show 2a05154` (RB-20), artifacts/reviewer/round-5.md (format and M1/M3 reference), web/app/globals.css, web/components/console-shell.tsx, web/components/ui.tsx, web/components/console/live-feed.tsx, web/components/console/charts/volume-chart.tsx, web/app/(console)/payments/page.tsx, web/app/(console)/payments/[id]/page.tsx, web/app/(console)/disputes/page.tsx, web/app/(console)/disputes/[id]/page.tsx, web/app/(console)/refunds/page.tsx, web/app/(console)/ledger/page.tsx, web/app/(console)/ledger/entries/[id]/page.tsx, web/app/(console)/governance/audit-trail.tsx, web/app/(console)/ops/{page,dlq,merchants,runs}.tsx, web/app/(console)/settings/processors/page.tsx, web/app/(console)/developers/webhooks/page.tsx, web/app/(marketing)/marketing.css, web/app/(marketing)/page.tsx, web/app/layout.tsx (mono font), DESIGN-TEMPLATE.html (token block), api/app/models/concerns/public_id.rb (id length), web/e2e/*.spec.ts.
Focus items: GD-0050 (RB-19), GD-0051 (RB-20, payments screens: web/app/(console)/payments/*, disputes/[id]).

# Reviewer Gate: focus check (GD-0050, GD-0051)

The review was read-only. No docker, e2e or web baseline was run, because a verification run was using the stack, so no rendering was checked visually. Pixel widths below are estimated from JetBrains Mono's 0.6em advance and the layout classes. I ran `npm --prefix web run typecheck` (clean), `npm --prefix web run lint` (clean, max-warnings 0) and `npm run test:web` (63/63 passed).

## Checklist

| # | Item | Result | Rationale |
|---|---|---|---|
| 1 | Payments and dispute screens are presentation-only. Money still goes through formatMoney | PASS | `payments/[id]/page.tsx:139` and `disputes/[id]/page.tsx:109` change only the `dl` grid template. The fields, values and conditions are unchanged, and Amount, Captured, Capturable, Fee and Refunded still call `formatMoney(*_minor, currency)`. `Item` drops `break-all` (`:301` and `:181`). `dd` keeps `overflow-wrap: anywhere` (`globals.css:108-112`), so `processor_ref` still wraps when it is too long. `payments/page.tsx:105` passes only an alignment class to `LiveStatus`, and `canWrite` still gates "New payment". `payments/page.tsx:225-231` swaps `max-w-[16rem] + truncate` for `cell-clip` with the same 16rem default and the same text and title. ids, hrefs, actions and auth gates are untouched. The RB-19 `a.mono` rule does affect the id links on these screens, but only their layout (see M1). |
| 2 | Sidebar active-item scroll and the `(width >= 48rem)` gate | PASS (advisory A1) | The effect (`console-shell.tsx:62-70`) writes only `box.scrollTop`, and `box` is the nav's parent: the sticky `overflow-y-auto` aside (`:154`) or the dialog. It never calls `scrollIntoView` or `window.scroll`, so the page itself never scrolls. The dependencies `[pathname, count]` are not changed by the effect, so it cannot loop. It reads two rects once per navigation, and those reads happen before the write, so it does not thrash layout. Below md the Sidebar is not rendered (`desktop` false), so the effect never runs. The ref guards return early when there is no nav. `(width >= 48rem)` is the exact query Tailwind v4 generates for `md`, so round-5 M1 is closed. The range syntax needs Safari 16.4 or later, which is Tailwind v4's own floor. |
| 3 | Accessibility | PASS (advisories A2, A3) | sr-only table: the wrapper `div.sr-only` (`volume-chart.tsx:65`) hides the table visually but keeps it in the accessibility tree, and the caption and `th scope` are intact. Governance: the ResizeObserver is created per `page` and disconnected in cleanup (`audit-trail.tsx:49-57`). It guards an undefined ResizeObserver. The hint line has a reserved height, so the table does not shift when it appears (`:162-164`). The region keeps `tabIndex=0`, `role=region` and its label (`:165`). Ops: the links target `#ops-merchants-h`, `#ops-dlq-h` and `#ops-runs-h`, which are the existing h2 ids (`merchants.tsx`, `dlq.tsx`, `runs.tsx`). They are `.btn` (`min-height: 44px`, `globals.css:160`), and `scroll-mt-20` (5rem) clears the 4rem sticky header. `title` attributes sit on non-interactive spans, or on a Link whose name stays its text (`webhooks/page.tsx:102`), so no accessible name is replaced. No focus styles changed. The tfoot keeps `<th scope="row">Total</th>` (`ledger/entries/[id]/page.tsx:138`), and card mode only sets `display: block` on it, as tbody and tr already had. |
| 4 | CSS regressions | FAIL (MINOR M1) | `.table td.cell-clip` has exactly three users: payments Description, webhooks URL and governance Target. The card overrides are repeated in both the `@media (max-width: 640px)` and `@container table-panel` blocks (`globals.css:326-337`, `:383-394`), after the base rule (`:263-270`), so they win at equal specificity. `.table td.cell-clip > *` (0,2,1) beats `a.mono` (0,1,1), so card-mode URLs wrap. The tfoot rules affect only the journal-lines table, the only `tfoot` in the app, and they correctly un-stick a footer `th` that had inherited `position: sticky; top: 0`. `accent-color` (`:229`) and `a.mono` (`:222`) live in `@layer components`, so Tailwind utilities still override them. The marketing pages have no `a.mono`, no `.btn.mono` and no radios or checkboxes. Landing: `.mk .s` has no `content-visibility` left (`marketing.css:147-150`), and `> :last-child:nth-child(3n + 1)` (`:217`) matches only a lone last card in the `#network` list (7 = 3·2+1), so round-5 M3 is closed. Regression: `a.mono { white-space: nowrap }` (M1). |
| 5 | Tokens (ADR-044) and no hard-coded colours | PASS | No custom property is added or changed in either diff. The new rules use `--cyan` and `--surf2` (template tokens, `DESIGN-TEMPLATE.html:19-20`) and `--violet-fill` (an existing project token, `globals.css:14,50`). No hex or rgba value was added. DESIGN-TEMPLATE.html is untouched. |
| 6 | e2e labels, roles and test ids | PASS | `w10-ops` locates its sections with `getByRole('region', { name: 'Dead-letter queue' / 'Scheduled runs' })`. The new nav links have the link role, not the region role, and the merchants row uses `tbody tr`, so neither is ambiguous. `w7-developer:35,39` finds the URL link by its text, which is still its name. `w9-governance:38` uses table name "Audit events", which is unchanged. `w3-failover` uses the "Sandbox priority" label, which is unchanged. `w6:39` counts `/^Journal je_/` links, which are unchanged. `w2:16` checks the status text "Live", which is unchanged. |
| 7 | CHANGELOG accuracy | PASS (advisory A4) | Every RB-19 and RB-20 bullet matches a change in the diff. A few omissions and wording points are listed in A4. |
| 8 | Test coverage (advisory) | ADVISORY (A5) | Nothing unit-tests the sidebar scroll, the overflow hint, the ops links or the cell-clip and tfoot CSS. e2e coverage of the touched screens is unchanged and was not re-run. |

## Findings

No BLOCKER or MAJOR.

### M1 (MINOR): `a.mono { white-space: nowrap }` defeats every wrapping fallback for linked ids

RB-19 added `white-space: nowrap` to every `a.mono` (`globals.css:222-225`). Most of these links are 27-character public ids (`<prefix>_` plus 24 base58 characters, `public_id.rb:1`), which take about 227px at 14px. Some show "Journal " plus an id, which takes about 294px. Under `nowrap`, `overflow-wrap: anywhere` in card-mode cells (`globals.css:308,365`) and in `dd` (`:111`), and `break-all` at `ledger/entries/[id]/page.tsx:92`, can no longer break the line. Estimated effects, not visually confirmed:

- **Card mode at 375px:** the value column is about 217px (343 minus borders, 24 padding, 88 label and 12 gap). The Payment link in refunds (`refunds/page.tsx:139`) and the Entry link in ledger (`ledger/page.tsx:276`) overflow by about 10px, so the table panel scrolls sideways.
- **Payment detail aside at lg:** "Journal je_…" (`payments/[id]/page.tsx:272`, about 294px) sits in a panel content box of about 190px at 1024px and about 275px at 1280px. It runs past the panel border. At 1024px it also widens the page.
- **Dispute detail ledger list:** the same happens (`disputes/[id]/page.tsx:165`, text-xs, about 252px against about 190px at 1024px).

Before RB-19 the id alone could already overflow the 1024px aside, but the line could still break at the space, so this change makes it worse. Fix: drop `nowrap` from `a.mono` and keep the cyan colour. Or scope `nowrap` to table mode (for example `.table:not(…) a.mono`, or outside the card media and container blocks). Also give the aside journal links `overflow-wrap: anywhere`.

### A1 (ADVISORY): the active-item scroll does nothing in the drawer

In the drawer, the Sidebar's effect runs before the Drawer's `showModal()` effect, because child effects run first. At that point the dialog is not open, so every rect is 0 and nothing scrolls. This is harmless and the CHANGELOG only claims the sidebar. If the drawer should behave the same way, run the measurement after `showModal()`, for example from the Drawer, or in a `requestAnimationFrame`.

### A2 (ADVISORY): ops section links can point at sections not yet mounted

`OpsDlq` and `OpsRuns` mount only after the previous section settles (`ops/page.tsx`), so an early click on "Dead-letter queue" or "Scheduled runs" does nothing. Consider rendering the section headings unconditionally, or disabling the links until their target exists.

### A3 (ADVISORY): the clipped values depend on `max-width` on a table cell

The table-mode clip relies on `max-width` on a `td` in an auto-layout table. CSS 2.1 leaves that undefined. The previous code used the same mechanism, so this is not a regression, but the governance Target clip is new and was not checked outside Chromium. The full value is still in the DOM for screen readers. For sighted keyboard users it is only in `title`, which is the existing pattern used for JSON `data`.

### A4 (ADVISORY): CHANGELOG omissions and wording

- `PageHeader` actions are now `items-center justify-start` on every page (`ui.tsx:26`). The entry reads as a fix to the webhook endpoint only.
- The DLQ "Last result" column is no longer muted (`ops/dlq.tsx:133`). This is not listed.
- At 1024px the payment and dispute summaries are now single-column, where they used to be two-up from `sm`. "Only goes two-up when a whole id fits" is accurate, but the narrower layout is not mentioned.
- "Fixed: landing sections no longer use content-visibility" fixes something that only ever existed in the same Unreleased section. Consider folding the two entries together.
- The Lighthouse figures (landing about 85, dashboard about 88) were not re-measured in this review.

### A5 (ADVISORY): coverage

Add unit tests for the following:

- The Sidebar effect: it scrolls the parent and never the window, and it does nothing when the active item is already visible.
- The `AuditTrail` overflow hint, using a mocked ResizeObserver.
- The ops section link targets, as an id-exists check.
- The card-mode clip and wrap and the 375px overflow behind M1, as a Playwright `scrollWidth <= clientWidth` assertion on refunds and ledger panels at Pixel 7.
