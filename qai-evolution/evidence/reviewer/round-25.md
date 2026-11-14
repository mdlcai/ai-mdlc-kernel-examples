Commits reviewed: a88f071 (F-36, arrow notation on the map result line), e45a5d3 (C-34, SPEC §3 / §5 F-27, ADR-52)
Previous round reviewed: round 24 (RB-25 PASS)

# Reviewer Gate: Review Round 25 (F-36)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show a88f071` and `git show e45a5d3`, ran test_expr.py (115 passed), the web expr/form tests (116 passed) and `npm run lint:invariants` (34 total / 33 machine: 32 pass, 1 not passing: INV-2, the known web-baseline carry). It probed 62 hand-written edge inputs against both mirrors, fuzzed 42,196 sources old-SDK vs new-SDK (0 regressions; the 522 changed results all contain an arrow and were errors before) and new-SDK vs TS mirror across three label sets (126,588 parses, 0 mismatches). It edited no repo file.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-34 | e45a5d3 | OK (no drift) | Doc changed first, reason in ADR-52; no endpoint, limit, error code or §6 surface change. |
| F-36 | a88f071 | **PASS** (0 blockers, 0 major, 3 low notes) | INV-10 holds (fixed prefix regex plus set lookup, no eval). SDK and TS agree on all 100 vectors; every existing vector's message, canonical, nodes and value unchanged. The separator must match the left-hand side (`Ψₙ₊₁ →`, `Ψₙ =` rejected in both mirrors). |

## Findings

- N-R25-1 (low, pre-existing): Python and JS `\s` disagree on `\x85`, `\x1c`-`\x1f` and `﻿`; the gap predates F-36 (same as N-R23-5). Safe: the SDK has the final say and the outcome is an error or the same canonical form. Carried with N-R23-5.
- N-R25-2 (low): the Map tab help text (apps/web/app/app/formula/FormulaForm.tsx:81) shows only `Ψₙ₊₁ = f(...)`; SPEC does not require the arrow form there. Accepted; optional follow-up.
- N-R25-3 (low, docs): ADR-51 (DECISIONS.md:472) describes labels as names followed by `=` without an "amended by ADR-52" pointer; ADR-52's Consequences records the change. Accepted.
