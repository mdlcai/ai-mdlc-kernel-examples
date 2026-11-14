Commits reviewed: ce0900b6989ad96f33c5883e258b9f15b6f14726 (C-15)
Previous round reviewed: 08c4029, e558094 (round 7)

# Reviewer Gate: Review Round 8 (C-15)

- **Mechanism:** a fresh-context subagent did a static review of `git show ce0900b` (the .webp is binary; only its RIFF/VP8 header was read). It ran `npm run lint:invariants`, `npx vitest run components/ui/Artwork.test.tsx --maxWorkers=1`, `npx tsc --noEmit` and `npx eslint` on the three changed TS/TSX files, and grepped apps/web for every class name the deleted HeroConsole used. It did not use docker or a browser, did not read FRONTEND-AUDIT.json, did not run scripts/govern.mjs, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-15 | ce0900b Landing hero artwork (ADR-40) | **PASS** (1 minor, 5 notes) | The Artwork primitive is correct and CSP-safe. The orbit CSS is fully removed, tokens are unchanged, there is a unit test, and lint is unchanged. HeroConsole's CSS is left behind in site.css as dead code. |

## C-15 (ce0900b): landing hero artwork

| Check | Result | Evidence |
|---|---|---|
| Static import types | PASS | next-env.d.ts:2 references `next/image-types/global`, which declares `*.webp` as `StaticImageData`. `@/*` maps to `./*` (tsconfig.json:18), so `@/assets/hero-quantum.webp` resolves. `npx tsc --noEmit` exits 0. The Dockerfile build stage copies all of `apps/web` (Dockerfile:15), so `assets/` is present at build time, and the emitted media ships in `.next/static` (Dockerfile:26). |
| Image file | PASS | The header is `RIFF....WEBPVP8 `. The VP8 frame size is 0x044D × 0x02C0 = 1101 × 704, which matches ADR-40 and the commit message. It is 115,276 bytes. |
| Artwork props and attributes | PASS | Artwork.tsx:26-36 sets `src`, `width`/`height` from the import (intrinsic ratio, no layout shift with `height: auto` at observatory.css:36), and `alt` (required prop). It also sets `decoding="async"`, `loading` eager/lazy by `priority`, and `fetchPriority` high/auto, which React 19 renders as `fetchpriority`. The test asserts this. `import type { StaticImageData }` adds no runtime dependency on next/image. The `eslint-disable-next-line @next/next/no-img-element` targets a rule that is loaded (eslint.config.mjs:27-31), and eslint with `--max-warnings 0` is clean. |
| Dead imports / references | PASS | page.tsx drops the HeroConsole import and adds `heroArt` and `Artwork`, and both are used. `grep -rn HeroConsole` over apps/web (including e2e) finds 0 hits. No e2e spec selected `.console` or the hero. |
| CSS: orbit removal | PASS | `grep -rn "orbit\|hero-core"` over apps/web finds 0 hits. That covers the rules, the `orbit-spin` keyframes and the removed `max-width: 767px` block. No `prefers-reduced-motion` rule referenced them. `.hero-art` stays as an empty, `aria-hidden` colour wash (page.tsx:116, observatory.css:11-26). |
| CSS: mask | PASS | observatory.css:37-44 intersects two linear-gradient masks. The `-webkit-mask-composite: source-in` is the correct legacy-WebKit equivalent of the standard `mask-composite: intersect`, and the standard pair follows the prefixed pair, so modern engines use the standard pair. A mask is not an inline style, so the CSP is unaffected. |
| CSS: print rule | PASS | observatory.css:318-323 still hides `.hero-art` (now only the gradient wash) and `body::before`. That still makes sense because both are decorative backgrounds. The artwork itself prints, which is acceptable for a content image with alt text. |
| CSP / inline styles | PASS | Neither Artwork.tsx nor page.tsx has a `style=` attribute, and the test asserts that the `<img>` has no `style` attribute. The image is a same-origin `/_next/static/media/*` URL, which `img-src 'self' data:` (proxy.ts:16) allows. Caddyfile:45-50 `handle /_next/static/*` adds `Cache-Control: public, max-age=31536000, immutable` and proxies to web, so the hashed file gets immutable caching with no Caddy change. |
| Conventions | PASS | The raw `<img>` lives in a `components/ui` primitive, and page.tsx styles it only through class names (`hero-visual`, `hero-visual-img`). `git show ce0900b -- apps/web/app/globals.css` is empty, so tokens are unchanged (ADR-30/37). There is a unit test (Artwork.test.tsx, 2 tests, both pass). The primitive is exported from components/ui/index.ts:1. ADR-40 matches the code, apart from N-R8-3. |
| Lint / tests | PASS, unchanged | `npm run lint:invariants`: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web-baseline, files=0), the known pre-existing state. Vitest: 1 file, 2 tests passed. |

## Findings

- **N-R8-1 (minor, C-15): apps/web/app/styles/site.css:242-417.** The whole "Product console in the fold" block is now dead CSS, about 176 lines. Every class it defines was used only by the deleted HeroConsole. Each name was checked against every `className=`/`cx(...)` token in apps/web/app, components and lib, and none of these 25 is used: `.console`, `.console-bar`, `.console-dots` (+ `span`), `.console-title`, `.console-body`, `.console-pane`, `.pane-head`, `.pane-meta`, `.circuit-art`, `.hist`, `.wire`, `.qlabel`, `.hist-label`, `.gate`, `.gate-clifford`, `.gate-entangle`, `.gate-measure`, `.gate-text`, `.link-entangle`, `.dot-entangle`, `.meter`, `.hist-axis-line`, `.hist-bar`, `.hist-bar-zero` and `.console-foot` (+ `div`/`dt`/`dd`), along with their `min-width: 640px` overrides at site.css:404-417. The generic names were checked specifically. `gate`, `wire`, `hist` and `meter` appear in apps/web only as prose or as different classes (`gate-chip*`, `gate-legend`, `gate-cat`, `gate-list`, `cb-console*`, `cb-link*`), so the bare selectors are provably unused. Do not remove the shared `.label`, `.mono` and `.sr-only` (base.css), which are used widely. **Fix:** delete site.css:242-417 in a follow-up C- commit. This is dead code with no behavioural effect.
- **N-R8-2 (note, C-15): apps/web/components/ui/Artwork.tsx:15,21,32 and apps/web/app/page.tsx:149.** Without a `srcset` carrying `w` descriptors, the HTML spec ignores `sizes`. So `sizes="(min-width: 1024px) 52vw, 100vw"` is harmless but does nothing. The ADR says the file is pre-sized and there is a single source. **Recommendation:** drop the `sizes` prop (or keep it only if a srcset is added later), so the API does not suggest responsive selection that is not happening.
- **N-R8-3 (note, C-15): DECISIONS.md:301.** ADR-40 says "CSP `img-src 'self'` unchanged", but the policy is `img-src 'self' data:` (proxy.ts:16). The conclusion still holds: same-origin, no change. The quoted directive should read `img-src 'self' data:`.
- **N-R8-4 (note, C-15): apps/web/components/ui/Artwork.tsx:27.** The base class `artwork` has no CSS rule anywhere in apps/web/app/styles. That is harmless as a hook, but either add a base rule (e.g. `display:block; max-width:100%; height:auto`, which would let `.hero-visual-img` shrink to just the mask) or drop the class.
- **N-R8-5 (note, C-15): apps/web/app/page.tsx:148.** The alt text contains the math glyphs `|ψ⟩ = α|0⟩ + β|1⟩`. Screen readers read these inconsistently (often "vertical line psi right angle bracket"). Consider "the state psi equals alpha zero plus beta one" or simply "a qubit state".
- **N-R8-6 (note, C-15):** this is a UI change, and INV-2 (web-baseline) is still the pre-existing `files=0` failure, so there is no baseline evidence of LCP or CLS for the new 115 KB eager hero image. That is not a regression in lint state. It should be covered when the web baseline is next produced.

Counts: 0 blocker, 0 major, 1 minor, 5 notes.

## Overall

**PASS.** C-15 implements ADR-40 as described. The new `Artwork` primitive is a typed, CSP-safe plain `<img>` with intrinsic dimensions, priority loading and required alt text, and it has a unit test. The orbit and hero-core CSS is removed cleanly, the mask syntax is correct, the image is same-origin and immutably cached via the existing Caddy `/_next/static/*` handler, tokens are untouched, and lint is unchanged at 32 pass / 1 fail (INV-2). The only cleanup worth scheduling is the dead HeroConsole CSS in site.css.

Verdict: PASS
