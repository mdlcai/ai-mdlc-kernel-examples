# Security Audit — pass 3 (re-scan)

Commit scanned: d426eab (RB-5; code tree after RB-3, RB-4 and RB-5). Same categories and commands as pass 1.
Raw output: `artifacts/security/pass-3/`.

| Category | Tool | Exit | Result | vs pass 1 |
|---|---|---|---|---|
| Dependencies (web, root) | `npm audit --omit=dev` | 0 / 0 | 0 vulnerabilities | same |
| Dependencies (api) | `bundle-audit check --update` (advisory DB 2026-10-17) | 0 | No vulnerabilities found | same |
| SAST (Rails) | Brakeman 8.1.0 | 0 | 0 warnings | same |
| SAST (web + api) | Semgrep 1.95.0 `p/default`, 297 rules / 298 files | 0 | 8 findings, all false positives | +1: `tainted-sql-string` at `concerns/pagination.rb:15` |
| Secrets | gitleaks 8.21.2, 74 commits | 1 | 6 findings, the same false positives as pass 1 (DESIGN-TEMPLATE.html example keys ×3, docs example key, Stripe contract-test fixtures ×2) | same |

## New Semgrep finding

- `ruby.rails.security.injection.tainted-sql-string` at `api/app/controllers/concerns/pagination.rb:15`:
  `relation.where(relation.arel_table[:id].lt(anchor.id))`.
- **Verdict: false positive.** It is an Arel predicate on the integer id of a record already loaded through the tenant scope. Arel binds the value, and no string reaches SQL.
- The file has not changed since F-01 (4f81f5c). Pass 1 ran Semgrep under a 2-CPU memory cap and only partly parsed some files, which is the likely reason it did not report this.

## Remediated findings re-checked

| ID | Re-check | Result |
|---|---|---|
| SA-1 | The throttles match the router's path and Rails' parsed params: query string, form body and every JSON alias. Specs cover the path spellings, form bodies, `?email=` and a `text/x-json` body. Reviewer round 2 N2 and N5 were fixed in RB-4 and RB-5, and passed re-verify 2. | Closed |
| SA-2 | Each attempt has a 5 s overall deadline, and slow endpoints are deferred for the rest of the sweep. Specs pass. | Closed. Residual LOW (Reviewer N3): an endpoint with ≥ 100 due rows can still fill one sweep batch |
| SA-3 | `safeNext` refuses control characters, backslashes, other origins and any parsed path starting with `//` (Reviewer N1, RB-4). The Reviewer fuzzed about 1.5 million inputs and none left the origin. | Closed |
| SA-4 | Deferred, LOW (the 72-byte bcrypt input). Unchanged. | Deferred |
| SA-5 | IPv6 transition ranges are in the block list. Specs pass. | Closed |
| SA-6 | A locked account pays the dummy bcrypt compare. Spec passes. | Closed |

**New findings:** CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 0.
**Verdict: PASS.** No CRITICAL, HIGH or MEDIUM finding is open. One LOW is deferred with its reason (SA-4), and one LOW is a residual (Reviewer N3).
