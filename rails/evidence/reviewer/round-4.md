Commit reviewed: 589798f
Previous round reviewed: 901ed2a
Mechanism: Task tool general-purpose subagent
Verdict: PASS

Inputs: `git show 589798f`, artifacts/reviewer/round-3.md, SPEC.md §Design System (SPEC.md:490-506), DECISIONS.md:23 (template component patterns), web/app/globals.css, web/app/(marketing)/marketing.css, web/components/console-shell.tsx, web/components/session-provider.tsx, web/components/marketing/site-header.tsx, web/e2e/.
Focus items: GD-0038 (RB-8)

# Reviewer Gate — focus check (GD-0038)

The review was read-only. No docker builds and no web baseline were run. I ran `npm --prefix web run typecheck` (clean), `npm --prefix web run lint` (clean, max-warnings 0) and `npm run test:web` (63/63 passed).

## Checklist

| # | Item | Result | Rationale |
|---|---|---|---|
| 1 | Sign out works in every session state on mobile | PASS | `console-shell.tsx:128` passes `onSignOut` only when `session` is set. That is the same condition as the header button (`:177`), so no state gains or loses Sign out compared with desktop. When the session is null (loading, error or anonymous), neither place offers Sign out, as before. `signOut` (`session-provider.tsx:119-127`) clears the session and calls `router.replace('/sign-in')` in a `finally`, so it works from inside the modal dialog. The console layout unmounts on navigation, which also tears down the dialog. |
| 2 | Drawer content is keyboard and screen-reader accessible | PASS | The Theme and Sign out buttons are native `<button>`s inside the `showModal()` `<dialog aria-label="Navigation">` (`:92-112`). They follow the Sidebar in DOM and tab order, and the Theme button keeps its descriptive `aria-label`. The modal dialog makes the rest of the page inert. A closed dialog is not rendered, so nothing is in the accessibility tree on desktop. In a modal dialog the UA sets `overflow:auto`, so the footer stays reachable when the operator nav is long. |
| 3 | No viewport where Theme or Sign out is unreachable | PASS (advisory A1) | Below 640px they are in the drawer only, and the Menu button shows because it is `md:hidden`. From 640 to 767px they are in both the header (`hidden sm:inline-flex`) and the drawer. From 768px up they are in the header only, and the drawer cannot be opened. Nothing is unreachable at any width. 640-767 duplicates them, which is harmless (A1). |
| 4 | `:where(:not(.btn))` keeps specificity and is supported | PASS | `.mk a:where(:not(.btn))` has specificity (0,1,1), the same as `.mk a`, because `:where()` contributes 0. Unlayered `.mk-nav a` (`marketing.css:70`) and `.mk-fcols ul a` still win over it as before. `:where()` and a simple `:not()` are supported in Chrome 88+, Firefox 78+ and Safari 14+, inside the project's browser targets. The non-primary `.btn` links now take `color: var(--text)` from `.btn`, the same value they inherited before. |
| 5 | Removing `overflow-hidden` keeps corner clipping and does not affect non-table panels | PASS | `overflow-hidden` is a Tailwind utility in `@layer utilities`, so it outranked `.panel:has(.table){overflow-x:auto}` in `@layer components` (`globals.css:255-258`). That is the root cause, and removing the utility is the correct fix. `overflow-x:auto` makes `overflow-y` compute to `auto`, which still clips content to the `border-radius: var(--r)` padding box, so the rounded corners hold. Each of the four edited panels contains only a `.table`, and they are already `tabIndex=0` labelled regions (WCAG 2.1.1). No non-table panel was touched. The sticky `thead` was already scoped to the panel, because overflow:hidden also creates a scroll container, so its behaviour is unchanged. |
| 6 | No other unlayered `.mk` rule overrides button colours | PASS (advisory A3) | `marketing.css` has these anchor rules: `.mk-nav a` (no `.btn` children; `site-header.tsx:33-39`), `.mk-mnav a` (`:118`), `.mk-fcols ul a` (`:755`) and `.mk-toc a` (`:820`, `:948`). Only `.mk-mnav a` matches `.btn` links (the mobile-menu `.acts`, `site-header.tsx:80-90`). It sets no colour or background, so there is no contrast regression. It does override `.btn` padding, radius and display, which is A3. |
| 7 | Tokens and design system (ADR-044) | PASS | No new or changed tokens. The new markup uses existing utilities (`border-line`) and the template's `.btn` / `.btn.sm` (36px) pattern (DECISIONS.md:23). SPEC.md:497's "collapses to a focus-trapped drawer below 768px" still holds. |
| 8 | CHANGELOG accurate to the diff | FAIL (MINOR M1) | Three of the four entries match the diff. The contrast entry names the wrong theme (M1). |
| 9 | Test coverage | ADVISORY (A2) | No e2e or unit test exercises Sign out or the theme toggle at any viewport. |

## Findings

No BLOCKER or MAJOR.

### M1 — MINOR — the CHANGELOG names the wrong theme for the CTA contrast fix

`CHANGELOG.md:95` says the primary buttons "keep their white label in dark mode". The defect was in the light theme. `.mk a { color: inherit }` gave the CTA the inherited `--text`.

- Light theme: `--text` is `#0b1222` (`globals.css:58`). On `--violet-fill #6a3ff0` (`globals.css:50`) that measures 3.19:1, which is the 3.18:1 the commit cites.
- Dark theme: the inherited text is `#f4f8ff`, already a light label at about 5.5:1.

How it misleads: a reader or QA re-checks dark mode, sees it was never broken, and misses that the light-theme regression is the one to guard. The commit message itself ("dark text on the violet fill") is correct. The scope is also wider than the entry says: the fix applies to every marketing `.btn[data-primary]` link, including the site-header "Sign up" CTA on all marketing pages (`site-header.tsx:42,84`), not only the landing and docs pages.

Fix: change "in dark mode" to "in the light theme" and "landing and docs" to "marketing".

## Advisories

- **A1 — INFO — Theme and Sign out are duplicated from 640 to 767px.** At this width both the header (`console-shell.tsx:176-186`) and the drawer footer (`:104-111`) show them. That is harmless, but the drawer footer could be `sm:hidden` to match the code comment at `:103` ("Below sm …").
- **A2 — LOW — no test covers sign-out or the theme toggle.** A case-insensitive grep for "sign out" and "theme" in `web/e2e` finds nothing, and no unit test imports `ConsoleShell`. The change moved the only Sign out control on the Pixel 7 project into the drawer, and no test would notice if it vanished. A drawer-aware `signOut(page)` helper, like `navTo` in `e2e/support/helpers.ts:39-44`, plus one assertion in any W-spec under both projects would close the gap.
- **A3 — LOW, pre-existing — `.mk-mnav a` restyles the mobile-menu `.btn` links.** `marketing.css:118-126` is unlayered (0,1,1), so on the mobile-menu CTAs it overrides `.btn`'s `display:inline-flex`, `padding:0 16px` and `border-radius:10px` with `flex`, `0 12px` and `8px`. It is the same class of problem as the one RB-8 fixed, but cosmetic only, and the 44px height is kept. Consider `.mk-mnav a:where(:not(.btn))`.
- **A4 — LOW — eleven other table panels keep `overflow-hidden`.** RB-8 removes it from only the four panels the baseline flagged at 1280px. These table-bearing panels still clip instead of scrolling when the table is wider than the panel:
  - customers/page.tsx:141
  - developers/api-keys/page.tsx:106
  - developers/events/page.tsx:162
  - developers/webhooks/page.tsx:87
  - disputes/page.tsx:125
  - ledger/page.tsx:127,258
  - ledger/entries/[id]/page.tsx:101
  - payments/page.tsx:201
  - refunds/page.tsx:113
  - components/webhook-deliveries.tsx:124

  They also lack the focusable-region treatment. The container query turns them into cards at 640px and below, so the exposure is narrow-desktop widths with long ids. A follow-up could apply the same pattern to all of them.
- **A5 — INFO — the drawer buttons are 36px.** The drawer's Theme and Sign out use `data-size="sm"`, which is 36px (template `.btn.sm`). SPEC.md:500 says "buttons 44px", and the commit makes the Menu button 44px for that reason. 36px passes WCAG 2.5.8 (24px) and the drawer has room, so dropping `data-size="sm"` there would match the 44px target on phones.
