Commit reviewed: dbc6219aa8d9978c7db0388978bd8d61d66bdf3a
Previous round reviewed: dff13cf

# Reviewer Gate: Review Round 6 (C-10, RB-6, C-9)

- **Mechanism:** a fresh-context subagent did a static review of `git log --oneline dff13cf..HEAD`, leaving out the "mdlc: governance record" commits. The commits in scope were RB-6 3d1fbd3, C-9 1fb4553 and C-10 08a8584. It ran `npm run lint:invariants` and `git grep`. It did not use docker, did not read FRONTEND-AUDIT.json, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-10 | 08a8584 release 0.4.0 | **PASS** | Only version strings change, 0.3.3 → 0.4.0. INV-34 passes and tag v0.4.0 points at 08a8584. |
| RB-6 | 3d1fbd3 ADR-38 | **PASS** (cold read) | Each round-5 follow-up is applied as ADR-38 describes. |
| C-9 | 1fb4553 REPORT / COMPLIANCE / CHANGELOG | **PASS** (cold read, 1 low, 1 info) | One finding count is miscopied from round 4 (N-R6-1). |

## C-10 (08a8584): release 0.4.0

| Check | Result | Evidence |
|---|---|---|
| Changes version strings only | PASS | 10 files and 15 lines change, and every line is `0.3.3` → `0.4.0`. The files are .env.example, apps/web/lib/env.ts, apps/web/package.json, compose.yaml (4 lines), package-lock.json (3 lines), package.json, both pyprojects, `qai/__init__.py` and config.py `app_version`. This is the same set of 10 files that C-2 changed. |
| package-lock.json | PASS | 3 hunks: the top-level `version`, `packages[""].version` and `packages["apps/web"].version`. No dependency entry changes. |
| No missed location | PASS | `git grep 0.3.3` outside CHANGELOG finds only history: BUILD-STATE `last_checkpoint`, VERSION.md rows 0.3.3 and round-4.md. It also finds the dependency versions `@jridgewell/trace-mapping 0.3.31` and `for-each ^0.3.3`. VERSION.md already has a 0.4.0 row. |
| INV-34 | PASS | `PASS INV-34 version-agreement files=10 -- 0.4.0: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree`. The overall lint result is `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. The only failure is INV-2 (web baseline, files=0), which is the known environment blocker and not something C-10 caused. |
| Tag | PASS | `git rev-list -n1 v0.4.0` = 08a85840edd0. |
| CHANGELOG 0.4.0 | PASS | `## 0.4.0 (2026-11-01) — Re-Entry R1` names ADR-34, ADR-35 (RB-5), ADR-36 (C-7), ADR-37 (C-8 / F-20) and ADR-38 (RB-6). |

## RB-6 (3d1fbd3): Review round 5 follow-ups (ADR-38), cold read

- N-R5-2 fixed: in ui.css, `.page-title` at ≥ 1024px changes from 48px to 40px. That is the SPEC §6 h1 step (SPEC.md:524).
- N-R5-3 fixed: DESIGN-TEMPLATE.html now loads Space Grotesk 500/600/700 from Google Fonts in place of Manrope. Only the `<link>` changes and the `:root` tokens do not, so the ADR-01/30 rule that tokens are copied verbatim still holds.
- N-R5-5 fixed: `--sidebar-w` is removed from globals.css. It was never in the template (0 hits in DESIGN-TEMPLATE.html), and nothing in apps/web references it.
- N-R5-1, N-R5-4 and the design-4 tab-order minor are recorded as accepted, each with a reason. ADR-38 states its status and the source round.

## C-9 (1fb4553): REPORT, COMPLIANCE, CHANGELOG, cold read

The claims were checked against artifacts/reviewer/round-5.md, artifacts/security/pass-5.md and artifacts/design/score-4.md.

| Claim | Matches? |
|---|---|
| Round 5 PASS over RB-5, C-7, C-8 and F-20; 1 low, 4 info; 0 blocking | Yes (round-5.md:4, :82) |
| Security pass 5 PASS-ADVISORY: critical 0, high 0, medium 0, low 1, info 3. RB-5 fixed P5-1 and P5-3; P5-2 (about 8.4 s for 1 unit) and P5-4 (SDK looser than API) are accepted | Yes (pass-5.md:7, :50-55, :61) |
| score-4 was 3/5 first, with the mobile wordmark major (`.brand` overriding `.topbar-brand`), fixed in 061d159, then 4/5 | Yes (score-4.md:3, :18, :29-31). The commit 061d159 is named in round-5.md:9 and :56. |
| Design-4 minors: tab order, Evidence card, Theory spacing, 375 jobs scroll cue, Refresh alignment | Yes (score-4.md:19-23) |
| W2.5 says all 80 `:root` tokens are equal and muted text is about 6.6:1 | Yes (round-5.md:52; score-4.md:15) |
| W2.7 says reduced motion also stops the hero orbit | Yes (round-5.md:54; base.css:224) |
| S9 R1 caps, with huge ints → `OUT_OF_RANGE` and wrong types → `INVALID` | Consistent with pass 5 and RB-5 |
| R1-4 row: "2 low, 3 info" | **No.** round-4.md:90 reads "0 blocking, 3 low, 2 info" (N-R4-1..3 low, N-R4-4..5 info). |

## Findings

- **N-R6-1 (low, C-9): REPORT.md:108.** The R1-4 row reports reviewer round 4 as "2 low, 3 info". artifacts/reviewer/round-4.md:79-90 has 3 low (N-R4-1, N-R4-2, N-R4-3) and 2 info (N-R4-4, N-R4-5). **Fix:** change the row to "3 low, 2 info" in a C- commit.
- **N-R6-2 (info, C-9): COMPLIANCE.md S11.** The S11 note says the F-20 redesign renders text only with no `dangerouslySetInnerHTML` or inline style, and cites Reviewer round 5. Round 5 does not state that explicitly; it covers styling locality. Security pass 5 covers F-19 only. A grep of apps/web/app and apps/web/components confirms that `dangerouslySetInnerHTML` is absent. The only `style={{}}` uses are in the `ImageResponse` generators (apple-icon.tsx, opengraph-image.tsx). They existed before this round and are not rendered into the HTML page, so the claim holds. Only the citation is loose.
- **N-R6-3 (info, CHANGELOG):** CHANGELOG.md has no 0.3.x sections, and its older section is still headed "0.1.0 (unreleased)". VERSION.md records those releases, so this is a cosmetic gap in the history.

Counts: 0 blocking, 1 low, 2 info.

## Overall

**PASS.** C-10 is a clean version-only release at 0.4.0, and INV-34 and the tag are correct. RB-6 and C-9 have no blocking defects.
