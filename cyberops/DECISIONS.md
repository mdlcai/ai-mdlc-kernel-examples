# DECISIONS.md — CyberOps

Assumptions, deviations, and technical debt. ADRs are append-only; supersede rather than delete.

---

### ADR-001 — Build Strategy
`build_depth: comprehensive`, `review_gates: auto`, `force_research: false`, `domain: Security` (from RESEARCH.md Build Strategy). Autonomous run: all gates self-verify and proceed; only hard-blocking errors or a NO-GO halt. `prompt_mode` blank → `direct` (default, logged).

### ADR-002 — Key decisions (blank fields resolved)
`deployment` → **cloud-native AWS**, with a **local docker-compose stack** as the runnable/deploy-in-sandbox target (RESEARCH `hosting_environment` = cloud-native AWS/GCP/Azure). `backend_language` (blank) → **TypeScript** (matches NestJS constraint, one language across the monorepo). `auth_model` (blank) → **email+password with breached-password screening, JWT in an httpOnly/SameSite cookie session, RBAC roles (owner/admin/analyst/viewer), MFA-ready** — chosen because the platform handles sensitive security data under SOC2 and needs first-party sessions, not a third-party IdP dependency, for the initial build. Rationale: least external dependency, satisfies NIST 800-63B-4 verifier rules directly.

### ADR-003 — Scale tier = large (50k+), with signal bumps
`scale: large` → connection pooling (+PgBouncer doc), Redis cache + BullMQ queue/workers, stateless multi-instance app behind LB, Redis-backed throttler, edge rate-limit documented, sustained-load perf verification target. Domain-signal bumps: `has_websocket` → Socket.io **Redis adapter** for cross-instance fan-out (raises the realtime row); `has_dual_write` → transactional outbox; `has_webhooks` → verify-before-DB + idempotency unique constraint.

### ADR-004 — Design tokens copied VERBATIM from DESIGN-TEMPLATE.html (overrides project config)
A design template (`index.html`, sha256 8b015b5d…) was uploaded. Per the RESEARCH "Design Template" precedence, its `:root` tokens are copied verbatim and are the canonical token set for the whole build, **overriding** the lime-green (`#4add2c`) values in `get_project_config`/Design Language. The template is a **dark cyber-ops glassmorphism** system. Exact `:root` block copied:

```
--cyber-base:#050810; --cyber-elevated:#0a0e1a; --cyber-overlay:#0f1424; --cyber-surface:#141a2e;
--rose:#f43f5e; --rose-dim:#e11d48;
--cyan:#06b6d4; --cyan-dim:#0891b2;
--sev-critical:#ef4444; --sev-high:#f97316; --sev-medium:#eab308; --sev-low:#3b82f6; --sev-info:#64748b;
--up:#00E68A; --down:#FF4466; --warn:#FFAA00;
--tx-1:#e8ecf4; --tx-2:#8892a8; --tx-3:#556677; --tx-4:#3D4F5F;
--bd-subtle:rgba(255,255,255,.05); --bd-default:rgba(255,255,255,.07);
--glass-bg:rgba(10,14,26,.55); --glass-strong-bg:rgba(10,14,26,.80);
--font-display:"Orbitron","Rajdhani","Eurostile",ui-sans-serif,system-ui,sans-serif;
--font-body:"Geist","Inter",system-ui,-apple-system,"Segoe UI",sans-serif;
--font-mono:"Geist Mono","JetBrains Mono",ui-monospace,"Cascadia Code",Consolas,monospace;
```
Surfaces ported from the template: landing (nav, hero, features, how-it-works, pricing, globe, footer) and the app shell (sidebar + topbar + glass cards + severity badges). Archetype remains `saas` (dashboard/tables/forms), expressed through these tokens. A light-mode variant is derived alongside (never replacing) the template values for the theme toggle; if any template token fails WCAG AA contrast on a screen, it is adjusted the minimum needed for AA and logged here (accessibility outranks fidelity).

### ADR-005 — Safety boundary: no live exploitation module (binding)
The build implements safe recon + a template-driven finding engine + exploit-validation **state** gated behind verified authorization scope and human approval; it does **not** fire exploit payloads at arbitrary targets. This is the legal/ethical boundary for a product that runs authorized offensive tooling (RESEARCH §5, R1/R2/R6/R7). The adapter interface leaves room for an authorized, sandboxed exploit runner behind the approval gate as future work. **Cascade:** none of the five key workflows is dropped — W1 validation is modeled as state + a simulated/gated validation adapter, so acceptance criteria are preserved.

### ADR-006 — Build-environment version pins vs RESEARCH §3.1 bleeding edge (RESEARCH-deliverable guard)
RESEARCH §3.1 verified the current bleeding edge (Next 16, Tailwind v4, TypeORM 1.1, NestJS 11). Several are on a transition boundary. To keep the sandbox build install/typecheck/build-clean, the build pins the nearest battle-tested majors where needed: **Next.js 15, Tailwind v3, TypeORM 0.3**, NestJS 11 (kept). **RESEARCH-deliverable guard analysis:** the *deliverables* — the Next.js+shadcn+Tailwind+TanStack frontend, the NestJS+TypeORM+Postgres backend, the design system, the accessibility/performance targets, and all §3 success metrics — are fully preserved; only the framework patch/major pin moves, and the token layer + component APIs used are compatible across these majors. This is a build-environment substitution (not a stack or design-system swap), which the guard permits with a documented equivalent. Cascade impacts: none to SPEC §5 workflows; noted for the operator to bump post-build.

### ADR-007 — Search index for dual-write = Postgres FTS in-cluster (build); Algolia/OpenSearch = documented swap
`has_dual_write` resolved via the transactional outbox. The second system (search index) is Postgres full-text search in the same cluster for the build (no extra external), drained by the outbox relay. Swapping to Algolia/OpenSearch is an env-config + adapter change documented in QUICKSTART.

### ADR-008 — Deploy target = local docker-compose (Deploy Reachability Outcome C)
AWS-managed externals (Secrets Manager, RDS, ElastiCache, Resend prod key, real Nuclei/Trivy binaries) are unprovisioned in the build environment. Per BUILD.md Deploy Target Reachability, the ship target is a **local docker-compose stack** (api, web, postgres, redis, worker); the Functional Smoke Test runs against `http://localhost`. Cloud swap documented in QUICKSTART. Protocol: RESEARCH `protocol_support: HTTPS only` → prod forces HTTPS + HSTS (managed cert / Let's Encrypt via proxy); **local dev uses HTTP on localhost** with the HTTPS-later path documented (logged here per Blank Field / protocol policy).

### ADR-009 — Stage 1 review gate outcome
`review_gates: auto` → self-verify completeness and proceed to Stage 2. Build Input Reconciliation complete (REPORT.md) — every non-blank field dispositioned, no unresolved Conflict. Dual-blank state N/A.

### ADR-010 — Multi-Agent Plan Gate (dispatch plan)
`review_gates: auto` → plan surfaced + logged, agents dispatched without halt. Wave plan (dependency-ordered; file-disjoint waves parallelize):
- **Wave 0 (foundation, sequential first):** F-01 scaffold/config/DB/health/error/logging, F-02 tenancy+RBAC+scope chokepoint+RLS. Everything depends on these — built by the orchestrator directly (shared surface).
- **Wave 1 (auth, sequential after W0):** F-03 register/login/logout/session, F-04 password reset. Shared auth module.
- **Wave 2 (core domain — file-disjoint, parallelizable):** F-05 assets+scope, F-06 scans+adapters+WS, F-07 findings. F-06 depends on F-05 (scope); F-07 depends on F-06 (findings from scans) → built in dependency order within the wave.
- **Wave 3 (analytics/domain — parallelizable after W2):** F-08 pentest, F-09 OSINT, F-10 compliance, F-11 remediation/retest, F-12 reports.
- **Wave 4 (cross-cutting backend):** F-13 alerts (email outbox + outbound webhooks), F-14 inbound webhooks, F-15 audit viewer.
- **Wave 5 (frontend surfaces):** F-16 dashboard, F-17 landing (template) + settings; plus all screens from §8 wired per vertical slice.
Bundle Integrity: each feature keeps its own per-feature banner + contract enumeration; no ADR bundles multiple features' obligations into a summary. Sequencing rationale: foundation and auth are shared-surface (sequential); domain modules are file-disjoint under `apps/api/src/<module>` + `apps/web/app/<route>` (parallel-safe). Estimated ~17 feature passes.
Given the single-orchestrator runtime and shared-file coupling (shared DTO contracts, migrations, app module wiring), features are built sequentially in dependency order by the orchestrator with per-feature gates, rather than dispatched to isolated parallel worktrees — this avoids migration/app-module merge conflicts while preserving the per-feature loop, banners, and gates. Logged per Bundle Integrity Rule.

### ADR-011 — Password hashing = Node core scrypt (not bcrypt/bcryptjs)
Native `bcrypt` requires node-gyp compilation that frequently fails on Windows; swapping to `bcryptjs` is explicitly prohibited by BUILD.md (Substitution Discipline). Chose **Node core `crypto.scrypt`** — a vetted, memory-hard KDF with no native-compile step and a timing-safe verify. Format `scrypt$N$salt$hash`. This satisfies the security primitive without the prohibited substitution and without a Windows build-tools dependency. See `common/security/password.service.ts`.

### ADR-012 — INV-2 refinement (SSRF boundary-order)
INV-2 was refined from a per-adapter `httpFetch` boundary-order to an orchestrator-level boundary-order: within `scan-orchestrator.service.ts`, `resolveAndValidate` must precede `adapter.run(`. Rationale: the real security property is that the orchestrator SSRF-validates and IP-pins the target once, before dispatching ANY adapter; adapters then operate only on the pre-validated `ResolvedTarget` (no adapter touches user-supplied DNS/URLs directly). `ARCHITECTURE.md` §9 and `invariants.json` updated together (this ADR).

### ADR-013 — Reviewer Gate FAIL remediation: object-level authorization enforced (INV-13/SEC-2)
The independent Reviewer Gate found `assertResourceAccess` was defined + unit-tested but never invoked — an intra-tenant IDOR on owner-bearing resources (OWASP A01, the #1 risk in our own threat model). Remediation (re-entered Stage 3):
- **Owner-scoping model:** owner-bearing resources (assets `ownerUserId`; pentests/reports/investigations `createdBy`) are owner-scoped for self-service roles (analyst/viewer) — they see and mutate only their own; admin/owner see all (collaboration override). Findings/scans remain org-collaborative per SPEC §8 (all members read the org's security data).
- Wired `assertResourceAccess` into `AssetsService.get/update/remove/verifyScope/getScope` and `ownerScopeWhere` into `list`, and into pentest/reports/osint get + list.
- **Made the invariant machine-checkable:** added INV-15 (`required-pattern`) asserting `assertResourceAccess(` appears in ≥4 owner-bearing services, so a false-green manual check can't recur; added an IDOR spec (`assets/idor.integration.spec.ts`).
Also fixed two Reviewer MEDIUMs: (1) `pinnedHttpProbe` now connects to the pre-validated IP (Host header / SNI) instead of re-resolving the hostname — closes the DNS-rebinding TOCTOU; (2) inbound webhook HMAC now verifies over the RAW request body (`rawBody: true`) so real third-party signatures verify.

### ADR-014 — Ownership model finalized (Reviewer Gate passes 2–4)
Resolved after independent review found residual IDOR: **owner-scoped** resources = assets, pentests, reports, investigations (self-service roles see/act on their own; admin/owner override). **Org-collaborative** = findings, scans (all members view the org's security intelligence). **Launching new work against an asset (create scan, create pentest) requires asset access** (`assertResourceAccess(asset)`), because the asset is an owner-scoped input; retest acts on an already-org-shared finding and stays collaborative. Fixed call sites added over three review rounds: assets get/update/remove/verifyScope/getScope, reports.get/download, pentest.get/decide/create, scans.create, osint.getInvestigation, + owner-filtered lists. Machine-guarded by INV-15; unit-locked by idor.integration.spec.ts.
