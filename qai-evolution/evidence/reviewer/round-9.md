Commits reviewed: 2c772ef897eca5a092f335c6d0056312ff2d9e62 (RB-8)
Previous round reviewed: ce0900b (round 8, C-15)

# Reviewer Gate: Review Round 9 (RB-8)

- **Mechanism:** a fresh-context subagent did a static review of `git show 2c772ef` against the findings in artifacts/reviewer/round-8.md. It extracted every class selector from the deleted lines of site.css and grepped apps/web (`*.tsx`, `*.ts`, `*.css`, excluding node_modules and .next) for each one as an exact token, so hyphenated lookalikes did not count as matches. It ran `npm run lint:invariants`, `npx vitest run components/ui/Artwork.test.tsx --maxWorkers=1`, `npx tsc --noEmit` and `npx eslint --max-warnings 0` on Artwork.tsx, Artwork.test.tsx and page.tsx. It did not use docker or a browser, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-8 | 2c772ef Review round 8 follow-ups | **PASS** (0 minor, 2 notes) | N-R8-1 to N-R8-5 are all resolved correctly. The deleted CSS was dead, Artwork is clean, the alt text is plain language, and ADR-40 now quotes the CSP accurately. Lint is unchanged. |

## RB-8 (2c772ef): round 8 follow-ups

| Check | Result | Evidence |
|---|---|---|
| N-R8-1: dead CSS removed | PASS | The deletion is 146 non-blank lines (177 lines with blanks) at site.css:242. It defines 25 class selectors: `.circuit-art`, `.console`, `.console-bar`, `.console-body`, `.console-dots`, `.console-foot`, `.console-pane`, `.console-title`, `.dot-entangle`, `.gate`, `.gate-clifford`, `.gate-entangle`, `.gate-measure`, `.gate-text`, `.hist`, `.hist-axis-line`, `.hist-bar`, `.hist-bar-zero`, `.hist-label`, `.link-entangle`, `.meter`, `.pane-head`, `.pane-meta`, `.qlabel` and `.wire`. 22 of them have 0 exact-token hits in apps/web. `console`, `gate` and `wire` do have hits, but every hit is prose or code (`console.error`, "gate counts", "release gate pending review", the circuit.css comment "--- console ---"). None is a `className`/`cx` token or a CSS selector: `grep '\.(gate\|wire\|console\|hist\|meter)\b'` over the CSS returns nothing. Lookalikes such as `cb-console*`, `gate-chip*`, `gate-legend` and `cb-qlabel` are different classes and are untouched. No template-literal class construction (`gate-${...}` and similar) exists. The e2e specs do not reference these classes. |
| Shared utilities kept | PASS | `.sr-only` (base.css:139), `.mono` (base.css:151) and `.label` (base.css:155) are still present. The deleted lines contain `.qlabel`/`.hist-label` but no bare `.label`, `.mono` or `.sr-only` rule. |
| site.css well-formed | PASS | The deletion begins after the `min-width: 1024px` hero block closes and ends just before `/* Sections */`. After the deletion the file has 139 `{` and 139 `}`. |
| N-R8-2 / N-R8-4: Artwork | PASS | Artwork.tsx no longer has the `sizes` prop or attribute and no longer imports `cx`. Its only import is `import type { StaticImageData }`, and that import is used. `className={className}` is passed through unchanged, so React omits the attribute when it is undefined. Nothing in apps/web still passes `sizes=` to Artwork, and no CSS rule targets `.artwork`, so removing the class affects nothing. `.hero-visual-img` (observatory.css:33-36) already sets `display:block; width:100%; height:auto`, so the image's layout did not depend on the base class. The test now asserts `className === 'hero-visual-img'`. All other assertions, including `no style attribute`, are unchanged. |
| N-R8-5: alt text | PASS | page.tsx:148 now reads "A qubit drawn as a Bloch sphere, a superposition of zero and one, hovering above a glowing QAI processor beside the steps Circuit, Simulate, Optimize and Validate." It contains no math glyphs, is meaningful plain language, and still describes the scene (ADR-40). |
| N-R8-3: ADR-40 CSP quote | PASS | DECISIONS.md:301 says `img-src 'self' data:`, and proxy.ts:16 has `"img-src 'self' data:"`. They match. |
| Lint / tests | PASS, unchanged | `npm run lint:invariants`: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web-baseline, `missing FRONTEND-AUDIT.json`), the known pre-existing state. Vitest: 1 file, 2 tests passed. `npx tsc --noEmit` exits 0. eslint `--max-warnings 0` on the three changed TS/TSX files exits 0. |

## Findings

- **N-R9-1 (note, RB-8): commit message.** The message says "177-line" block. That is the raw count including blank lines, and it agrees with the diffstat (177 deletions in site.css). It contains 146 non-blank lines. This is accurate and needs no action.
- **N-R9-2 (note, RB-8): apps/web/components/ui/Artwork.test.tsx:23-26.** The lazy-load test does not assert that a missing `className` renders no `class` attribute. Now that `cx` is gone, that is the new pass-through behaviour. React handles it correctly, so this is optional extra coverage.
- Carried forward: N-R8-6 (no web baseline evidence for the hero LCP/CLS, INV-2 pre-existing) is still open and out of RB-8's scope.

Counts: 0 blocker, 0 major, 0 minor, 2 notes.

## Overall

**PASS.** RB-8 closes every round 8 finding it targeted (N-R8-1 to N-R8-5) with no behavioural side effects. The removed CSS was provably unused, the shared utilities are intact, the Artwork primitive is minimal with no unused imports, the alt text reads well in a screen reader, and the ADR-40 CSP quote matches proxy.ts. Lint, types, eslint and the unit test are all unchanged or green.

Verdict: PASS
