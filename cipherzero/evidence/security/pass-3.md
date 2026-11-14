# Security Audit — pass 3 (re-scan)
Commit scanned: af52231a351c7ff3a3c182542dbf23509a8a4517
Previous passes: artifacts/security/pass-1.md (59bbee6), artifacts/security/pass-2.md (32b23e1)
Diff reviewed: `git diff --ignore-cr-at-eol 32b23e16167f8ae37162304f2df1276224026a59 af52231a351c7ff3a3c182542dbf23509a8a4517` (27 files; every server, web and e2e hunk read, plus smoke-test.log screened for tokens). The follow-on commit e8666d6 only appends GOVERNANCE-DECISIONS.jsonl. Tools ran against a `git archive af52231` snapshot and a bare clone, so the uncommitted working-tree changes are excluded.

Residual: CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 5 (deferred 4)

## Pass-1/Pass-2 findings — status
| ID | Severity | Status | Evidence (file:line / test) |
|---|---|---|---|
| SEC-1 | MEDIUM | FIXED | apps/api/src/services/workloads.ts:267 refuses issuance with a 409 for `kind === 'agent'`. apps/api/src/middleware/authenticate.ts:88 rejects an agent `czw_` with a 401. Neither file changed in the settle diff. Test: apps/api/test/workloads.test.ts (pass 2). |
| SEC-2 | LOW | DEFERRED | Unchanged: apps/api/src/services/auth.ts:203, :236 and apps/api/src/services/team.ts:340 still return 409 "already exists". The reason stands: the fix needs a SPEC §7 contract change, per-IP limits apply, and only account existence leaks. |
| SEC-3 | LOW | FIXED | apps/api/src/routes/auth.ts:61-62 requires `currentPassword`. apps/api/src/services/auth.ts:362-363 runs the limiter and then `verifyPassword` before the secret is stored. |
| SEC-4 | LOW | DEFERRED | Unchanged: apps/api/src/services/vault.ts:29 AAD still omits purpose (vault.ts is not in the diff). The reason stands: the fix needs an AAD version byte and a full re-seal (ADR scope), and exploitation needs DB write access. |
| SEC-5 | LOW | DEFERRED | Unchanged: apps/api/src/services/ops.ts:96-98 (not in the diff). The reason stands: retries hit only idempotent jobs, no data crosses tenants, and there is no platform-operator role. |
| SEC-6 | LOW | FIXED | packages/core/src/env.ts:50 `.refine(v => !/change[_-]?me/i.test(v))`. The compose-only `change-me` passwords for Postgres, the replica and Grafana are still deferred, because they bind only to loopback and `dev:secrets` generates them. |
| SEC-7 | LOW | DEFERRED | Unchanged: .github/workflows/ci.yml:41, :45, :113, :129, :147 are still on `@v4` tags. The reason stands: first-party actions, a `contents: read` token, no secrets and no `pull_request_target`. |
| SEC-8 | MEDIUM | FIXED | apps/web/src/app/(console)/settings/settings-client.tsx:254 is now `<form method="post" …>` around the `currentPassword` input at :256. A sweep of every `<form>` in apps/web/src found that all password-bearing forms post: settings :137 and :254, login LoginForm.tsx:58, register RegisterForm.tsx:66, invite AcceptInviteForm.tsx:124, and team team-client.tsx:279 and :462. The TOTP-code form at settings-client.tsx:223 still has no method; see SEC-11. |
| SEC-9 | MEDIUM | FIXED | apps/api/src/services/quantum.ts:411-431 adds `keyScopeBlocker`, called at :474 before a successor is created or any object moves. A blocked asset gets the step `policy_review_required`, which is not in `COMPLETE` (:386), so the plan cannot validate (:642-649). Test: apps/api/test/quantum.test.ts:346 (objects stay on key A, A stays active, advance returns 409, no `-pq` key is created; validates once the rule covers the successor). Bypass attempts are listed below. |
| SEC-10 | LOW | FIXED | apps/api/src/services/auth.ts:48-52 adds `reauthLimiter`: the per-IP+email bucket plus a per-user `reauth:${userId}` bucket (10 per 15 min, packages/core/src/ratelimit.ts:42). It is used in `mfaSetup` (:362) and `changePassword` (:425), and those are the only `verifyPassword` call sites besides login (:281). Spreading guesses across IPs is now capped at 10 per 15 minutes per account. There is no dedicated regression test (advisory). |

### SEC-9 bypass attempts
- **Empty `keyIds` (meaning all keys).** In a policy rule, `inList` treats `[]` or a missing list as "any key" (packages/core/src/policy/index.ts:69-70), so such a rule still matches the successor after migration. Nothing is lost and no block is needed, and `covers()` correctly returns false. Grants cannot be empty: routes/workloads.ts:42 requires `.min(1)`, and `grantScopeViolation` (vault.ts:109) treats any list as an allow-list, so the successor is out of scope and access fails closed.
- **Draft or pending bundle versus active bundle.** The blocker reads `status: 'active'`, the same row trust-core evaluates (apps/trust-core/src/service.ts:173), and activation supersedes the previous bundle atomically (service.ts:543-547). A key-A rule that is still pending when migration runs and is activated afterwards would not cover A-pq. That change has to pass two-person approval and signature, and the approver sees the key ids, so it is not a single-writer bypass. Informational: the policy editor could warn when a rule names a disabled key.
- **Successor already exists.** `existing?.id` is passed as `toKeyId`, and the block lifts only when the rule names both A and the existing A-pq. A writer who pre-creates a key named `A-pq` gains nothing, because the rule still has to name it through a signed, approved activation. An existing key with the wrong profile or status is still refused at :494.
- **Retry path (migrating → validated).** `advancePlan` calls `migrateAssets` on the retry at :640. Only `migrated` and `already_target` steps are carried forward, so the blocker re-runs for `policy_review_required`, `successor_created` and `failed` steps. A rule activated after a partial move therefore blocks the rest.
- **Grants.** The query filters on `status: 'active'`, `expiresAt > now` and `keyIds has A`. That matches authenticate's own validity checks; grant statuses are active, expired and revoked (worker grant-expiry.ts:18-24). The grant branch has no test (advisory).
- **Signing keys.** The blocker runs before the encrypt/sign branch, so a `purpose: 'sign'` key named in a rule is not retired either.
- **TOCTOU.** A policy activation or grant issuance that lands between the blocker check and `moveObjectsToKey` is a narrow race. A grant scoped to A fails closed afterwards. A rule activated in that window has passed dual approval. Informational only.
- **Other move paths.** `moveObjectsToKey` has a single caller (quantum.ts:505). Rewrap keeps the same key id.

## New findings in the diff

### SEC-11 — LOW — TOTP code form without `method="post"`
- Clause violated: BUILD.md Security Audit Gate, web delivery ("credential-bearing form without `method="post"`"); SPEC §4 credential handling. This predates the settle diff (pass 2 mentioned it as optional). It is recorded now because this pass checked every credential-bearing form.
- Location: apps/web/src/app/(console)/settings/settings-client.tsx:223 (`codeForm`, `<form className="stack form-narrow" onSubmit=… noValidate>` containing `name="code"` `autoComplete="one-time-code"`). It is rendered on initial load as the step-up "Verify" form whenever `user.mfaEnabled && !mfaFresh` (:283-288).
- Exploit path: If the user submits before hydration, the browser issues `GET /settings?code=NNNNNN`, which puts the TOTP code in the URL, history and any log that records query strings. The code is valid for about 90 s and was never consumed (the replay key is unset). Someone who reads that log within the window and also holds the victim's session cookie could gain `mfaFresh` step-up. Both the session cookie and a sub-2-minute window are required, hence LOW.
- Remediation: Add `method="post"` to the form at settings-client.tsx:223, which is one attribute. Add a lint or e2e check that every form containing `type="password"` or `autoComplete="one-time-code"` has `method="post"`.

No other new issues in the diff:
- **Access control and tenancy.** The `keyScopeBlocker` queries filter on `tenantId`. Transport and keyless assets now become `failed` instead of `operator_action_required` and still block validation; plan creation refuses them (ADR-065).
- **Injection.** No new raw SQL, `eval` or `child_process`.
- **Crypto.** Unchanged.
- **Secrets.** Gitleaks found nothing in the 2-commit range. The new e2e code uses the `GOOD_PASSWORD` fixture constant. smoke-test.log contains only PASS lines and no tokens.
- **Web.** The changes are styling (badge tones, `nowrap`, truncation, CSS), removal of `decoding="async"` and a `title={w.name}` attribute, which React escapes. There is no `dangerouslySetInnerHTML` or `href` sink, and security-headers.ts and proxy.ts are unchanged.
- **Unchanged areas.** packages/, compose.yaml, Dockerfile, deploy/, .github/ and package-lock.json did not change between 32b23e1 and af52231.

## Tool results
- **Secrets: Gitleaks v8.30.1 (`zricethezav/gitleaks:v8.30.1`).**
  - `git --redact --exit-code 1 --log-opts=af52231` on a bare clone (47 commits): exit 1, 3 hits.
  - `--log-opts=32b23e1..af52231` (2 commits): exit 0, no leaks.
  - `dir` on the af52231 snapshot: exit 1, 3 hits.
  - All 6 hits are the known `generic-api-key` test fixtures that pass 1 triaged: apps/api/test/team.test.ts (×2) and apps/api/test/auth.test.ts:61. Nothing new.
- **SAST: Semgrep 1.177.0 (`semgrep scan --config p/default --config p/typescript --config p/nodejsscan --metrics=off`).**
  - Exit 0, 38 results, 14 errors (2 partial parses: Dockerfile and packages/core/src/db/index.ts; 12 rule timeouts). JSON is at `artifacts/security/semgrep-pass3.json`.
  - No result is new compared with pass 2. The 17 fewer results are all in tooling scripts (scripts/govern.mjs, scripts/invariant-lint.mjs) where rules timed out this run, and none of them changed.
  - Timeouts on files touched or near the diff: apps/web/src/app/(console)/quantum/quantum-client.tsx (not in the diff), apps/api/src/routes/workloads.ts and packages/core/src/env.ts (both unchanged). The diff hunks were reviewed by hand.
  - The two touched-file results, `node_password` at settings-client.tsx:111-112, are state variable names, as pass 1 triaged.
- **Dependencies: `npm audit --omit=dev` (npm 10.9.3).** Exit 0, 0 vulnerabilities. Full `npm audit` reports 1 low, the same dev-only esbuild advisory GHSA-g7r4-m6w7-qqqr.
- **Dependencies and config: Trivy 0.74.0 (`fs --scanners vuln,misconfig,secret --severity HIGH,CRITICAL --exit-code 1`).** Exit 0: package-lock.json 0, Dockerfile 0.
- **License.** Not flagged by RESEARCH.md, so skipped.

## Deferred LOWs
- SEC-2, SEC-4, SEC-5 and SEC-7: deferred for the reasons in the table above, which are unchanged from pass 2.
- SEC-6 compose residual: deferred (loopback only, and `dev:secrets` generates the values).
- SEC-11 (new LOW): not deferred. It is a one-attribute fix, recommended for the next batch, and does not block.

## Verdict
**PASS WITH ADVISORY.** No CRITICAL, HIGH or MEDIUM finding is open. SEC-1, SEC-3, SEC-6, SEC-8, SEC-9 and SEC-10 are verified fixed in code. Advisories:
- SEC-11 is a LOW one-line fix.
- There are no regression tests for the SEC-10 per-user bucket or the SEC-9 grant branch.
- Four LOWs (SEC-2, SEC-4, SEC-5, SEC-7) and the SEC-6 compose residual remain deferred.
- Semgrep had rule timeouts, all on unchanged files.
