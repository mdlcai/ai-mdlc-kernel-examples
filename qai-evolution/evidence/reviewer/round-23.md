Commits reviewed: 1fa7892 (F-35, Formula Lab accepts Ψ, ψ, ₙ and the map's Ψₙ₊₁ = label), against 6aee29a (C-29, SPEC F-27 / §3 map and ADR-51)
Previous round reviewed: round 22 (C-27, RB-22, RB-23 PASS)

# Reviewer Gate: Review Round 23 (Re-Entry R7, F-35)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show 1fa7892` and `git show 6aee29a`, ran `npm run lint:invariants` (34 total / 33 machine: 32 pass, 1 not passing: INV-2, the known web-baseline carry), ran test_expr.py (100 passed) and the web expr/form tests (102 passed), and pushed 21 hostile and edge inputs through both parsers (the TypeScript mirror compiled into the scratchpad). It edited no repository file.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-35 | 1fa7892 Formula Lab reads Ψ notation | **PASS with notes** (0 blockers, 0 major, 4 low) | INV-10 holds: the name rule gains only `[Ψψ]` and an optional `ₙ`; labels are a fixed prefix match on the last line, declared by the caller; canonical output drops the label and stays ASCII; aliases are a Map, so `constructor` and `__proto__` cannot resolve through the prototype. SDK and TS agree on messages, columns and node counts for all 15 new vectors and 21 probes. Only sources that used to be rejected parse differently. |

## Findings

- N-R23-1 (low): `ₙ` is dropped from any name, reserved ones included (`piₙ + sinₙ(nₙ)` reads as `pi + sin(n)`). Both parsers agree and ADR-51's wording allows it. Accepted.
- N-R23-2 (low): in energy mode `E` is both a label and a valid definition name, so `E = psi^2\nE = E*2` parses. Unambiguous and previously an error. Accepted.
- N-R23-3 (low): no shared vector covers `constructor` / `__proto__`, though the commit message claims the protection (confirmed by hand). Follow-up RB-25.
- N-R23-4 (low): form.ts placed `MAP_LABELS` / `ENERGY_LABELS` between `variablesFor` and its JSDoc. Follow-up RB-25.
- N-R23-5 (medium, older than F-35, out of scope): Python's Unicode `\d` / `\s` differ from JavaScript's (Python accepts `٣ + psi`; `\x1c` vs `﻿` whitespace). Not caused by F-35; carried as a parity note for a later fix (for example `re.ASCII` and explicit whitespace classes in both parsers).
