# DECISIONS.md — MCP Sentinel

Architecture Decision Records, assumptions, deviations, technical debt, and gate
outcomes. Reconciled against the codebase at Stage 4.

## Build Strategy (resolved at Stage 1)
- `build_depth: comprehensive` → exhaustive SPEC (every state transition, error
  path, race/idempotency), threat model in ARCHITECTURE §10 / RESEARCH §5,
  per-feature targeted security + architecture-adherence reviews, `SECURITY-AUDIT.md`,
  `DESIGN-NOTES.md`, representative load test at smoke.
- `review_gates: auto` → all approval gates and the post-Stage-2 menu are
  suppressed; each stage summarizes and auto-proceeds. Logged here per kernel.
- `prompt_mode` blank → default `direct`.
- `force_research: false` and §3 was absent → research was performed (not skipped).

## ADRs

- **ADR-001 — Backend on Node.js LTS (not Bun).** Hono is runtime-portable; Node
  chosen for ecosystem/library maturity (official MCP SDK, undici, pino, re2). A
  later Bun migration is low-cost because Hono abstracts the runtime.
- **ADR-002 — Drizzle ORM (not raw pg).** Type-safe schema + migrations while
  retaining the `pg` driver for low-level control. Satisfies `database_preference: PostgreSQL`.
- **ADR-003 — Deterministic rule engine (not LLM-as-judge).** Avoids
  scanner-prompt-injection (RESEARCH T4), meets the <10s perf target, and yields
  fully explainable findings. Encoded as INV-7.
- **ADR-004 — `http://` targets permitted even though Sentinel is HTTPS-only.**
  `protocol_support: "HTTPS only"` governs *Sentinel's own* serving surface. Target
  MCP servers may legitimately be plaintext dev endpoints; SafeConnector still
  performs full IP validation regardless of scheme. Non-`http(s)` schemes denied.
- **ADR-005 — IP pinning via a custom undici dispatcher (not the npm `ip` package).**
  `ip` CVE-2024-29415 misclassifies private addresses. Sentinel resolves DNS
  itself, validates every resolved IP, and pins the connection. Encoded as INV-1/INV-2.
- **ADR-006 — `re2` (linear-time) for untrusted-string matching (not native RegExp).**
  Detection rules run on attacker-controlled tool descriptions; native backtracking
  RegExp is a ReDoS vector (RESEARCH T5). Encoded as INV-9.
- **ADR-007 — Server-side session cookies (not JWT).** Enables instant revocation
  and is simpler at `scale: small — under 1k users`.
- **ADR-008 — argon2id password hashing + HIBP k-anonymity breach screening.**
  NIST 800-63B aligned (length-not-composition, breached-password check, rate-limit
  before hash, no forced rotation). Satisfies the security baseline floor.
- **ADR-009 — Design template `:root` tokens copied verbatim as canonical tokens.**
  The customer uploaded `DESIGN-TEMPLATE.html`; per DESIGN.md Part III its `:root`
  block is transcribed exactly and overrides `get_project_config` where they
  conflict (template `--color-bg:#f6f8fb` / `--color-surface:#ffffff` /
  `--color-accent:#2563eb` etc.). Exact copied block below.
- **ADR-010 — `domain_signals: ["has_webhooks"]` dispositioned Deferred.** MCP
  Sentinel has **no inbound webhook surface** in v1 (RESEARCH Non-Goals: no
  continuous monitoring, no external callbacks; all scans user-initiated). The
  signal does not map to a built feature. The webhook *domain knowledge* is instead
  applied to the product's subject matter — Sentinel's `has_webhooks` risk awareness
  informs detection of MCP servers that declare `tools.listChanged` (the rug-pull /
  mutable-contract signal). No inbound webhook endpoint, idempotency table, or
  signature-verification handler is built because none is reachable. Re-evaluate if
  v2 adds continuous re-scan webhooks.
- **ADR-011 — `testing_strategy: "test-after"` honored.** Tests are written
  immediately after each feature's implementation within the same feature slice
  (per the Stage 3 loop), not test-first. Coverage still gates each feature.

### ADR-009 — Verbatim copied `:root` design tokens (canonical token set)

```css
:root {
  --color-bg: #f6f8fb;
  --color-surface: #ffffff;
  --color-surface-2: #eef2f8;
  --color-fg: #0c1424;
  --color-muted: #5b6779;
  --color-border: #e1e7f0;
  --color-accent: #2563eb;
  --color-accent-soft: #e8f0ff;
  --color-accent-strong: #1d4ed8;
  --color-success: #15803d;
  --color-success-soft: #e7f6ec;
  --color-warning: #b45309;
  --color-warning-soft: #fdf0db;
  --color-error: #c5293b;
  --color-error-soft: #fbe9eb;
  --color-on-accent: #ffffff;
  --font-display: "Manrope", system-ui, sans-serif;
  --font-body: "Inter", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px;  --space-4: 16px;
  --space-5: 24px; --space-6: 32px; --space-7: 48px;  --space-8: 64px; --space-9: 96px;
  --radius-sm: 6px; --radius-md: 12px; --radius-lg: 20px; --radius-pill: 999px;
  --shadow-sm: 0 1px 2px rgba(12, 20, 36, 0.06), 0 1px 1px rgba(12, 20, 36, 0.04);
  --shadow-md: 0 6px 16px rgba(12, 20, 36, 0.08), 0 2px 6px rgba(12, 20, 36, 0.05);
  --shadow-lg: 0 28px 64px rgba(12, 20, 36, 0.16), 0 10px 24px rgba(12, 20, 36, 0.10);
  --maxw: 1200px;
}
[data-theme="dark"] {
  --color-bg: #080b12;        --color-surface: #10151f;   --color-surface-2: #161d2b;
  --color-fg: #e8eef8;        --color-muted: #93a1b8;     --color-border: #232c3d;
  --color-accent: #4d86f7;    --color-accent-soft: #15233f; --color-accent-strong: #6ea0ff;
  --color-success: #4ade80;   --color-success-soft: #11281c;
  --color-warning: #fbbf4d;   --color-warning-soft: #2a2010;
  --color-error: #ff6b7a;     --color-error-soft: #2c151a;  --color-on-accent: #08101f;
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.4);
  --shadow-md: 0 8px 20px rgba(0,0,0,0.45);
  --shadow-lg: 0 30px 70px rgba(0,0,0,0.6), 0 12px 28px rgba(0,0,0,0.45);
}
```
Landing/home surface scaffold ported from `DESIGN-TEMPLATE.html` (sticky nav,
two-column hero with product mockup, bento feature grid, risk strip, CTA block,
footer).

## Multi-Agent Plan Gate (Stage 3 — review_gates: auto → logged, auto-dispatched)
Build features re-decomposed into 8 vertical slices (endpoint(s)+screen(s)+e2e
delivered together; UI never collapsed). Waves by dependency (file-disjoint
slices may parallelize; coupled ones sequential):
- **Wave 1 (foundation):** F-00 Scaffold — monorepo (apps/web, apps/api,
  packages/shared), tsconfig/eslint/vitest, Drizzle schema + migrations (all 9
  tables, INV-6 unique), `/api/health`, `tokens.css` (INV-10), Tailwind theme,
  app shell. Tables: users, sessions, endpoints, scans, findings,
  inventory_items, scan_diffs, share_links, audit_log.
- **Wave 2 (parallel, file-disjoint):**
  - F-01 Auth slice — POST `/api/auth/register|login|logout`, GET `/api/auth/me`;
    argon2id, session cookie, CSRF, rate limit, HIBP screen; Register + Login
    screens; e2e auth bootstrap.
  - F-02 SafeConnector — `apps/api/src/scanner/safe-connector.ts` (INV-1/INV-2);
    DNS resolve + resolved-IP deny-list + IP pinning + redirect re-validation. Unit IP matrix.
  - F-03 Scanner engine — McpClient (Streamable HTTP enumerate `initialize`→
    `tools/list`/`resources/list`/`resources/templates/list`/`prompts/list`,
    INV-14), Normalizer (zod + definition_hash), RuleEngine (7 detectors,
    linear-time matcher, INV-9), Grader; bundled known-good/known-bad fixture servers.
- **Wave 3 (scan→report, depends F-01/F-02/F-03):** F-04 Scan+Report slice —
  POST `/api/scans`, GET `/api/scans`, GET `/api/scans/:id`; ScanService
  orchestration; Dashboard, new-scan, scan-in-progress, Report, finding
  drill-down + raw JSON-RPC viewer. Covers W1+W2. e2e W1, W2.
- **Wave 4 (parallel, depend F-04):**
  - F-05 History slice — GET `/api/endpoints`, GET `/api/endpoints/:id`, DELETE
    `/api/endpoints/:id`; Endpoint history screen + grade trend. W4. e2e W4.
  - F-06 Rescan+Diff slice — POST `/api/scans/:id/rescan`; DiffService; Compare/diff
    view. W3. e2e W3.
  - F-07 Share/Export slice — POST `/api/scans/:id/share`, GET `/api/share/:token`,
    GET `/api/scans/:id/export`; shared public report + export action. W5. e2e W5.

**Bundle Integrity (per-feature enumeration above is flat/scannable).** Wave
count ≈ 4; scope ≈ 8 slices, ~16 endpoints, 13 screens, 7 detectors, ~40+ tests.
Sequencing rationale: F-00 is hard foundation; F-01/F-02/F-03 are file-disjoint
(auth vs scanner-egress vs rule-engine) so conceptually parallel; F-04 needs all
three; F-05/F-06/F-07 each extend F-04 on disjoint routes/screens. As a single
build agent I execute in this dependency order, emitting a completion banner per slice.

### ADR-012 — Linear-time matcher instead of native `re2` (amends ADR-006).
The `re2` npm package is a native addon requiring node-gyp/C++ build tooling that
is unreliable cross-platform (notably Windows) and would make `npm ci` fragile in
the deploy path. To uphold the ReDoS invariant (RESEARCH T5, INV-9) without a
native dependency, detection uses a **custom linear-time matcher**
(`apps/api/src/scanner/rules/matcher.ts`): bounded input (≤32 KB/field), substring
/ `indexOf` / single-pass character & Unicode-category scanning — inherently
linear, no backtracking. **No native backtracking `RegExp` is run on untrusted
MCP strings in the rules dir.** INV-9 guidance updated to reference the
linear-time matcher rather than the `re2` module. Cascade: none — same guarantee
(linear-time, bounded), no RESEARCH deliverable downgraded.

### ADR-013 — `scrypt` (node:crypto built-in) for password hashing (amends ADR-008).
Same reasoning as ADR-012: `argon2`/`bcrypt` are native addons that make `npm ci`
fragile cross-platform. Node's built-in `crypto.scryptSync` (N=2^15, r=8, p=1,
per-user 16-byte salt, 64-byte key, timing-safe compare) is a memory-hard KDF
explicitly listed as acceptable by NIST 800-63B §5.1.1.2. Zero native dependencies
→ reproducible install in every environment. HIBP k-anonymity breach screening
(ADR-008) is retained unchanged. Cascade: none — KDF strength preserved, no
RESEARCH deliverable downgraded; the security baseline (breached-password screen,
length-not-composition, rate-limit-before-hash) is fully honored.

### ADR-014 — Direct JSON-RPC Streamable HTTP client (drop `@modelcontextprotocol/sdk`).
Enumeration needs only a small fixed set of methods (`initialize`, `tools/list`,
`resources/list`, `resources/templates/list`, `prompts/list`) and demands full
control over the byte ceiling, total deadline, and — critically — SSRF IP-pinning
on every request (the SDK transport will not honor an IP allow-list, per RESEARCH
§3.8 caveat). A thin direct client over the SafeConnector-pinned `undici`
dispatcher gives that control and removes a heavy dependency, so the official SDK
is not used. ARCHITECTURE §4 already permitted "a thin direct JSON-RPC path … to
keep byte/time caps enforceable." Spec semantics (Accept header, SSE vs JSON,
session id, protocol version, legacy fallback) are implemented per the verified
RESEARCH §3.1 transport notes. Cascade: none — capability unchanged, security
strengthened.

### ADR-015 — Opt-in SSRF allow-list (`SCAN_ALLOW_HOSTS`, default empty).
The SSRF guard correctly denies loopback/private targets — but two legitimate
cases need a private target: (a) a user scanning their own localhost/internal MCP
dev server, and (b) the test/smoke fixtures. Rather than weaken the guard, an
explicit, default-EMPTY allow-list lets an operator name specific trusted hosts
that may bypass the private-IP denial. Production default = empty = fully locked
down (every threat-model control intact). The allow-list still resolves and pins
the IP; it only skips the private-range rejection for the named hosts. Documented
in `.env.example` and QUICKSTART. Smoke/integration tests set it to the fixture
host only. Cascade: none — the default posture is unchanged; this is an additive,
explicitly-opt-in capability.

### ADR-016 — Reviewer Gate advisories dispositioned (non-blocking).
1. **Register anti-enumeration residual.** Registering an *existing* email with a
   *wrong* password returns a 422 envelope while a fresh email returns 201 — a
   shape difference an attacker could probe (timing is already equalized via the
   dummy hash). Fully closing this requires an email-verification "check your
   inbox" flow, which is **out of v1 scope** (RESEARCH Non-Goals: no email/multi-
   user collaboration). **Accepted** for v1 with per-IP rate limiting as the
   compensating control; v2 fix = verification-link registration. Logged, not a blocker.
2. **Compare route param.** SPEC §10 lists `/app/scans/:id?compare=:prevId`; the
   build renders the diff **inline** from the persisted `scan_diffs` row vs the
   prior completed scan. W3's terminal (diff view) is reachable; the explicit
   `?compare=` selector is deferred as a UX nicety. **Accepted.**
3. **Drizzle email index cosmetic drift.** The Drizzle index is `.on(t.email)`
   while the authoritative SQL migration uses `lower(email)`; the SQL migration
   runs at boot and the service lowercases email before insert, so behavior is
   correct. Cosmetic only. **Accepted.**

## Gate outcomes (appended as the pipeline runs)
- Stage 0: GO (conditional on SSRF invariants INV-1/INV-2 — encoded). §3 was absent → researched.
- Stage 1: review_gates auto → self-verified, proceeded. Build Input Reconciliation complete (16/16 non-blank fields, 0 unresolved Conflict). See REPORT.md.
- Stage 2: review_gates auto → self-verified, proceeded. SPEC comprehensive; UI Surface 13 screens; all W1–W5 UI-terminal.
- Multi-Agent Plan Gate: review_gates auto → logged above, auto-dispatched (no halt). 8 slices, ~4 waves.
- Stage 3: complete (8 features). Verification Gate green; invariant lint 10/10 + 4 manual.
- Security Audit Gate (3-pass): review_gates auto → auto-remediated 1 CRITICAL + 1 HIGH + 7 MODERATE dependency findings (drizzle-orm→0.45.2, removed unused drizzle-kit, vite→6, vitest→4.1). Pass 3 re-scan: 0 vulnerabilities. SAST/secret tooling absent → PASS WITH ADVISORY (manual review clean). See SECURITY-AUDIT.md. Patch → 0.1.1.
- COMPLIANCE.md Gate: Posture A 15 / B 0 / C 0 — all requirements met. Auto-proceeded.
- Design Quality Gate: 13/13 screens ✓; Template Conformance 0 token drift. Auto-proceeded.
- Reviewer Gate: PASS (no correctness-class failures; generic scanner; invariants enforced). 3 advisories → ADR-016.
- Stage 4: Verification Gate + startup verification green; continuation scaffold emitted; Deploy Reachability Outcome C (local docker stack).
- Drift Detection Gate (ship): 0 drifts (10/10 machine + 4 manual upheld). Auto-proceeded to deploy.
- Deploy + Functional Smoke: deployed local docker stack; 17/17 Key-Workflow flows pass. Three root-cause defects found by the smoke/live-UI verification were fixed in-place (re-entered Stage 3): Hono onError mounting boundary, undici pinned-lookup {all:true} contract (+regression test), CSS token @import order + CSP font/script. VERSION → 0.1.1-functional-verified.
