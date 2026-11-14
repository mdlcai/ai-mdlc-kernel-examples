Commits reviewed: f8480c7 (F-37, square brackets: index [n] and grouping), df6e771 (C-37, SPEC §3 / §5 F-27, ADR-53)
Previous round reviewed: round 25 (F-36 PASS)

# Reviewer Gate: Review Round 26 (F-37)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show f8480c7` and `git show df6e771`, ran test_expr.py (132 passed; 198 with `-k "formula or expr"`), the web expr/form tests (132 passed) and `npm run lint:invariants` (34 total / 33 machine: 32 pass, 1 not passing: INV-2, the known web-baseline carry). It fuzzed 40,097 sources (about 100 hand-written bracket, index, label and depth cases) against 6 variable and label sets: 240,582 parses per implementation. It found 0 SDK/TS mismatches at HEAD and at the parent, and 0 parent-vs-HEAD regressions (every changed result was an error before). Adversarial 2,000-character inputs ran in at most about 20 ms. It edited no repo file.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-37 | df6e771 | OK (no drift) | Doc changed first, reason in ADR-53; no endpoint, limit, error code or §6 surface change. |
| F-37 | f8480c7 | **PASS** (0 blockers, 0 major, 4 low notes) | INV-10 holds and "no indexing" holds: `[n]` is dropped in the lexer, `[ ]` returns the inner node, other indexes error. The mirrors agree exactly; `x[0]` is the only expectation change; the ADR-52 separator rule holds; label keys are built identically; the regexes are linear. |

## Findings

- N-R26-1 (low): a doubled index such as `Ψₙ₊₁ₙ =` or `Ψₙ₊₁[n] =` is accepted as the next-state label, because the trailing `ₙ`/`[n]` is dropped after `ₙ₊₁`. SPEC's "may be added or left out" allows it; both mirrors agree; harmless. Accepted.
- N-R26-2 (low): `[n]` is also dropped from function and constant names (`sin[n](x)`, `pi[n]`), matching ADR-51's `sinₙ`. Accepted.
- N-R26-3 (low, latent): a declared label whose left-hand side normalises to empty would match a bare `ₙ`; no production label set can produce one. Accepted as a note for future `result_labels` callers.
- N-R26-4 (low, coverage): 18 vectors' `labels` input widened to the new MAP_LABELS with expectations unchanged; the old tuple is covered by the reviewer's fuzz. Accepted.
- Carried: N-R25-1 / N-R23-5 (Python vs JS `\s` on `\x85`, `\x1c`-`\x1f`, U+FEFF); the SDK's parse decides.
