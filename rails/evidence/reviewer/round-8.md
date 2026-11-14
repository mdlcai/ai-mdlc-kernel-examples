Commit reviewed: be7df80
Previous round reviewed: b638db5
Mechanism: Task tool general-purpose subagent
Verdict: PASS

Inputs: `git show be7df80` (RB-23: CHANGELOG.md, web/app/(marketing)/marketing.css, web/app/(marketing)/page.tsx, web/components/console/live-feed.tsx), artifacts/reviewer/round-7.md (format reference), DESIGN-TEMPLATE.html (literals `#B6A2FF` and `#C9B8FF`), web/app/globals.css (dark and light tokens, `.panel`), web/app/(console)/payments/page.tsx, web/e2e/**.
Focus items: GD-0054 (RB-23: landing grid CSS, ecosystem colour tokens, LiveStatus min width).

# Reviewer Gate: focus check (GD-0054)

The review was read-only. No docker, e2e or web baseline was run, because a verification run was using the stack. Layout and contrast were judged from the code and computed, not checked visually. I ran `npm --prefix web run typecheck` (clean), `npm --prefix web run lint` (clean, max-warnings 0) and `npm run test:web` (17 files, 63/63 passed).

## Checklist

| # | Item | Result | Rationale |
|---|---|---|---|
| 1 | Grid rule | PASS | `marketing.css:217-228`: at `min-width: 768px`, `.mk #network .cards` and `.mk #ecosystem .cards` use `repeat(3, minmax(0, 1fr))`, and `> :last-child:nth-child(3n + 1)` spans `1 / -1`. `NODES` has 7 entries (`page.tsx:20-62`: Stripe, Sandbox, Visa · Mastercard, Amex · Discover, Fiserv · Worldpay, ACH · Bitcoin · Lightning, Scan to pay). `ECO` has 7 (`page.tsx:114-132`). In both, 7 = 3n+1, so the 7th card spans the row. The only `id="network"` and `id="ecosystem"` are `page.tsx:227` and `:553`, each with one `ul.cards` (`:239`, `:561`). No other CSS references these selectors. The base `.mk .cards` rule (`:211`) still applies everywhere else. Overflow at 768: `.wrap` padding is 24px (`:31`), so the content is 720px. After two 12px gaps, each column is about 232px. That leaves about 196px of content in `.mk-node` (padding 18) and about 192px in `.card` (padding 20), or about 15px less with a desktop scrollbar. The longest token that cannot break is an 18px h3 word, such as "THRΞΛTWΛTCH" (about 140px). The longest meta is "DEFERRED · NOT IN V1" (about 180px with the dot, and it can wrap at its spaces). `minmax(0, 1fr)` stops the min-content size from pushing a track wider. No overflow is expected. |
| 2 | Tokens (ADR-044) | PASS (advisory A1) | `--eco-rails` and `--eco-stardancer` are new names, declared on `.mk` (`marketing.css:10-11`) and overridden in `:root[data-theme='light'] .mk` (`:19-20`). They sit beside `--hero-grad`, `--soft` and the other tokens there. The dark values `#b6a2ff` and `#c9b8ff` match the template's own ecosystem literals (`DESIGN-TEMPLATE.html:1254` rails `c: '#B6A2FF'` and `:1259` sd `c: '#C9B8FF'`), so dark mode renders exactly as before. The commit does not touch globals.css or DESIGN-TEMPLATE.html, and no template token changed. In light mode the cards are `.panel` (`globals.css:140-145`): a gradient from `--panel-a` rgba(255,255,255,.92) to `--panel-b` rgba(246,248,252,.92), over `--bg` #f6f8fc. Composited, that runs from about #fefeff to #f6f8fc. Not `--surf`. `--violet` light (#6a3ff0) is 5.86:1 on white and 5.51:1 at the bottom of the gradient, so it passes everywhere. `#7b5ce0` is 4.72:1 on white, about 4.60:1 where the h3 sits (25-45% down the card), and 4.44:1 at the very bottom (#f6f8fc). The name renders at 18px and is not large text, so 4.5:1 applies. It passes where the text actually is, with little margin (A1). Dark contrast is 7.5-11:1. Light mode applies only through `data-theme='light'` (`globals.css:40`), with no `prefers-color-scheme` path, so the override covers every way light mode is reached. |
| 3 | `text-wrap: balance` scope | PASS | `marketing.css:705-707` sets `.mk-final .eyebrow`. The only `.eyebrow` inside `.mk-final` is the closing line at `page.tsx:575`. The section eyebrows sit outside `.mk-final` and are unaffected. |
| 4 | LiveStatus `lg:min-w-32` | PASS (advisory A2) | There are two users. (a) `payments/page.tsx:105` in `PageHeader actions` with `justify-start lg:justify-end`: from lg up the reserved 8rem is unchanged, and below lg the status sizes to its text, so "New payment" sits beside it, as intended. (b) The dashboard `LiveFeed` header (`live-feed.tsx:50-54`), `flex justify-between`: the status is the end item, so with no min width only its own box narrows. The "Live activity" heading does not move, and the text still sits at the right edge. e2e `w2-payment.spec.ts:15-16` checks `feed.getByRole('status')` text `Live`. Role and text are unchanged. |
| 5 | CHANGELOG accurate to the diff | PASS (advisory A1) | The two new Fixed bullets (CHANGELOG `:30-31`) match the CSS, tokens, balance and LiveStatus change. The RB-22 performance sentence was also reworded to "(header, footer, logo and page calls to action) no longer prefetch their routes", which closes round-7 A3. "Keep AA contrast in the light palette" holds at the text position (see #2). The commit message's "4.72:1 on white" is measured against white, not the card surface. |
| 6 | e2e selectors unaffected | PASS | `web/e2e` never visits the marketing pages and does not select `#network`, `#ecosystem` or their list labels. The only LiveStatus selector is `w2-payment.spec.ts:16` (role `status`, text `Live`), which is unchanged. The other `getByRole('status')` hits (`w5`, `w6`, `w10`, `w11`) target different components. |

## Findings

No BLOCKER, MAJOR or MINOR.

### A1 (ADVISORY): the light `--eco-stardancer` passes with a thin margin, and was measured on white

`#7b5ce0` comes to about 4.60:1 behind the h3 on the light `.panel` gradient, against the 4.72:1 the commit message gives for white. At the bottom of the card it falls to 4.44:1. No text sits there today, but if the card layout changes (for example the name is moved down, or the card is centred vertically in a spanning row), the margin is gone. A shade one step darker, such as `#7552dc` or `#7350d8`, would clear 4.5:1 on `#f6f8fc` and keep the lilac hue. Future contrast claims should be measured against the composited panel colour rather than white.

### A2 (ADVISORY): below lg the payments header can shift on connect

Below lg, the status has no reserved width. When the text changes ("Connecting…" → "Live" → "Reconnecting…"), the "New payment" button moves sideways by the difference in text width. The longest text, "Paused — sign in again to resume", may push the button onto its own line. This is the deliberate trade made in the commit, and small, but it brings back a little layout shift on tablet and phone that the old `min-w-32` prevented. One alternative is to reserve a smaller width below lg, for example `min-w-24`, which fits "Reconnecting…". Separately, the class order (`items-center lg:min-w-32 gap-2`) is not Tailwind-sorted. Lint does not enforce sorting, so this is cosmetic.

### A3 (ADVISORY): coverage

No unit test pins the landing grid counts. A test that asserts 7 items in each list, or a `data-count` attribute, would catch a change to 6 or 8 items, which would quietly break the "lone last card spans" rule (3n+1). No test covers the light ecosystem colours.
