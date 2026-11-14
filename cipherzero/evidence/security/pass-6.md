# Security Audit — pass 6 (re-entry R-3 delta: F-25, F-26, F-27, F-28)
Base commit: 6ec898373e634df5722086a89382e3e8a0e39f1d
Commit scanned: 393294e36221a24f3581b22791004c696043a21f
Mechanism: Claude Code Agent tool, general-purpose subagent (fresh context), read-only
Residual (pass 1): CRITICAL 0 · HIGH 0 · MEDIUM 3 · LOW 3 · INFO 3

## Scope reviewed

`git diff 6ec8983..HEAD`: 93 files, +7,640/−641. I read every source hunk in full: apps/prober/src/* (egress, env,
mtls, server, routes, ct, problem, semaphore, main), apps/scanner/src/tls.ts, apps/worker/src/prober-client.ts,
jobs/domain-verify.ts, jobs/external-discovery.ts, runner.ts, main.ts, apps/api/src/services/quantum-domains.ts,
routes/quantum.ts, packages/core/src/quantum/{domain-name,ingest}.ts, env.ts, plan.ts, entitlements.ts, queue.ts,
redis.ts, the two migrations, compose.yaml, Dockerfile, deploy/pki/bootstrap.mjs, playwright.config.ts,
e2e/helpers/dns.ts and apps/prober/test/fixtures/w8-target.ts. For the web side (domains-panel.tsx and
quantum-client.tsx) I checked only the XSS sinks and role gating. I read them against SPEC §4 "External discovery",
SPEC §3 domain rows, ARCHITECTURE §5.10, §6.5, the §7 threat row "External prober" and §9 INV-53 to INV-57.

## Scan categories

### Dependency vulnerability
- **Tool:** `npm audit --omit=dev` and `npm audit` (npm, Node 22.19.0).
- `--omit=dev`: **found 0 vulnerabilities.**
- Full: **8 (1 low, 7 moderate), all dev-only.** The low is esbuild GHSA-g7r4-m6w7-qqqr, already known from pass 3. The
  7 moderates are minimatch GHSA-g3hv-75wg-m249 and the typescript-eslint chain that depends on it
  (`@typescript-eslint/*`, `typescript-eslint`). They are new since pass 5, but R-3 did not bring them in: the
  advisory was published later, and the R-3 lockfile diff does not touch those packages. Neither ships in a runtime
  image.
- **What R-3 brings in:** `apps/prober` depends on `cz-scan` (workspace) and `zod` 4.6.7 (already present). Both are
  bundled by tsup, so the prober image has no node_modules. `tldts` 7.4.18 moves from dev to a runtime dependency of
  `@cz/core` and has 0 advisories.
- **Trivy 0.74.0** (`fs --scanners vuln,misconfig --severity HIGH,CRITICAL`): see "Tool results (Trivy)" at the end.

### SAST
- **Tool:** Semgrep 1.177.0 (docker `semgrep/semgrep:1.177.0`). Command:
  `semgrep scan --config p/default --config p/typescript --config p/nodejsscan --metrics=off --json` over the 66
  added or modified .ts/.tsx/.mjs/.js files in the delta. The JSON is at `artifacts/security/semgrep-pass6.json`.
- **Result:** exit 0, 9 results and 26 errors (all rule timeouts, on apps/api routes/services/tests, prober
  routes.ts and egress.test.ts, the web quantum panels, the e2e spec and core env.ts). I reviewed the hunks in the
  timed-out files by hand.
- **Triage:** all 9 results belong to classes triaged in earlier passes.
  - `helmet_*` ×4 (app.ts): helmet is configured. These are the same INFO results as passes 3 and 5.
  - `regex_dos` (app.ts:55, scanner tls.ts:64): fixed, anchored patterns with no nested quantifiers.
  - `bypass-tls-verification` (scanner tls.ts:61): deliberate. The probe records what the endpoint offers and never
    trusts it, the same as pass 5.
  - `detect-non-literal-regexp` (core domain-name.test.ts:51): a test file.
  - `node_secret` (core quantum-ingest.test.ts:14): a fixture name.
- Semgrep found nothing in the prober. The manual review below covers the rest.

### Secret detection
- **Tool:** Gitleaks v8.30.1 (docker `zricethezav/gitleaks:v8.30.1`):
  `git /repo --log-opts=6ec8983..HEAD --redact`. 29 commits and about 390 KB were scanned. **No leaks found (exit 0).**
- `git ls-files .pki .env` returns nothing, so neither is tracked.
- **Grep over the added lines** for PEM private keys, `sk_live`, AKIA and `czw_` tokens of 20+ characters: one hit,
  the `OPENSSH` detection regex in the core secret guard (expected). `smoke-test.log` contains the `czw_` prefix only
  (27 mentions), never a token body.

### Configuration and infrastructure
- **Manual review** of compose.yaml, the Dockerfile `prober` stage and the bootstrap script.
- **Networks.** The prober is on `[probe-ctl, egress]` only. `probe-ctl` is `internal: true`. The worker is on
  `[default, probe-ctl]`. The api, postgres, redis, trust-core and caddy are not on probe-ctl. The prober has no
  DATABASE_URL or REDIS_URL, and its env schema has no slot for them. Hardening: `read_only`, `cap_drop: [ALL]`,
  `no-new-privileges`, `mem_limit 256m`, `pids_limit 128`, and a USER node image with no node_modules. These pass.
- **The prober mounts the whole `pki` volume.** That includes `ca.key` and every other service's private key, all
  readable by the same uid. This is **SEC-28**.
- **mTLS.** The prober server sets `requestCert`, `rejectUnauthorized` and TLSv1.3 minimum, and allows only
  `spiffe://…/sa/worker` (checked against `socket.authorized`). The worker client verifies the chain against the
  platform CA, runs the default hostname check (`prober`), and pins `spiffe://…/sa/prober`. These pass.
- **Caddy and headers.** R-3 changes nothing in the Caddyfile, proxy.ts or security-headers.

### Environment variable audit
- `.env.example` documents EXTERNAL_DISCOVERY_ENABLED, PROBER_URL, PROBER_PORT, PROBER_ALLOWED_SPIFFE_IDS,
  PROBER_DNS_SERVERS, PROBER_CONCURRENCY, PROBER_TIMEOUT_MS, CT_SEARCH_URL (commented, default crt.sh) and
  PROBER_ALLOW_CIDRS (commented). This passes.
- **PROBER_ALLOW_CIDRS** is refused at boot when NODE_ENV=production (apps/prober/src/env.ts:281). The Dockerfile
  prober stage sets `NODE_ENV=production`, compose sets it too, and compose never passes PROBER_ALLOW_CIDRS. Only the
  Playwright test prober sets it, with NODE_ENV=test and 127.0.0.1/32. This passes.
- PROBER_URL must be `https://` (core env.ts). CT_SEARCH_URL must be `https://`, or `off`/empty to disable.
- No secrets are flagged; none are new.

### Web delivery headers and CSP
- **Source:** `FRONTEND-AUDIT.summary.json`, committed at 7f7cdd9, which is the `--full` run after F-28.
- `site.headers` pass and `site.cookies` pass. `summary.rollup` has only two Lighthouse performance groups and no
  `console` groups.
- domains-panel.tsx has no `dangerouslySetInnerHTML`, and no `href` is built from hostnames. Hostnames and TXT values
  render as text. No CSP, header or cookie code changed.

### License compliance
Skipped: RESEARCH.md does not flag it.

## Findings

### SEC-28 — MEDIUM — The internet-facing prober can read the platform CA key and every service's private key
- **Clause:** ARCHITECTURE §5.10 and §7 "External prober" (network isolation, "holds no … credentials"), INV-55,
  §6.5 (one leaf per service). Isolation is the prober's whole control.
- **Exploit path:** pki-init runs as `node` (Dockerfile tools stage, USER node). It writes `ca.key`, `trust-core.key`,
  `api.key`, `worker.key` and `prober.key` into the shared `pki` volume with mode 0600, all owned by uid 1000
  (deploy/pki/bootstrap.mjs:107). The prober runs as the same `node` user and mounts that whole volume
  (compose.yaml:196). It is the one process that parses hostile, tenant-steered input: TLS handshakes with arbitrary
  public servers, CT JSON and DNS answers. Suppose a memory-safety or logic bug in that path (Node/OpenSSL handshake
  parsing, or a future dependency) gives code execution. The attacker can then read `ca.key` and send it out over the
  `egress` network. With the CA key they can mint a leaf for any SPIFFE id: `sa/api` or `sa/worker`, which
  trust-core accepts for wrap, unwrap and sign, or `sa/prober`. Any later foothold on the default network then
  becomes full impersonation of trust-core's callers. They also get trust-core's own key, so they can impersonate
  trust-core to the api and worker. The network isolation stays intact, but the credential isolation it is meant to
  give does not.
- **Remediation:** have bootstrap.mjs write per-service directories (`/pki/<svc>/{tls.crt,tls.key}` plus a public
  `/pki/ca.crt`), and keep `ca.key` in a separate `pki-ca` volume that only pki-init mounts. In compose.yaml:196,
  mount only what the prober needs, for example with the volume `subpath` option or a dedicated `pki-prober` volume
  that pki-init fills. The prober should get `ca.crt`, `prober.crt` and `prober.key` only. Set MTLS_* to those paths.
  Apply the same scoping to api, worker and trust-core (lines 109, 139, 165). Add an invariant or test that the
  prober service's mounts contain no `*.key` other than its own.

### SEC-29 — MEDIUM — A hostile TLS endpoint can crash the shared prober with a throw inside the `secureConnect` listener
- **Clause:** SPEC §4 External discovery (8 concurrent probes per prober, shared by all tenants), ARCHITECTURE §7
  "probe flooding". It is an availability failure of a cross-tenant service driven by tenant-controlled input.
- **Exploit path:** a tenant verifies its own domain and adds `x.<domain>` pointing at a TLS server it runs. The
  server presents a leaf whose notAfter is malformed (for example UTCTime month 13). In this pass, Node 22.19 parsed
  such a certificate and returned `validTo === "Bad time value"`, and
  `new Date("Bad time value").toISOString()` threw RangeError. apps/scanner/src/tls.ts:85 runs that expression
  inside the `socket.once('secureConnect', …)` callback (tls.ts:75), outside the Promise executor's synchronous
  scope. The throw becomes an uncaught exception, and apps/prober/src/main.ts has no handler, so the process exits.
  Every in-flight request for every tenant fails. The worker then sees `ProberUnavailableError` and sets the global
  `PROBER_DOWN_KEY`, so on-demand verify and scan answer 503 for every tenant for 300 s. Compose restarts the prober,
  and the attacker re-triggers the crash on each Scan now (hourly per domain, across 10 domains on Pro) and on the
  daily sweep. `cert.publicKey` and `getEphemeralKeyInfo` in the same callback are further throw sources. In this
  pass, a stock OpenSSL server refused to serve the patched certificate ("no suitable signature algorithm"), so I did
  not reproduce delivery over the wire. A TLS stack that sends certificate bytes opaquely (rustls, or a hand-built
  stack) would remove that obstacle. The certificate is never verified (`rejectUnauthorized: false`), so the client
  side does not stop it.
- **Remediation:** in apps/scanner/src/tls.ts:75–90, wrap the body of the `secureConnect` handler in
  `try { … } catch (e) { fail(e as Error); return; }`. Compute `notAfter` defensively:
  `const t = Date.parse(cert.validTo); …(Number.isFinite(t) ? { notAfter: new Date(t).toISOString() } : {})`. In
  apps/prober/src/main.ts, add `process.on('uncaughtException', …)`: log the error and keep serving, or exit only
  after draining. Add a test that probes a fixture server presenting a certificate with a bad time, using raw DER
  in w8-target or a socket that writes canned bytes, and asserts a `reachable` finding without notAfter and no
  crash.

### SEC-30 — MEDIUM — The probe timeout is an idle timeout, so a slow-drip endpoint holds the prober's global slots without limit
- **Clause:** SPEC §4 External discovery ("a 10 s timeout", "8 concurrent probes per prober, 2 per domain").
- **Exploit path:** `tls.connect({ …, timeout: timeoutMs })` (apps/scanner/src/tls.ts:67) sets `socket.setTimeout`.
  That timer resets on every byte, so it does not bound the handshake. A tenant points several hosts under its
  verified domains at a server that trickles a ServerHello, then a large certificate chain, one byte every 9 s. Each
  probe then holds one of the 8 prober semaphore slots (apps/prober/src/routes.ts:187) for as long as the server
  likes, and each host walks up to 8 ports in series (routes.ts:151). `server.requestTimeout` (server.ts:372) closes
  the inbound HTTP request but does not cancel the handler, so the slot and the outbound socket stay held. Scan now on
  4 Pro-plan domains gives 8 lanes (2 per domain), which fills the prober. Further calls queue, and past 256 waiters
  get 503 `BUSY`. The worker maps that 503 to `ProberUnavailableError` (apps/worker/src/prober-client.ts:187), which
  sets the global `PROBER_DOWN_KEY`. Other tenants' verifies and daily scans then fail or retry until the attacker
  stops.
- **Remediation:**
  - In tls.ts, add an absolute deadline: `const timer = setTimeout(() => fail(new Error(`no TLS handshake within ${timeoutMs} ms`)), timeoutMs);`,
    cleared on `secureConnect` and on error. Keep the idle timeout as well.
  - In apps/prober/src/routes.ts:151, give each `/probe-tls` request an overall budget of
    `ports.length × timeoutMs`, and stop trying further ports once it is spent.
  - In prober-client.ts, map 503 `BUSY` to a distinct, retryable error that does not set `PROBER_DOWN_KEY`.
  - Optionally cap in-flight probes per tenant in the prober request (the worker can send `tenantId` as an opaque
    bucket key).
  - Add a test with a trickling fixture that asserts the probe settles within about `timeoutMs`.

### SEC-31 — LOW — The worker trusts prober output with no schema, locator binding or ingest caps
- **Clause:** SPEC §4 Scan ingest (locator ≤ 512 characters, ≤ 5,000 findings, control and bidi characters refused,
  ADR-085, SEC-26), INV-57, ARCHITECTURE §5.10 (findings for the probed host:port). The prober is a trust boundary:
  it is the internet-facing component.
- **Exploit path:** apps/worker/src/prober-client.ts:171–183 checks only that `findings` is an array.
  external-discovery.ts:290–305 passes it straight into `ingestFindings`. That function applies the secret guard but
  not the API's zod schema, which lives on the route. A compromised or buggy prober, or a future change to
  `probeTls` output, could inject findings with locators outside the probed domain, for example `tls://other-
  tenant-host:443`. It could also inject control or bidi characters, unbounded counts (1,000 hosts × 8 ports × 2 =
  16,000 findings per report, against the 5,000 cap on the customer path), or a forged PQC `algorithm` that moves a
  migration plan to `validated` (W8.6). That last case is forged compliance evidence.
- **Remediation:** validate each `/probe-tls` response in prober-client.ts with a zod schema:
  - `status` enum.
  - `ports` ⊆ the requested ports.
  - Each finding has `source ∈ {tls, certificate}` and `locator === tls://${host}:${port}` for a requested port.
  - `algorithm` and `protocol` are at most 64 characters and contain no C0/C1 or bidi characters.
  - `notAfter` is an ISO datetime.
  - There are at most `2 × ports.length` findings.

  Treat a violation as `ProberUnavailableError`. Alternatively, move the API's finding schema into
  `@cz/core/quantum/ingest`, so `ingestFindings` enforces it for both origins.

### SEC-32 — LOW — A verified domain whose challenge lookup keeps failing (SERVFAIL or timeout) never lapses
- **Clause:** SPEC §2.1 and §4 ("re-checked daily. Two consecutive misses lapse the domain"), INV-56.
- **Exploit path:** apps/worker/src/jobs/domain-verify.ts:249–252 counts a `ProberLookupError` (502 UNRESOLVABLE:
  SERVFAIL, upstream timeout, or a 400 INVALID_TARGET) as `failed` and changes nothing. A tenant verifies
  `example.com` and later loses it: it expires, is sold, or is parked. If the new zone answers SERVFAIL for
  `_cipherzero-challenge.example.com`, which is common with a lame delegation or broken DNSSEC on parked domains, the
  domain stays `verified` with no limit. The daily sweep and the hourly Scan now keep probing hosts the tenant no
  longer controls, under the new owner's names.
- **Remediation:** store `consecutive_lookup_failures` (or reuse `last_checked_at` and keep `last_verified_check_at`).
  Lapse a verified domain after N consecutive UTC days, for example 7, without a successful lookup. Audit it as
  `quantum.domain.lapsed` with `reason: 'unresolvable'`. Add a test that drives N days of SERVFAIL to `lapsed`.

### SEC-33 — LOW — CT parsing can use more memory than the prober's 256 MB limit
- **Clause:** SPEC §4 External discovery (resource caps), ARCHITECTURE §7 "probe flooding".
- **Exploit path:** apps/prober/src/ct.ts:218 allows a 10 MB body. It buffers the whole body, then a
  UTF-8 string, then `JSON.parse` of tens of thousands of objects, then a sort (ct.ts:295–305). That costs several
  times 10 MB of heap per call. With `PROBER_CONCURRENCY` 8, eight concurrent `/ct-search` calls for large domains can
  exceed `mem_limit: 256m` (compose.yaml:201). The kernel then kills the prober, with the same cross-tenant effect as
  SEC-29. A tenant can grow its own crt.sh result by issuing many certificates over time. Only names under the domain
  survive filtering, but the parse cost is paid first.
- **Remediation:** lower `CT_MAX_BYTES` to 2–4 MB, or stream-parse. Also run `/ct-search` through its own semaphore
  of 1–2 in routes.ts, separate from the TLS probe slots.

### SEC-34 — INFO — The CT fetch re-resolves the CT host after the egress guard
apps/prober/src/routes.ts:120–124 checks `CT_SEARCH_URL`'s host with the guard. `fetch` then resolves it again, so
the result is not pinned. The code comment documents this. The URL is operator configuration, not tenant input,
`redirect: 'error'` is set, and the prober has no route into the platform. If wanted, pin it with an undici `Agent`
whose `connect.lookup` returns the guarded address.

### SEC-35 — INFO — The per-domain run lock is released unconditionally
apps/worker/src/jobs/external-discovery.ts:218 runs `redis.del(lock)` in `finally`. A run that outlives
`RUN_LOCK_MS` (2 h) can therefore delete a newer run's lock, and two runs then probe one domain (4 lanes instead of 2).
Fix: compare-and-delete with a Lua script, `if get(k)==token then del(k)`, using a random token instead of
`runAt`.

### SEC-36 — INFO — A long domain can never be verified
`normalizeDomainName` accepts names up to 253 characters. `_cipherzero-challenge.` adds 22, and the prober refuses
names over 253 (400 INVALID_TARGET). For a domain longer than 231 characters, every verify is therefore counted as
`failed` and the domain stays `pending` for good. The effect is functional only. Fix: refuse such names in
packages/core/src/quantum/domain-name.ts:47 (`name.length > 231`), with a clear message.

## Checked with no finding
- **Egress guard (INV-54).**
  - IP literals and bracketed forms are refused.
  - A and AAAA are resolved once, CNAMEs are followed by the resolver, and every answer is checked.
  - One non-public answer refuses the whole name.
  - IPv4-mapped, NAT64 64:ff9b::/96 and 6to4 2002::/16 are unwrapped. Everything outside 2000::/3 is refused, which
    covers ULA, link-local, ::/96 IPv4-compatible, 64:ff9b:1::/48, `fd00:ec2::254` and AWS IPv6 metadata.
  - CGNAT 100.64/10 (which includes Alibaba's 100.100.100.200), 169.254/16 and all IANA special-purpose IPv4 blocks
    are refused.
  - The socket dials the guarded IP with SNI set to the host (`tls.connect` with an IP never does DNS), so rebinding
    between resolve and connect does not work.
  - The port allowlist is checked before DNS. There are no redirects and no application data.
  - CT-returned names pointing inward are refused per host and stored as `private`.
- **Prober surface.**
  - The SPIFFE check runs before routing.
  - Bodies are capped at 16 KB, and the content-type must be JSON.
  - Schemas are strict zod objects.
  - Problem+json details carry only the host and DNS error codes, the worker consumes them, and they never reach a
    customer.
- **TXT verification takeover.**
  - The token is 32 random bytes, fresh on every add and on re-add after removal, and unique per (tenant, domain) row.
  - Names are normalized: trim, trailing dot, lower case, IDNA A-label.
  - Public suffixes (PSL, including private suffixes) are refused, so `github.io` cannot be claimed.
  - The match is exact on the joined character-strings.
  - Another tenant's token is useless without DNS control of the name.
  - Lapse after two daily misses is computed per UTC day, so it cannot be forced by repeated clicks.
- **Tenant isolation and IDOR.** Every domain and host route resolves `liveDomain(tenantId, id)` first, and every
  query and update has `tenantId` in its `where`. A foreign id, an unknown id and a removed domain all return 404.
  Orphan-host adoption is limited to the same tenant.
- **Roles.** POST, PATCH and DELETE on domains, plus verify, scan and POST hosts, are guarded by
  `requireRole(owner, admin, security_officer)`. GETs are open to any member, as SPEC §3 specifies.
- **Quotas and rate limits.**
  - The `domains` limit (1/10/unlimited) is checked inside the add transaction under the plan lock and counts
    non-removed domains.
  - Scan now is 1 per domain per hour through Redis `SET NX PX`, released if the enqueue fails. It requires
    `verified` (409 otherwise).
  - Verify is deduped by a jobId over 15 s and by the global API limiter.
  - There are 1,000 hosts per domain under an advisory lock, on both the add and discovery paths.
- **Jobs (INV-46).** Both jobs are tenant-scoped with daily windows. External discovery claims the window at startup
  catch-up. Manual runs use a fresh window and carry `domainId`, with a per-domain run lock. INV-56 is re-checked
  every 25 probes and enforced by the guarded `status='verified'` update in the ingest transaction.
- **Audit.** add, ports_changed, verify_requested, verified, lapsed, removed, scan_requested, host_added and
  scan.ingested all go through `appendAudit` on a transaction.
- **Invariant lint.** `node scripts/invariant-lint.mjs` reports 60 total, 59 machine (59 pass, 0 fail) and 1 manual.
- **W8 fixture.** The DNS, TLS and control servers bind to 127.0.0.1 only. The fixture lives in test/ and is not in
  the tsup entry or the image.
- **Third-party scanning.** A verified tenant can point names under its own domain at any public IP and get a
  handshake on 8 ports. This is inherent in the design and is limited by verified domains only, the port allowlist,
  handshake-only probes, 1,000 hosts and the hourly cap, as ARCHITECTURE §7 accepts.

## Tool results (Trivy)
**NOT COMPLETED.** The command was `docker run --rm -v <repo>:/src:ro aquasec/trivy:0.74.0 fs --scanners vuln,misconfig
--severity HIGH,CRITICAL --skip-dirs node_modules,apps/web/.next,.pki --exit-code 1 --quiet /src`. It aborted with
`FATAL … walk dir error: readdirent /src: cannot allocate memory`, a Docker Desktop bind-mount limit on the Windows
host. Trivy produced no results. Dependency coverage relies on `npm audit` (above). I reviewed the Dockerfile and
compose misconfiguration by hand under "Configuration and infrastructure"; the only finding there is SEC-28.
Pass 2 should re-run Trivy on a Linux host or against the built `prober` image (`trivy image`).

## Round 2 (80fc26b)
Fix commit: 80fc26b (R-3 Review Round fix batch, ADR-094). Base of the review: `git diff 393294e 80fc26b`.
Mechanism: Claude Code Agent tool, general-purpose subagent (fresh context), read-only on source.

### What I ran
- `npx vitest run apps/scanner/test/tls.test.ts apps/prober/test/routes.test.ts apps/prober/test/pki-mount.test.ts
  apps/worker/test/domain-verify.test.ts`: 4 files, 44 tests, all pass, none skipped (OpenSSL present, so the
  `--export` run in pki-mount.test.ts executed).
- `npx vitest run apps/prober apps/scanner apps/worker/test/external-discovery.test.ts`: 14 files, 242 tests, all pass.
- `npm run lint:invariants`: 60 total, 59 machine (58 pass, 1 fail), 1 manual. The one failure is INV-2 (web
  baseline freshness; a baseline run is in progress and it is out of scope for this round).
- Semgrep 1.177.0 (`p/default`, `p/typescript`, `p/nodejsscan`) over tls.ts, certs.ts, routes.ts, main.ts,
  prober-client.ts, bootstrap.mjs and pki-mount.test.ts: 2 results, `bypass-tls-verification` and `regex_dos` at
  apps/scanner/src/tls.ts:67–71. Both are the same results triaged in round 1, and the diff did not change them.

### SEC-28 — RESOLVED
- deploy/pki/bootstrap.mjs:23–30 parses `--export <svc>:<dir>`. Lines 119–128 copy only `ca.crt`, `<svc>.crt` and
  `<svc>.key`, remove everything else in the export dir, and chmod the key to 0600 and the certificates to 0644. The
  export runs on every boot, so a re-issued CA or leaf reaches the per-service volumes. An exported service cannot
  plant symlinks there, because it mounts its volume read-only.
- compose.yaml:35 pki-init mounts `pki` plus the four `pki-<svc>` volumes. trust-core (:126), api (:156), worker
  (:182) and prober (:213) each mount only `pki-<svc>:/pki:ro`. `pki` (with ca.key) is mounted by pki-init alone.
  The MTLS_* paths are unchanged and still resolve.
- Dockerfile:33–34 creates `/x/<svc>` owned by node, so the fresh named volumes copy that ownership and pki-init
  (USER node) can write them. Each running service sees only its own key, so the shared uid 1000 no longer exposes
  other keys.
- apps/prober/test/pki-mount.test.ts asserts the mounts and runs the export (including pruning a planted `ca.key`).
  This covers the invariant/test the remediation asked for.
- Residual: the old `pki` volume of an upgraded deployment still holds every key, but no running service mounts it.

### SEC-29 — RESOLVED
- apps/scanner/src/tls.ts:58–61 `notAfterOf` uses `Date.parse` and leaves notAfter out when the value is not finite.
  It is used at tls.ts:108 and in certs.ts:108, for the local-file path too.
- tls.ts:89–96 runs the whole `secureConnect` body (`getProtocol`, `getEphemeralKeyInfo`, `getPeerCertificate`,
  `publicKeyAlgorithm`) inside try/catch and routes any throw to `fail`, which rejects only that probe.
- tls.ts:88 changes `once('error')` to `on('error')`, so a second socket error (for example a reset after `end()`)
  is no longer an unhandled `'error'` event, which would also have been a crash.
- apps/prober/src/main.ts:53–56 adds `uncaughtException` and `unhandledRejection` handlers that log and continue,
  as the backstop. The tls.ts deadline still settles any probe whose listener died, so a stray throw cannot leak a
  semaphore slot. See SEC-37 for what this handler costs.
- Test gap (not blocking): tls.test.ts unit-tests `notAfterOf`, but no test sends a malformed certificate over the
  wire. The try/catch makes the over-the-wire case safe whatever the parse result.

### SEC-30 — PARTIALLY RESOLVED (MEDIUM remains)
Fixed:
- tls.ts:77–78 sets an absolute handshake deadline, so a dripping peer is cut off at `timeoutMs`. It is cleared in
  `fail` (:81) and in `settle` (:111), and `settled` guards against a double settle. No timer leaks: every path clears
  it, and it is `unref`'d. The idle timeout is kept, so a peer that never sends FIN after `end()` is still torn down.
  tls.test.ts has the drip-fixture test the remediation asked for.
- apps/prober/src/routes.ts:128,155–160 gives each request a budget of `PROBE_BUDGET_PORTS` (4) × T. The check runs
  before each port, so the real ceiling is about 5 × T (50 s at the default) plus DNS (A and AAAA run in parallel,
  about 30 s worst case with c-ares at 10 s × 2 tries). That stays under the worker's 120 s client timeout and the
  prober's 130 s requestTimeout at the default T, so a slow request no longer turns into ProberUnavailableError.
- apps/worker/src/prober-client.ts:61–85,177,189,207 maps 503 BUSY to ProberBusyError, which neither job treats as
  ProberUnavailableError, so it no longer sets PROBER_DOWN_KEY. Retry behaviour: BullMQ gives 3 attempts with
  exponential backoff from 2 s (packages/core/src/queue.ts:64–65), then the DLQ, so there is no retry storm. With
  one worker (https.Agent `maxSockets: 8`, BullMQ concurrency 4), the prober's semaphore queue is in practice never
  more than 8 deep, so BUSY is close to unreachable anyway.

Not fixed (the residual MEDIUM):
- **Problem.** Nothing bounds a whole run, and nothing keeps one tenant to its share of the prober. The optional
  per-tenant cap from round 1 was not done.
- **Attack.** A Pro tenant verifies 4 domains. Through `POST /hosts` it points 1,000 names per domain at one IP that
  accepts TCP on the 8 allowlisted ports and then stays silent or drips bytes. It then presses Scan now on each domain.
- **Cost per run.** Each host now costs up to about 5 × 10 s, so one run takes about 1,000 × 50 s / 2 lanes, roughly
  7 hours. The four runs hold all 8 prober lanes. They also hold all 4 slots of the single BullMQ worker
  (apps/worker/src/scheduler.ts:57, one queue for every job type).
- **Effect.** For those hours, every tenant's domain verify, scan, grant-expiry, rotation sweep and other jobs wait.
  ADR-062 priority only orders the waiting jobs and does not preempt running ones. RUN_LOCK_MS (2 h,
  external-discovery.ts:178) expires before such a run ends, so with SEC-35 a second run of the same domain can start
  on the next hourly Scan now and stack.
- **Severity.** The budget cut the cost per host from about 80 s (or unbounded with a drip) to about 50 s, but it did
  not remove the cross-tenant starvation.
- **Fix:**
  1. Give `runExternalDiscovery` a wall-clock budget, for example `RUN_BUDGET_MS = 20 min`, checked in the `pool`
     lane before each host. Record partial results, then resume from the next host on the next window or mark the
     rest unprobed.
  2. Run `external-discovery` on its own BullMQ queue/Worker with concurrency of at most 2, or cap it at 1 in-flight
     run per tenant (a Redis counter keyed by tenantId). Then it cannot take the general worker's 4 slots.
  3. Optionally, send `tenantId` to the prober as an opaque bucket and cap its in-flight probes per tenant at 2
     there as well.
  4. Make RUN_LOCK_MS longer than the run budget.
  5. Add a test: a run over N silent hosts stops at the budget, and a verify job queued behind it starts within
     the budget.

### New findings introduced by the fix
- **SEC-37 — LOW — The prober's log-and-continue `uncaughtException` handler also swallows boot-time server errors,
  and it logs no stack.**
  - Where: main.ts:53 registers the handler after `server.listen` (:43), and server.ts adds no `server.on('error')`.
  - Effect: an async listen failure (EADDRINUSE or EACCES on 4100) is now logged and swallowed. The process stays
    alive but never listens. The prober service has no healthcheck, and `restart: unless-stopped` never fires. The
    prober is then silently down (the worker sees ECONNREFUSED, sets PROBER_DOWN_KEY, and discovery fails closed).
    This is an availability and operability defect, and an attacker cannot trigger it.
  - It also logs only `err.message` and swallows every future programming error without a stack. Node documents
    the process state after an uncaughtException as undefined.
  - Fix:
    1. Add `server.on('error', (e) => { log.error(...); process.exit(1); })`.
    2. Log `err.stack`.
    3. Exit, draining first, once more than a small number of uncaught exceptions arrive per minute.
    4. Optionally, add a compose healthcheck on the prober (TCP connect to 4100).
- **SEC-38 — INFO — `bootstrap.mjs --export` prunes recursively with no guard on the target dir.**
  - Where: bootstrap.mjs:122–124 runs `rmSync(..., { recursive: true, force: true })` on every entry of `<dir>`
    except the three kept files. Nothing stops `<dir>` from being the `--out` dir or one of its ancestors.
  - Effect: `--export prober:/pki` would delete ca.key and every other key, so the next boot mints a new CA and
    re-issues everything. `--export x:/` would delete whatever node can write. The value is operator configuration
    (compose.yaml), not tenant input, so this is a footgun only.
  - Fix:
    1. Refuse a `dir` that is equal to `out` or an ancestor of it (`path.relative(dir, out)` does not start with `..`).
    2. Refuse duplicate dirs.
    3. Remove only regular files (`lstat`), without `recursive`.
- **SEC-39 — INFO — The worker's 120 s client timeout is hard-coded, while the prober's per-request time scales with
  PROBER_TIMEOUT_MS.**
  - Where: the worker uses 120 s at apps/worker/src/main.ts:32. PROBER_TIMEOUT_MS accepts up to 60,000
    (apps/prober/src/env.ts:40).
  - Effect: one request can take about (PROBE_BUDGET_PORTS + 1) × T plus about 3 × T of DNS. Above about T = 15 s,
    a slow request exceeds 120 s. The worker's socket timeout then raises ProberUnavailableError and sets the global
    PROBER_DOWN_KEY, the round-1 SEC-30 effect. The default T = 10 s is safe.
  - Fix: cap PROBER_TIMEOUT_MS at 15,000, or give the worker PROBER_TIMEOUT_MS and derive its timeout as
    `(PROBE_BUDGET_PORTS + 1 + 3) × T + 10 s`.

### SEC-31 to SEC-36 (recorded, not required this round)
I checked each one against the diff. None changed:
- SEC-31: prober-client.ts still checks only that `findings` is an array.
- SEC-32: domain-verify.ts was not touched.
- SEC-33: ct.ts and the CT semaphore were not touched.
- SEC-34: routes.ts `searchCt` was not changed.
- SEC-35: the external-discovery.ts lock was not changed.
- SEC-36: domain-name.ts was not touched.

All six stay at their round-1 severity. ADR-094 "Known gaps" records them.

### Residual after round 2
CRITICAL 0 · HIGH 0 · MEDIUM 1 (SEC-30 residual: cross-tenant starvation of prober lanes and worker slots, no run
budget) · LOW 4 (SEC-31, SEC-32, SEC-33, SEC-37) · INFO 5 (SEC-34, SEC-35, SEC-36, SEC-38, SEC-39)
