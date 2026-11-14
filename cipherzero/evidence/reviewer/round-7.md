# Reviewer Gate — round 7 (re-entry R-3: F-25..F-28)
Commit reviewed: 393294e36221a24f3581b22791004c696043a21f
Previous round reviewed: 6ec898373e634df5722086a89382e3e8a0e39f1d
Mechanism: Claude Code Agent tool, general-purpose subagent (fresh context), read-only except this file
Verdict: FAIL

Inputs:
- artifacts/reviewer/round-6.md (format reference only; its verdicts were not read)
- RESEARCH.md, ARCHITECTURE.md (§5.10, §9), SPEC.md (§2, §3, §5, §6, §10, §11 R-3), invariants.json, CLAUDE.md
- `git diff 6ec8983..393294e` and the working tree at HEAD: apps/prober/**, apps/scanner/src/tls.ts,
  apps/worker/src/{jobs/domain-verify,jobs/external-discovery,prober-client,main,runner,scheduler}.ts,
  apps/api/src/{services/quantum-domains,services/quantum-ingest,services/quantum,routes/quantum,routes/quantum-ingest}.ts,
  packages/core/src/{quantum/domain-name,quantum/ingest,plan,entitlements,queue}.ts, packages/core/prisma/migrations/2026111{3,4}*,
  apps/web/src/app/(console)/quantum/{domains-panel,quantum-client}.tsx, compose.yaml, Dockerfile, deploy/pki/bootstrap.mjs,
  README.md, QUICKSTART.md, and the R-3 tests (apps/prober/test/*, apps/worker/test/{domain-verify,external-discovery}.test.ts,
  apps/api/test/{quantum-domains,quantum-ingest}.test.ts, packages/core/test/{domain-name,quantum-ingest}.test.ts, e2e/w8-external.spec.ts)
- Lint line (given): `60 total / 59 machine (59 pass, 0 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- FRONTEND-AUDIT.summary.json (gitSha ea0336b, freshness current; only failures are Lighthouse performance for landing 66 and dashboard 72, which were accepted)
- Verification Gate verdict line (given): npm test 718/718 in 58 files; typecheck, lint and lint:invariants pass; smoke --fixtures 9/9;
  e2e w8/w7/w5 pass on desktop-chrome and mobile-pixel7; web:baseline --full INV-2 pass. Coverage was not measured.
- No suites were re-run (per the brief). Only targeted read-only commands.

## Checklist (BUILD.md Reviewer Gate)
| # | Item | Verdict | Rationale (file:line) |
|---|---|---|---|
| 1 | RESEARCH requirements implemented, not stubbed | PASS | Every R-3 requirement is real code: the isolated prober with an IANA BlockList guard (apps/prober/src/egress.ts:20-47, 191-224), TXT verification (apps/worker/src/jobs/domain-verify.ts; packages/core/src/quantum/domain-name.ts:93-108), a CT parser with caps (apps/prober/src/ct.ts), the shared ingest (packages/core/src/quantum/ingest.ts:115-119), the domains limit Free 1 / Team 10 / Enterprise unlimited (packages/core/src/entitlements.ts), and egress addresses documented (README.md:80-93, QUICKSTART.md:81-83). |
| 2 | SPEC features have behavior tests | PASS | The §11 R-3 list is covered through production paths. Verify and lapse go through runDomainVerifyForOrg (apps/worker/test/domain-verify.test.ts:82,111). Discovery, probe and ingest go through runExternalDiscoveryForOrg (apps/worker/test/external-discovery.test.ts:119,174,197,234,262). The API has 409/429/503/limit/roles tests (apps/api/test/quantum-domains.test.ts:85,118,185,221,236). The guard has rebinding and port-before-DNS tests (apps/prober/test/routes.test.ts), and mTLS has its refusal cases (apps/prober/test/mtls.test.ts). |
| 3 | Architecture matches ARCHITECTURE.md | PASS | The prober is its own app with no DB or Redis environment (apps/prober/src/env.ts). The worker reaches it over HTTPS mTLS with a SPIFFE check (apps/worker/src/prober-client.ts). The ingest lives in core and is shared by the API and the worker (apps/api/src/services/quantum-ingest.ts:19-33). The networks are probe-ctl (internal) and egress (compose.yaml:197), as ARCH §5.10 requires. |
| 4 | Scope discipline (nothing outside SPEC) | PASS | PATCH ports, the hosts list and add, and the scans `origin` filter are all SPEC §3 endpoints (apps/api/src/routes/quantum.ts; apps/api/src/services/quantum-ingest.ts:48-56). No unrequested surface was found. |
| 5 | Security | FAIL | The internet-facing prober mounts the whole shared PKI volume (compose.yaml:196 `volumes: [pki:/pki:ro]`). That volume holds ca.key and every service's private key (deploy/pki/bootstrap.mjs:46-47, 69-107), all owned by uid node with mode 0600. The prober also runs as `node` (Dockerfile:86, tools stage Dockerfile:31-32), so it can read the CA signing key, worker.key and trust-core.key. This contradicts RESEARCH R-3 ("holds no secrets worth taking") and the ARCH §5.10 isolation intent. See Finding 1. Everything else in the security review was sound (egress guard, port check before DNS, connect-to-checked-address with SNI at apps/scanner/src/tls.ts:61-64, CT `redirect:'error'`, TLS 1.3 mTLS with a SPIFFE allowlist, PROBER_ALLOW_CIDRS refused in production, INV-57 guard before DB). |
| 6 | Cold read: no obvious bugs | PASS | Guarded updates `{id, tenantId, status: d.status}` hold up under races (apps/worker/test/domain-verify.test.ts:198). The miss counter counts once per UTC day. Discovery re-checks the domain every 25 probes. Non-blocking note: RUN_LOCK_MS = 2 h in apps/worker/src/jobs/external-discovery.ts, but a worst-case run (1,000 hosts × up to 80 s across 2 lanes) can take longer than that, so two runs could overlap. Idempotent scanId `ext:<id>:<window>` and the guarded ingest limit the damage. |
| 7 | Runnable as documented | PASS | The prober image stage and compose service are present (Dockerfile:79-90, compose.yaml:176-206), README and QUICKSTART are updated, and the given Verification Gate evidence shows the stack and e2e W8 green. |
| 8 | Manual invariants (one verdict each) | PASS | INV-11 "Every scheduled trigger has an effect test": PASS. R-3 adds two cron schedules (packages/core/src/queue.ts SCHEDULES) and each has effect tests for every trigger. external-discovery: cron at external-discovery.test.ts:300, startup catch-up at :310, manual at :174/:197. domain-verify: cron with effect and window claim at domain-verify.test.ts:82-108, manual at :181. Startup for domain-verify runs the same catchUp → runJob path (apps/worker/src/scheduler.ts:30-38). The original jobs keep their effect tests in apps/worker/test/jobs.test.ts, which the delta does not touch (e.g. :294 startup). |
| 9 | Nothing declared is unreachable | PASS | Every admitted state has a production writer and a test through that writer. Domain states (migration 20261113000000 CHECK): verified and lapsed are written by apps/worker/src/jobs/domain-verify.ts (applyVerifyResult plus the guarded updateMany), tested at domain-verify.test.ts:82,111 (lapsed → verified at :129). removed is written by apps/api/src/services/quantum-domains.ts:220-237, tested at quantum-domains.test.ts:85,344. removed → pending is written at quantum-domains.ts:127-128, tested at quantum-domains.test.ts:97. Host statuses (migration 20261114000000 CHECK): pending, wildcard and the apex/ct origins are written by discoverHosts, and the manual origin by addHost. reachable, unreachable and private are written by the probe pool, tested at external-discovery.test.ts:119,262 and quantum-domains.test.ts:286. Scan origin `external` is written by ingestFindings from the worker, tested at external-discovery.test.ts:119. Config EXTERNAL_DISCOVERY_ENABLED, PROBER_* and CT_SEARCH_URL are all read and tested (domain-verify.test.ts:171; prober env.test.ts). |

Counts: 8 PASS / 1 FAIL / 0 UNCERTAIN.

## Focus items
| ID | Verdict | Rationale (file:line) |
|---|---|---|
| F-25 prober | FAIL | The guard, mTLS, SNI-to-checked-address and container hardening are correct (apps/prober/src/egress.ts:111-224, routes.ts:137-170, server.ts; compose.yaml:197-202). However, the deployment gives the egress-facing container read access to ca.key and every other service's private key (compose.yaml:196; deploy/pki/bootstrap.mjs:46-107; Dockerfile:86). See Finding 1. |
| F-26 domains and TXT verification | PASS | Name normalization refuses IP literals, wildcards, public suffixes and reserved TLDs (packages/core/src/quantum/domain-name.ts). The state machine and miss-per-UTC-day rule are tested through the job (domain-verify.test.ts:82-131). The domains limit is enforced inside the creating transaction (quantum-domains.ts:96-144; test :85). A prober outage gives 503 and sets PROBER_DOWN_KEY (domain-verify.test.ts:142; quantum-domains.test.ts:195). |
| F-27 external discovery, CT and shared ingest | PASS | INV-56 assertDomainVerified (external-discovery.ts:140-147). The label-boundary filter, wildcard-not-probed rule, cap and private-never-reprobed rule are tested (external-discovery.test.ts:119,174,234). The INV-57 secret guard runs before the first tx (packages/core/src/quantum/ingest.ts:115-119). Fingerprint merging with cz-scan and plan validation by an external re-scan are tested (apps/api/test/quantum-ingest.test.ts:422). The DB CHECK ties origin to exactly one of workload or domain (migration 20261114000000). |
| F-28 console Domains tab and W8 e2e | PASS | The tab has add, TXT name and value, Verify / Scan now, Ports, Remove, the pending and lapsed banners, the hosts list and add, the private-host banner with the cz-scan pointer, and 409/429/503 handling. PLAN_LIMIT falls through to the shell's plan-limit state (apps/web/src/app/(console)/quantum/domains-panel.tsx:590-612; apps/web/src/components/shell/ConsoleShell.tsx:351). Origin column in scan reports. FRONTEND-AUDIT.summary.json has every screen cell passing; the Lighthouse perf residuals on landing and dashboard were accepted and are outside this change. e2e W8 passed on both projects (given). |

## Findings
1. **FAIL: the prober can read the CA private key and every service's private key.**
   - Where: compose.yaml:196 (`volumes: [pki:/pki:ro]` on the prober), together with deploy/pki/bootstrap.mjs:46-47 and :69-107, Dockerfile:31-32 and Dockerfile:86.
   - What is wrong:
     - bootstrap.mjs writes `ca.key`, `trust-core.key`, `api.key`, `worker.key` and `prober.key` into the single `pki` volume.
     - The files are mode 0600, owned by the `node` uid that pki-init runs as.
     - The prober image also runs as `node`, so the 0600 mode gives no isolation.
     - The prober is the one component on the internet egress network that processes attacker-influenced input: CT JSON, remote TLS handshakes and DNS answers.
     - A bug there, such as RCE or arbitrary file read, therefore leaks the CA key and lets an attacker mint a `spiffe://…/sa/worker` or `…/sa/trust-core` certificate. That breaks every mTLS boundary, including trust-core's.
     - This contradicts RESEARCH R-3 "the prober holds no secrets worth taking" and the ARCH §5.10 isolation goal.
   - Fix:
     - Have pki-init write a prober-only directory, e.g. `/pki/prober/{ca.crt,prober.crt,prober.key}`, or a separate `pki-prober` volume containing only those three files.
     - Mount only that directory on the prober, either with `volume: { subpath: prober }` or the dedicated volume, at `/pki:ro`.
     - Keep `ca.key` out of every volume a runtime service mounts. Better still, keep it only in pki-init's own volume. Optionally run the prober under a distinct uid.
     - Add a structure test, e.g. in apps/prober/test/structure.test.ts or a compose test, that parses compose.yaml and fails if the prober mounts a volume that pki-init populates with `ca.key` or another service's key.
