Commit reviewed: 901ed2a
Previous round reviewed: d426eab
Inputs: artifacts/reviewer/round-2.md, git diff d426eab..901ed2a, SPEC.md, ARCHITECTURE.md §9, SECURITY-AUDIT.md, artifacts/security/pass-3.md
Mechanism: Task tool general-purpose subagent
Focus items: GD-0036 (RB-6)

# Reviewer Gate — focus check (GD-0036)

The review was read-only. No suites, builds or docker were run. Scope: `git diff d426eab 901ed2a`. It changes `api/spec/services/refunds/create_spec.rb` (+3/-1) and adds `SECURITY-AUDIT.md` (+124, now tracked: `git ls-tree 901ed2a` has blob 91ce933). It also changes `GOVERNANCE-DECISIONS.jsonl` (+1, the GD-0035 record for RB-5), which the brief did not list. That file is a governance append only.

## Checklist

| # | Item | Result | Rationale |
|---|---|---|---|
| 1 | Spec tests the SPEC behaviour through the production path, not weakened | PASS | `create_spec.rb:79-80` now asserts that `Problem#code == "NOT_FOUND"` (which `problem.rb:6` maps to 404) and that `Refund.count == 0`. It calls the real `Refunds::Create.call!`, which looks the intent up through `PaymentIntent.for_tenant` and raises NOT_FOUND before any transaction (`refunds/create.rb:11-13`). This matches SPEC SEC-5 (SPEC.md:180) and the §7 `NOT_FOUND` row (SPEC.md:527). It is stricter than the old field-error assertion, and the HTTP 404 is also covered at `spec/requests/refunds_spec.rb:125-126`. |
| 2 | No production code changed, and nothing else in the diff adds problems | PASS | The diff touches no file under `api/app`, `web/` or config (`git diff --stat`). The only other file is an append to `GOVERNANCE-DECISIONS.jsonl` (GD-0035, commit d426eab). |
| 3 | SECURITY-AUDIT.md internally consistent | PASS | 1. A grep for `RESCAN`, `TODO`, `TBD`, `FIXME` and `XXX` finds nothing, so round-2 item 7's `RESCAN_BODY` placeholder is gone (round-2.md:17). 2. Pass 3 names commit d426eab and verdict PASS (SECURITY-AUDIT.md §3 and header table). Both agree with pass-3.md:3 and its closing verdict. 3. The raw outputs agree: `exits.txt` (bundle-audit 0, brakeman 0, semgrep 0, gitleaks 1), npm 0 vulnerabilities ×2, bundle-audit "No vulnerabilities found", Brakeman "Security Warnings: 0", semgrep "Ran 297 rules on 298 files: 8 findings" with `pagination.rb` flagged, gitleaks "74 commits scanned … leaks found: 6". 4. The counts are right: the §2 table has 1 HIGH (SA-1), 2 MEDIUM (SA-2, SA-3) and 3 LOW (SA-4, SA-5, SA-6), which matches "HIGH 1 · MEDIUM 2 · LOW 3". "Remediated 5, deferred 1" matches. The open count of CRITICAL/HIGH/MEDIUM 0 with SA-4 deferred and N3 residual matches pass-3.md. |
| 4 | Anything else wrong in the diff | PASS (advisories) | No blocking defect. The advisories A1–A4 are below. |

## Findings

### A1 — LOW (advisory) — Reviewer round-2 N4 is not recorded as a residual

SECURITY-AUDIT.md §3 lists Reviewer N3 as the only residual. Its §5 "Idempotent retries" says the retry "resumes the same payment … under the same processor idempotency key".

Round 2 also raised N4 (round-2.md:73-78), a LOW advisory about money-path resume edges:

- Failover resume re-runs the chain from attempt 1, so it can double-charge when an earlier account errored transiently.
- The 60 s stale window is shorter than the Stripe read timeout.
- A `/confirm` retry after a commit answers 409.

The code is unchanged. `payments/confirm.rb:60-63` still derives `first_attempt_no` from the persisted attempts, and those are rolled back on a step-3 failure. N4 should be listed as a known residual, or its disposition stated. It is outside this diff's code scope and does not block.

### A2 — LOW (advisory) — the Semgrep explanation is speculative

SECURITY-AUDIT.md §3 and pass-3.md say pass 1 "likely" missed `pagination.rb:15` because some files were only partly parsed under the memory cap. The pass-3 raw output says the same thing ("Some files were skipped or only partially analyzed", `semgrep.txt` scan summary), even though it exits 0. The wording is hedged, so this is acceptable. Pass 3 is still not a complete-parse run.

### A3 — INFO — wording in the pass table and §3

The pass-2 row's commit cell reads "4aa49b4, 9335abe, RB-5". It should read d426eab for consistency.

§3 says "Reviewer round 2 found three gaps". N5 was found on the round-2 re-verify, so the wording is slightly loose.

### A4 — INFO — the spec title promises more than it tests

The example is titled "does not find another tenant's or mode's payment" (`create_spec.rb:74`), but it only exercises another tenant. A live-mode intent of the same merchant is not exercised. This is pre-existing and not a weakening.

Verdict: PASS
