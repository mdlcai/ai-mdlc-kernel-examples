# DECISIONS.md — Sentinel

Architecture Decision Records (ADRs). Assumptions, deviations, blank-field defaults, and technical debt. Referenced as `ADR-NN` throughout the build.

> Review gate: `review_gates: auto` (explicit in RESEARCH.md). All stage gates and the multi-agent plan gate auto-proceed; outcomes logged here rather than halting. Logged per the kernel's review-gate rule.

## ADR-001 — Tenant isolation: app scoping + Postgres RLS (defense-in-depth)
`multi_tenant: true` and `pii_handling` is set. Chose `withOrgScope` app-level scoping as the testable primary mechanism, with Postgres RLS as a second layer. Trade-off: double cost; justified by data-sensitivity of security findings + git credentials.

## ADR-002 — Scanner sandbox runtime: Docker + hardened flags, gVisor when available
Untrusted scanner engines run user-supplied code/targets. Chose Docker with `--network none|restricted --read-only --cap-drop ALL --no-new-privileges --pids-limit --memory --cpus --rm`, gVisor (`runsc`) when the host provides it. Firecracker rejected as too heavy for the small tier. Spike open (RESEARCH §6): raw-socket scanners under gVisor.

## ADR-003 — Canonical Finding shape inspired by SARIF 2.1.0
Adopt SARIF-2.1.0 field semantics for the internal `Finding` rather than a bespoke schema; most engines already emit SARIF, and it is the open standard (RESEARCH §3.5/§3.9).

## ADR-004 — Deterministic dedup fingerprints
Dedup by `hash(class-specific key)`: SAST/IaC `ruleId+file+lineRange`; SCA `cve+package+version`; network `host+port+service`. Explainable and testable over ML merge — aligned with the "trust the short list" product thesis.

## ADR-005 — Password hashing: argon2id
NIST 800-63B memorized-secret verifier alignment. No silent substitution to bcrypt/bcryptjs (would be an INV-3 violation).

## ADR-006 — Demo SCA scanner default-on; production engines behind `SCANNERS_DOCKER_ENABLED`
To keep the build genuinely runnable and smoke-testable on hosts without the nine scanner images, the default scanner is a real SCA engine that parses actual dependency manifests and matches them against a committed OSV-shaped vulnerability fixture set, producing real Findings through the full normalize→enrich→dedup→persist pipeline. Production engines (Trivy, Semgrep, OSV-Scanner, gitleaks, checkov, ZAP, Nuclei, nmap, masscan) are sibling adapters gated by an env flag, sharing the identical orchestrator/normalizer/dedup/UI. This is a scoped MVP cut (RESEARCH "MVP cut" step 2: one scanner end-to-end), not a downgrade of the architecture. Logged as technical debt: wire the remaining adapters + Docker images.

## ADR-007 — API error contract: RFC 9457 problem+json
RFC 7807 is obsoleted by RFC 9457 (RESEARCH §3.5). All API errors use a `problem+json` helper carrying `type/title/status/detail/errors[]` so the UI can bind field-level messages to controls (DESIGN.md §5 error-contract consumption).

## ADR-008 — `has_geo` domain signal not exercised at small tier
The blueprint lists `has_geo` among domain signals, but the product surface (repo/domain/IP scanning) has no geospatial requirement in SPEC §5. Noted and deferred; no geo subsystem built. Revisit if a geo-aware feature is specced.

## ADR-009 — Protocol: HTTPS only
`protocol_support: "HTTPS only"`. Production: TLS at reverse proxy, HTTP→HTTPS redirect, HSTS. Local dev: mkcert. The local-stack smoke test runs against the proxied `https://localhost` (with a documented HTTP `http://localhost:<port>` direct-to-api fallback for environments without mkcert; logged so the smoke test URL is honest about which it hit).

## ADR-012 — Security Audit: postcss MEDIUM accepted (bundled in Next, not reachable)
Pass 1 flagged postcss <8.5.10 (GHSA-qx2v-qp2m-jg93, MEDIUM). Auto-remediated the
first-party copy via a root `overrides: { postcss: ^8.5.10 }` → 8.5.15. The residual is
Next.js's *bundled* postcss 8.4.31, which npm overrides cannot replace; downgrading Next is
an unacceptable breaking change. The advisory is XSS in postcss stringify of UNTRUSTED CSS;
Next's bundled copy runs only at build time on our own first-party CSS and never at
runtime, so it is not reachable in our threat model. Disposition: ACCEPTED, tracked to bump
when Next bundles postcss ≥8.5.10. See SECURITY-AUDIT.md.

## ADR-013 — invariants.json ↔ implementation reconciliation (JS, not TS)
The build is authored in ESM JavaScript (no transpile step; runs on Node 22 directly), so the §9/invariants.json file references authored as `.ts` were corrected to `.js`, INV-6's pattern narrowed to `execFileSync(...)` (the literal `child_process` token only appeared in docs, causing a false positive) with `**/*.md`/`**/*.json` excluded, and INV-7's `in` glob retargeted to the actual `apps/api/src/routes/webhooks.*` with `first/second` set to the real verify→recordDelivery boundary. ARCHITECTURE §9 and invariants.json amended together in the same pass (kernel convention). No invariant was disabled.

## ADR-011 — Multi-Agent Plan Gate (logged, auto-dispatched)
review_gates: auto. Plan surfaced + logged, not halted. Waves: A foundation (config/core/db/invariant-lint) → B api+worker (auth, orgs, projects, targets, scans, findings, webhooks, orchestrator, SandboxRunner, demo SCA adapter, normalize/dedup, notifications) → C web (Next.js shell + 11 screens) → D tests+CI+scaffold. B↔C contract-coupled → sequential; files disjoint within waves. ~14 features / 4 waves. Built in dependency order on a single shared working tree (parallel agents would collide on shared packages), which is the kernel-sanctioned fallback when work is not file-disjoint.

## ADR-010 — review_gates dual-state
`review_gates: auto` is explicit (not dual-blank), so the conservative default-halt at the Multi-Agent Plan Gate does not apply; the plan is summarized, logged, and auto-dispatched per the outer prompt and kernel `auto` row.
