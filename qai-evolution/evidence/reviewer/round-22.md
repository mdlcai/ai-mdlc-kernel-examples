Commits reviewed: bfb538e0703ad4fac158c2c1c3185a5df87ddd4a (C-27, Release 0.8.0: Re-Entry R6, Evolution evidence protocol), ade705e0c34c8d8b3730ca86cd617cb27518003e (RB-22, Round 21 follow-ups: N-R21-1, N-R21-2), d17614efd5b76563d2eea84f26d968146422ecb7 (RB-23, W12 e2e scopes the 429 banner to the protocol form)
Previous round reviewed: round 21 (RB-21 PASS with minors; F-31, F-32, F-33 and F-34 PASS, carried or given RB-21; no FAIL)

# Reviewer Gate: Review Round 22 (C-27 Release 0.8.0, with RB-22 and RB-23)

- **Mechanism:** a fresh-context subagent read `artifacts/reviewer/round-21.md`, the verdict tables and findings of round-19.md, and these diffs: `git show bfb538e`, `git show 177e197` (the 0.7.0 release, used as the pattern), `git diff 177e197 bfb538e -- compose.yaml package-lock.json`, `git show ade705e -- SPEC.md DECISIONS.md` and `git show d17614e`. It read the commit messages for `git log 177e197..bfb538e` (C-26, F-31..F-34, RB-20..RB-23). It checked the code against the docs: services/api/qai_api/services/jobs.py `check_protocol_slot` (:134-146) and `requeue` (:320-355); repositories/jobs.py `get_job(lock=True)` (:72-75, `with_for_update`); packages/qai-sdk/qai/protocol.py `GOVERNANCE_MIN_EPSILON` (:39), `eps` (:391) and `epsilonGate` (:434); SPEC.md :119, :161, :245, :374, :756 and :765; DECISIONS.md ADR-49 and ADR-50 (:444-463); apps/web/app/app/protocol/ProtocolForm.tsx (form at :136, banner at :207, `</form>` at :213); components/ui/ErrorBanner.tsx :44 and Field.tsx :52. It ran:
  - `git grep "0.7.0"` and `git grep "0.8.0"` over bfb538e, excluding CHANGELOG, VERSION and artifacts.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (**32 pass, 1 fail**), 1 manual. The one fail is INV-2 (web baseline), the known carry. INV-34 passes: "0.8.0: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval 0; pending review **C-27**; in rework none; blocked none.

  It did not run smoke, `npm run e2e`, `npm run test:sdk`, the API suite, web lint or the web baseline. The build agent reports smoke 29/29, e2e 50/50 (W12 included), SDK 519 passed and 9 skipped, and web lint clean on the rebuilt 0.7.0/HEAD stack; a 0.8.0 rebuild and smoke were running at the same time and are not covered here. It did not touch Docker, made no request to https://localhost:8443, did not read `.env` or FRONTEND-AUDIT.json, and ran govern.mjs only as `queue`. It created only this file and staged and committed nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| C-27 | bfb538e Release 0.8.0 | **PASS with notes** (0 blockers, 0 major, 0 minor, 2 notes) | The version bump is the same 12 files and the same 15 lines as 177e197, and every one moves 0.7.0 → 0.8.0. No other manifest, lockfile or compose line changed. INV-34 passes. The CHANGELOG and VERSION row cover the right commits. Two CHANGELOG phrases are imprecise (N-R22-2, N-R22-3). |
| RB-22 | ade705e Round 21 follow-ups | **PASS with minors** (0 blockers, 0 major, 1 minor) | N-R21-1 is closed: SPEC :245 lists `epsilonGate`, which matches protocol.py:434. N-R21-2 is closed in substance: SPEC :161, F-33 :756 and the AC :765 name requeue. But they say "a requeue of a dead protocol job", while the code (and SPEC :119) allow failed or dead (N-R22-1). |
| RB-23 | d17614e W12 locator | **PASS** | The scoped locator is sound. The banner renders inside the "Evidence protocol" form, and the scope excludes Next's route announcer. |

## Check by check

| Check | Result | Evidence |
|---|---|---|
| C-27: same files as the 0.7.0 release | PASS | Both commits touch the same 12 files: .env.example, CHANGELOG.md, VERSION.md, apps/web/lib/env.ts, apps/web/package.json, compose.yaml (4 lines), package-lock.json (3), package.json, packages/qai-sdk/pyproject.toml, qai/__init__.py, services/api/pyproject.toml and qai_api/config.py. Outside the CHANGELOG and VERSION, each hunk is just the version string. |
| C-27: nothing else in the manifests, lockfile or compose | PASS | `git diff 177e197 bfb538e -- compose.yaml` has one non-version line, `PROTOCOL_TIMEOUT_SEC: ${PROTOCOL_TIMEOUT_SEC:-900}`, and that came from F-33 (7931aff), not C-27. The lockfile diff across the release is only the three version fields. No dependency changed. |
| C-27: no stale 0.7.0 | PASS | `git grep 0.7.0` at bfb538e finds only DECISIONS.md:410 (ADR-48 "Releases 0.7.0", historical, correct). The `0.8.0` hits in package-lock.json :6000 and onwards are third-party `engines`/peer ranges and have nothing to do with the bump. |
| C-27: INV-34 | PASS | version-agreement over 10 files agrees on 0.8.0. |
| C-27: VERSION row | PASS | `0.8.0 | 2026-11-08 | Re-Entry R6 | ... F-31 ... F-32 ... F-33 (migration 0013), F-34 (ADR-49); RB-20; RB-21 (ADR-50); RB-22; RB-23`. It matches the format of the 0.7.0 row, the commit date and the feature commits. ADR-49 says "Releases 0.8.0" (DECISIONS.md:444). |
| C-27: CHANGELOG covers the log | PASS | `git log 177e197..bfb538e` contains (besides governance records) RB-18 and RB-19, then C-26, F-31..F-34 and RB-20..RB-23. RB-18 and RB-19 are round 16/17 follow-ups that landed after 0.7.0 was tagged and were not in the 0.7.0 section. The 0.8.0 section does not list them either. See the note under N-R22-3. |
| C-27: CHANGELOG F-32 | PASS | The six tests, up to 30 seeded runs with distributions and CIs, disjoint splits locked before test, the worse/invalid/missing injections, and the seed-2026 defaults passing all six match F-32's commit message and rounds 19-21 (round 21 ran `test_defaults_with_seed_2026_pass_all_six`). |
| C-27: CHANGELOG F-33 | PASS | Strict params, one protocol per tenant (429 QUEUE_FULL, Retry-After 60, on submit and requeue: jobs.py:327-329 → :139-146), `PROTOCOL_TIMEOUT_SEC` default 900, migration 0013 whose downgrade refuses while protocol rows exist, and releases may require `protocol`. All match the code and round 19. |
| C-27: CHANGELOG F-34, RB-21, RB-22, RB-23, C-26 | PASS | F-34: the form, Advanced, the panels, BoxPlot, vitest and W12. RB-21: ε_g = max(ε, 1e-9), and `epsilon` and `epsilonGate` are recorded (round 21). RB-22 and RB-23 match their diffs. C-26: SPEC F-31–F-34, W12 and ADR-49 (the commit also touched ARCHITECTURE 2.11 and the build plan; leaving those out is not wrong). |
| C-27: CHANGELOG F-31 | Imprecise | "adopted only when its J beats the checkpoint by ε". The rule is `V ∧ J ≤ J(Ψn) − ε` (SPEC :374), so at ε 0 a tie is accepted, and the gate first rejects missing evidence and invalid output. See N-R22-2. |
| C-27: CHANGELOG RB-20 | Imprecise | "under the job row lock". The slot check takes the per-kind advisory lock (`lock_kind_slot`, jobs.py:139). The row is also `FOR UPDATE`, but the RB-20 commit message says "advisory lock". The UI fix for N-R19-4 is not mentioned. See N-R22-3. |
| RB-22: SPEC :245 `epsilonGate` | PASS | The schema now reads `…, jCheckpoint, epsilon, epsilonGate, sha256After}]`. That is the set of keys protocol.py writes (:433-434), and it matches ADR-50. |
| RB-22: requeue 429 wording | PASS with a gap | SPEC :161 ("a requeue of a dead protocol job while another is queued or running"), F-33 :756 ("by submit or requeue") and the AC :765. The code checks the slot for any requeue of a protocol job, and requeue accepts `DEAD` **or** `FAILED` (jobs.py:323-324). SPEC :119 says "409 unless dead or failed". So ":161 and :765 say dead only" is narrower than the code. See N-R22-1. |
| RB-22: ADR-50 note | PASS | DECISIONS.md adds one consequence line that records both changes. It is docs only and no code changed. |
| RB-23: locator | PASS | `page.getByRole('form', { name: 'Evidence protocol' }).getByRole('alert')`. The `<form aria-label="Evidence protocol">` (ProtocolForm.tsx:136) gets the implicit `form` role because it has an accessible name. `SubmitBanner` → `ErrorBanner` (`role="alert"`, ErrorBanner.tsx:44) renders at :207, inside the form, which closes at :213. Next's `__next-route-announcer__` is outside the form, so strict mode is satisfied. `Field` errors are also `role="alert"` (Field.tsx:52) inside the form. In this flow the params pass client validation, so no field error renders when the 429 arrives. A later change that shows a field error together with the banner would re-trigger strict mode, but that is not a defect today. No other e2e spec uses a bare `getByRole('alert')`. |
| Invariants and queue | PASS | INV-2 only (the carry). The queue holds C-27 for review. |

## Findings

- **N-R22-1 (minor, RB-22 / F-33): SPEC :161 and the AC :765 say "a requeue of a dead protocol job", but requeue also accepts failed jobs.**
  - Location: SPEC.md:161 and :765. The code is services/api/qai_api/services/jobs.py:323-327: `if job.status not in (DEAD, FAILED)` → 409, then `check_protocol_slot` for any protocol job. SPEC :119 and the RB-20 commit say "failed or dead".
  - Impact: a reader of §3 or the AC could think a failed protocol job can be requeued past the slot. The code is right. The two SPEC lines are narrower than §4 and the code.
  - Fix: change "a dead protocol job" to "a failed or dead protocol job" at :161 and :765. Docs only.
- **N-R22-2 (note, C-27): the CHANGELOG F-31 line says "beats the checkpoint by ε".**
  - The rule is J ≤ J(Ψn) − ε, so at ε 0 a tie is adopted (round 21 confirmed the search gate keeps this). The line also leaves out that missing evidence and invalid output are rejected first. A suggested wording: "adopted only when its evidence is complete, V holds and J ≤ J(Ψn) − ε".
- **N-R22-3 (note, C-27): the CHANGELOG RB-20 line names the wrong lock and leaves out the UI fix.**
  - "under the job row lock" should be "under the per-kind advisory lock" (the job row is also locked, but the advisory lock is what makes the slot count race-safe). "random-search and governance-run fixes" leaves out N-R19-4 (Advanced opens on a field error). RB-18 and RB-19 (post-0.7.0 follow-ups to rounds 16 and 17) are in the 0.8.0 range but in neither CHANGELOG section. Add them if the CHANGELOG is meant to be complete; otherwise this is accepted.

Counts: 0 blockers, 0 major, 1 minor, 2 notes.

## Carried forward (human action)

- **Queue:** C-27 (GD-0163) is pending review. This round is its review evidence. N-R22-1..3 are doc wording and can go in a later RB commit. They do not block the release.
- **Still open from earlier rounds:** N-R20-3 (governance gates against the last accepted checkpoint, not Ψ\*; SPEC :732), N-R20-4 (the Advanced re-open edge case), N-R21-3 (the ε_g reason string), and N-R19-5, -6, -7 and -8. All are notes.
- **INV-2:** still fails (web baseline stale). It needs a quiet-host baseline run or an explicit human acceptance.
- **0.8.0 stack:** the rebuild and smoke at 0.8.0 were running at the same time as this review and are not covered here. Confirm `/api/health` reports 0.8.0 and smoke passes before tagging.

## Not run

Smoke, `npm run e2e`, `npm run test:sdk`, `npm run test:integration`, the web vitest, web lint, `npm run lint:py` and `npm run web:baseline`. The build agent reports smoke 29/29, e2e 50/50, SDK 519 passed and 9 skipped, and web lint clean on the 0.7.0/HEAD stack. C-27 changes only version strings and docs, RB-22 only docs, and RB-23 only one e2e locator, so the reported runs are enough for this round. No Docker and no request to localhost:8443.

## Overall

- C-27 is a clean, by-the-pattern bump. INV-34 agrees, and there is no stale version and no stray manifest change. The CHANGELOG and VERSION row are accurate apart from two wording points.
- RB-22 closes N-R21-1 exactly and N-R21-2 in substance. One word ("dead" → "failed or dead") is still to fix.
- RB-23 is the right fix for the strict-mode failure.

Verdict: PASS with notes for C-27. PASS with minors for RB-22. PASS for RB-23. No commit fails.
