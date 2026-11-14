Commits reviewed: 07989fb (RB-25, Round 23 follow-ups: N-R23-3, N-R23-4)
Previous round reviewed: round 23 (F-35 PASS with notes)

# Reviewer Gate: Review Round 24 (RB-25)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show 07989fb`, ran test_expr.py (104 passed) and the web expr/form tests (106 passed; the four new vectors pass by name in both), and `npm run lint:invariants` (34 total / 33 machine: 32 pass, 1 not passing: INV-2, the known web-baseline carry). It edited no file.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-25 | 07989fb | **PASS** (0 blockers, 0 major, 1 low note) | N-R23-3: four reject vectors added, with identical SDK and TS messages; no existing vector changed. N-R23-4: the JSDoc is back on `variablesFor`; comment-only move. No behaviour changed. |

## Findings

- N-R24-1 (low): of the four new vectors, `constructor` and `toStringₙ + Ψ` reach the name lookup ("unknown name"); `__proto__ = 1\nΨ` and `hasOwnProperty = Ψ` are rejected structurally first. Parity still holds and the reviewer found no prototype leak (aliases are a Map, functions and constants use hasOwnProperty.call). Accepted: two lookup vectors cover the claim.
