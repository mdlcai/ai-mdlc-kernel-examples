Commit reviewed: b638db5
Previous round reviewed: 2a05154
Mechanism: Task tool general-purpose subagent
Verdict: PASS

Inputs: `git show 604b832` (RB-21), `git show b638db5` (RB-22), artifacts/reviewer/round-6.md (format, M1 and A4 reference), web/app/globals.css, web/app/(marketing)/{layout,page}.tsx, web/app/(marketing)/docs/page.tsx, web/components/marketing/{site-header,site-footer,content,site.test}.tsx, web/proxy.ts, web/node_modules/next/dist/client/link.d.ts (Next 16.4.2), web/app/(console)/payments/[id]/page.tsx, web/app/(console)/disputes/[id]/page.tsx, web/app/(console)/ledger/entries/[id]/page.tsx, web/e2e/**, and the RB-19/RB-20 diffs for `ui.tsx` and `ops/dlq.tsx` (to check the A4 CHANGELOG items).
Focus items: GD-0053 (RB-22, marketing Links set `prefetch={false}`).

# Reviewer Gate: focus check (GD-0053; RB-21 follow-up)

The review was read-only. No docker, e2e or web baseline was run, because a verification run was using the stack. Prefetching runs only in production builds, so the prefetch behaviour was judged from the code and the Next 16 `Link` types, not from observation. I ran `npm --prefix web run typecheck` (clean), `npm --prefix web run lint` (clean, max-warnings 0) and `npm run test:web` (63/63 passed).

## Checklist

| # | Item | Result | Rationale |
|---|---|---|---|
| 1 | RB-21: `a.mono` is cyan at rest and wraps again. No other rule regressed | PASS (advisory A1) | `globals.css:223-225` now holds only `color: var(--cyan)`. `white-space: nowrap` is gone, so `overflow-wrap: anywhere` in card cells (`:308`, `:365`) and in `dd` (`:112`) applies again, and "Journal je_…" can break at its space. The only `nowrap` left in globals.css is `.table td.cell-clip > *` (`:270`). Card mode overrides it, and no linked mono id uses it. No Link combines `mono` with `truncate` or `whitespace-nowrap`. The one remaining `mono … whitespace-nowrap` is the governance IP cell (`audit-trail.tsx:202`), which is not a link and is short. `ledger/entries/[id]/page.tsx:92` keeps `break-all`. Nothing else in globals.css changed. Round-6 M1 is closed. |
| 2 | RB-22: href, classes, aria and onClick unchanged; `prefetch={false}` valid; console and auth untouched; nothing inconsistent left | PASS | Every hunk only inserts `prefetch={false}`. The href, className, `data-primary`, `data-size`, `aria-label`, `aria-current`, key and `onClick={() => setOpen(false)}` stay the same: `site-header.tsx:30,35,42-51,81,89-98`, `site-footer.tsx:21-23`, `page.tsx:202-210,414-419,587-592`, `docs/page.tsx:136-141`. The footer change is a Prettier reflow only. Next 16.4.2 types `prefetch?: boolean \| 'auto' \| null` and documents `false` as "will not prefetch any data, even on hover" (`link.d.ts:46-59`), and typecheck passes. The commit touches only `app/(marketing)/**`, `components/marketing/**` and CHANGELOG.md. No console or auth file changed. SiteHeader and SiteFooter are used only in `(marketing)/layout.tsx:14,18`. Completeness: the only files under marketing that import `next/link` are the four changed ones, and every `<Link` in them now has `prefetch={false}`. The two grep hits without it on the same line are multi-line Links with the prop on the next line (`site-header.tsx:42-43,89-90`). The docs TOC anchors (`docs/page.tsx:77,82`) and the attribution link (`site-footer.tsx:33`) are plain `<a>` elements, which never prefetch. |
| 3 | Behavioural risk of disabling prefetch | PASS (advisory A2) | Navigation is unchanged. `Link` still intercepts the click and does a client-side transition, and only the speculative RSC fetch is skipped. Hash links (`/docs#idempotency`, `/#network` and so on) still soft-navigate and scroll to the fragment. On the same page this was never a prefetch matter. The mobile menu still closes through the preserved `onClick` handlers (`site-header.tsx:81,95`). `proxy.ts` only sets CSP and security headers, and it does no auth redirect, so no redirect depended on a prefetched payload. The cost is that the first click to /sign-in, /sign-up, /docs or /forgot-password waits for its RSC fetch (A2). |
| 4 | CHANGELOG accuracy | PASS (advisory A3) | RB-21 covers every item in round-6 A4. PageHeader alignment matches `ui.tsx` (`items-center justify-start`). The DLQ "Last result" entry matches `ops/dlq.tsx` (`text-muted` removed). "One column at 1024px" was added. The content-visibility revert is folded into the performance entry, and its separate Fixed bullet was removed. "No longer wrap" was dropped from the mono-link bullet, which matches the code. RB-22's sentence matches the diff, with one omission noted in A3. |
| 5 | e2e selectors unaffected | PASS | `web/e2e` never visits the marketing pages and never selects a marketing link. The only matches are the sign-in and sign-up form buttons (`getByRole('button', …)` in `support/helpers.ts:34`, `w1-onboarding.spec.ts:18,23` and `w10-ops.spec.ts:34`), which are in `(auth)` and unchanged. The `w6` `/^Journal je_/` link-name check is unaffected by RB-21, because the link text did not change. |
| 6 | Test coverage (advisory) | ADVISORY (A4) | `site.test.tsx` still checks the hrefs for the nav, actions and footer, and it passes. No test pins `prefetch={false}`, and no test covers the `a.mono` wrapping. |

## Findings

No BLOCKER, MAJOR or MINOR.

### A1 (ADVISORY): the bare id in the 1024px aside can still overflow (pre-existing)

Removing `nowrap` restores the break at the space in "Journal je_…". The id token itself is about 227px at 14px (`payments/[id]/page.tsx:272`) and about 194px at 12px (`disputes/[id]/page.tsx:165`). Both are wider than the roughly 190px content box of the aside at 1024px. Neither link has `overflow-wrap: anywhere` or `break-all`, so the token can still run a little past the panel. This predates RB-19, so it is not a regression. Round-6 M1 suggested the fix: add `overflow-wrap: anywhere` (for example `[overflow-wrap:anywhere]`) to those two links, or to `a.mono` globally. This is estimated, not checked visually.

### A2 (ADVISORY): the trade-off for the first click

With `prefetch={false}`, Next 16 also skips the hover prefetch. The first navigation from the marketing site to /sign-in, /sign-up, /docs or /forgot-password waits for the full RSC round-trip. On localhost behind Caddy that is small, but it is a deliberate trade of a slower first click for lower TBT. A later option is to prefetch the two auth CTAs on hover or intent, for example with `router.prefetch` in `onMouseEnter` or `onFocus`. That would keep TBT clean and restore the instant navigation. The 80–100ms TBT gain and the landing score of about 85 are the commit's own measurements and were not re-measured in this review.

### A3 (ADVISORY): the CHANGELOG wording is slightly narrow

The RB-22 entry says marketing links no longer prefetch "the sign-in, sign-up and docs routes". The change also stops the logo and footer from prefetching `/`, and the footer from prefetching `/forgot-password`. "Marketing links no longer prefetch routes" would be exact. The Lighthouse figures in the same entry were not re-measured here.

### A4 (ADVISORY): coverage

- Add a check that every marketing `Link` passes `prefetch={false}`. It could mock `next/link` and record the props, or be a lint rule scoped to `app/(marketing)` and `components/marketing`. Without one, a new marketing CTA could quietly bring the TBT regression back.
- Add a style assertion, or a 375px e2e or visual check, that `a.mono` has `white-space: normal`. That would catch any return of round-6 M1.
