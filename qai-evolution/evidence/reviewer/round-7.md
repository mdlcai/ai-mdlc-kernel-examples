Commits reviewed: 08c4029503c9ab1ccc560fc7eaaf31e34092f666 (C-12), e558094ba20528b95700263d62d2135fdcaa3919 (C-13)
Previous round reviewed: dbc6219

# Reviewer Gate: Review Round 7 (C-12, C-13)

- **Mechanism:** a fresh-context subagent did a static review of `git show 08c4029` and `git show e558094`. It ran `npm run lint:invariants`, `git grep`/`grep` over apps/web, DESIGN-TEMPLATE.html and SPEC.md, and diffed the token blocks. It did not use docker or a browser, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-12 | 08c4029 Wordmark `Ψ ◈ QAI // QUANTUM INTELLIGENCE` (ADR-39) | **PASS** (1 minor, 3 notes) | Matches ADR-39. No "I AM" is left in apps/web, DESIGN-TEMPLATE.html or SPEC.md. The CSS collapse is correct in both headers. |
| C-13 | e558094 release 0.4.1 | **PASS** (1 note) | Only version strings change, 0.4.0 → 0.4.1. INV-34 passes. The lockfile touches only the three root and workspace version fields. |

## C-12 (08c4029): rebrand

| Check | Result | Evidence |
|---|---|---|
| Matches ADR-39 | PASS | The wordmark (Brand.tsx:29-33), root and app title templates (app/layout.tsx:21-36, app/app/layout.tsx:8), page.tsx:14 absolute title, manifest.ts:5 `name` (short_name stays `QAI`), opengraph-image.tsx:3,33-35, global-error.tsx:12, SiteFooter.tsx:8, DESIGN-TEMPLATE.html:2,7,434-439,564,867 and SPEC.md:528 all change. The tagline uses mono, 0.22em tracking, uppercase and `--color-fg-2` (site.css:28-34), as ADR-39 specifies. The package, API title and repo docs are untouched, as the "Not changed" clause says. |
| Leftover "I AM" | PASS | `grep -rn "I AM"` over apps/web (excluding node_modules/.next), DESIGN-TEMPLATE.html and SPEC.md finds 0 hits. The remaining hits are historical: CHANGELOG 0.4.0/0.4.1 entries, VERSION.md:13, DECISIONS ADR-36/39 and artifacts/. |
| CSS: tagline hidden below 1280px | PASS | site.css:456-461 `@media (max-width: 1279px) { .site-nav .brand-slash, .site-nav .brand-word { display: none } }` and shell.css:131-136 do the same for `.topbar`. Specificity (0,2,0) beats the single-class `.brand-*` rules, and `.brand-word` declares no `display`, so the globals.css import order (shell before site) does not matter. The old `.site-nav .brand-lead` hide rule is gone correctly, because the lead is now `QAI` and must stay visible. The `.topbar .topbar-brand, .groups, .subnav-bar` rule that follows (shell.css:138) is intact, and its ≥1024 counterpart (shell.css:323) is unchanged. No other selector is affected. |
| Accessibility | PASS | Brand.tsx:19 sets `aria-label="QAI Quantum Intelligence home"`. Ψ, ◈, `//` and the wave SVG are `aria-hidden`. The visible words "QAI … Quantum Intelligence" appear inside the accessible name, which meets WCAG 2.5.3 label-in-name. The DOM text is "Quantum Intelligence" and CSS uppercases it, so screen readers do not spell the letters out. The template's `aria-label` matches (DESIGN-TEMPLATE.html:434). |
| Design tokens | PASS | `git show 08c4029` has no `+`/`-` lines declaring a `--token` in DESIGN-TEMPLATE.html or globals.css. The only globals.css change is the comment at line 9. The OG image's new literals `#B7A0FF` and `#C3C9DA` equal `--color-violet-strong` (template:25, globals:28) and `--color-fg-2` (template:17, globals:18). |
| Tests and e2e | PASS | No `*.spec.ts`, `*.test.ts` or `*.test.tsx` under apps/web asserts the brand text, a page title (`toHaveTitle`) or the home-link name. The only nearby assertion is auth.spec.ts:12, "Create your QAI workspace", which the commit does not affect. |

## C-13 (e558094): release 0.4.1

| Check | Result | Evidence |
|---|---|---|
| Changes version strings only | PASS | 10 files and 15 lines change, and every line is `0.4.0` → `0.4.1`. The files are .env.example, apps/web/lib/env.ts, apps/web/package.json, compose.yaml (4), package-lock.json (3), package.json, both pyprojects, `qai/__init__.py` and config.py. This is the same set as C-10. |
| package-lock.json | PASS | 3 hunks: the top-level `version`, `packages[""].version` and `packages["apps/web"].version`. No dependency entry changes. |
| No missed location | PASS | `git grep 0.4.0` outside CHANGELOG and artifacts finds only history (BUILD-STATE, DECISIONS:255, VERSION.md:13) and unrelated dependency versions in the lockfile (type-check, espree, retry). VERSION.md:14 and CHANGELOG.md:3 have 0.4.1 entries. |
| INV-34 / lint | PASS, unchanged | `PASS INV-34 version-agreement files=10`. The overall result is `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web-baseline, files=0), which is the known pre-existing state. Nothing changed. |

## Findings

- **N-R7-1 (minor, C-12): apps/web/app/login/_auth/AuthShell.tsx:25 and apps/web/app/not-found.tsx:12.** These pages render the full `<Brand />` outside `.site-nav` and `.topbar`, so the 1279px collapse does not reach them. The new mark is much wider than the ADR-36 mark. A rough estimate below 640px, where the wave is hidden, is about 300px: a 20-character mono tagline at 12px plus 0.22em tracking is about 197px. `.auth-top` has 32px padding on each side (auth.css:11). That probably still fits at 375px, but at a 320px viewport (WCAG 1.4.10 reflow) it is likely to cause horizontal overflow. This is an estimate and was not measured. **Fix:** in a C- commit, extend the collapse to `.auth-top` and `.status-card` (or make it the base `.brand` behaviour below 640px), then confirm at 320 and 375px.
- **N-R7-2 (note, C-12): DECISIONS.md:270.** ADR-36's status still reads "accepted". ADR-39's title says it supersedes ADR-36, but ADR-36 itself does not record that. **Fix:** change the status to "superseded by ADR-39".
- **N-R7-3 (note, C-12): DESIGN-TEMPLATE.html:180 and :414.** The template's `.brand .word` is still display-weight 700 rather than the mono, tracked, uppercase, `fg-2` tagline that ADR-39 describes and site.css:28-34 implements. The template also has no 1280px tagline collapse. Its only narrow rule hides `.brand-wave` at line 414. These are styling rules, not tokens, so ADR-01/30 holds. The template is just a less exact reference for the mark.
- **N-R7-4 (note, C-12): CHANGELOG.md:4.** "The tagline collapses to `Ψ ◈ QAI` on narrow headers" is accurate, but the breakpoint is 1280px, which covers every tablet and most laptops. The full tagline appears only at ≥ 1280px. That is intentional and documented in ADR-39, so this is recorded for awareness only.
- **N-R7-5 (note, C-13):** there is no `v0.4.1` tag (`git tag -l "v0.4*"` lists only v0.4.0). C-10 was tagged. The commit message does not claim a tag, so this is a release-process gap only if tags are expected for every release.

Counts: 0 blocker, 0 major, 1 minor, 4 notes.

## Overall

**PASS.** C-12 implements ADR-39 consistently, with no leftover brand text, correct collapse CSS, an accessible link name and no token drift. C-13 is a clean version-only release at 0.4.1, INV-34 passes, and the lint state is unchanged at 32 pass / 1 fail (INV-2).

Verdict: PASS
