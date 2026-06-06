# ARCHITECTURE.md — MCP Sentinel

> Architectural blueprint (Stage 1). Source of truth for **architecture**.
> `SPEC.md` is the source of truth for **behavior**. Build depth: **comprehensive**.
> Derived from `RESEARCH.md` (Stage 0). All architectural rules are enumerated in
> §9 and mirrored machine-checkably in `invariants.json`.

## 1. System overview

MCP Sentinel is a single-tenant-per-user web application that audits the
security posture of Model Context Protocol (MCP) servers. A user submits an MCP
endpoint URL (+ optional, never-persisted auth token); Sentinel connects over
the Streamable HTTP transport, enumerates the server's tools / resources /
prompts, runs a **deterministic rule engine** over those definitions, and
produces a graded report (posture grade A–F + severity-ranked findings, each
with evidence and remediation). Reports are persisted per endpoint so a re-scan
can be diffed against the previous result (new / resolved / changed findings,
including mutated tool definitions = the rug-pull signal).

**Architectural pillars (from RESEARCH §5 threat model):**
1. **SSRF containment is the spine.** Every outbound connection passes through a
   single hardened connector that resolves DNS app-side, validates every
   resolved IP against a deny-list, pins the connection to that IP, and
   re-validates on every redirect hop. No other code path may make an outbound
   request to a user-supplied host. (INV-1, INV-2)
2. **The detection engine is deterministic — no LLM in the analysis path.**
   Poisoned tool descriptions are treated as inert data. (INV-7)
3. **User auth tokens are ephemeral.** Never written to the database, logs, or
   reports; redacted everywhere. (INV-3, INV-4)
4. **Bounded consumption.** Hard time + byte + JSON-depth caps and a ReDoS-safe
   matching strategy protect the scanner from a hostile target. (INV-9)

## 2. Layered architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Frontend  (React 18 + Vite + TypeScript, Tailwind tokens)   │
│  SPA: Scan form · Report view · Findings drill-down ·        │
│  Endpoint history · Diff/compare · Auth (login/register)     │
└───────────────▲─────────────────────────────────────────────┘
                │  REST/JSON over HTTPS  (cookie session)
┌───────────────┴─────────────────────────────────────────────┐
│  Backend  (Hono on Node.js LTS, TypeScript)                  │
│                                                              │
│  ── HTTP layer ──────────────────────────────────────────   │
│  Routes (/api/*) · auth middleware · CSRF/Origin guard ·     │
│  rate limiter · zod request validation · problem+json errors │
│                                                              │
│  ── Service layer ───────────────────────────────────────   │
│  AuthService · ScanService (orchestrates a scan) ·           │
│  ReportService · DiffService · EndpointService               │
│                                                              │
│  ── Scanner core ────────────────────────────────────────   │
│  SafeConnector (SSRF guard + IP pinning)  →  McpClient       │
│  (enumerate via Streamable HTTP)  →  Normalizer  →           │
│  RuleEngine (deterministic detectors)  →  Grader             │
│                                                              │
│  ── Data layer ──────────────────────────────────────────   │
│  Drizzle ORM  →  PostgreSQL                                  │
│  withUserScope() canonical tenant-scoped query wrapper       │
└──────────────────────────────────────────────────────────────┘
```

The frontend and backend are separate packages in an npm workspaces monorepo
(`apps/web`, `apps/api`, `packages/shared`). `packages/shared` holds the zod
schemas and TypeScript types shared across the REST boundary (single source of
contract truth).

## 3. Data flow — a scan

1. `POST /api/scans` `{ url, authToken? }` → auth middleware → rate limiter →
   zod validation (`url` must be `http(s)` per protocol policy).
2. `ScanService` creates an `endpoints` row (find-or-create by normalized URL,
   per user) and a `scans` row (`status='running'`).
3. **SafeConnector** parses the URL, rejects non-`http(s)` schemes, resolves the
   host to IPs (app-side `dns.lookup` all addresses), rejects if *any* resolved
   IP is in a denied range, and produces a pinned dispatcher (undici) bound to
   the validated IP with `Host` header preserved.
4. **McpClient** performs the MCP handshake (`initialize`) then enumerates
   `tools/list`, `resources/list`, `resources/templates/list`, `prompts/list`
   (paginating on `nextCursor`), under hard timeouts and a byte ceiling. SSE and
   `application/json` responses both handled. Falls back to legacy HTTP+SSE probe
   if Streamable HTTP `initialize` is rejected.
5. **Normalizer** zod-validates and canonicalizes the raw JSON-RPC results into a
   stable `ServerInventory` (tools/resources/prompts with sorted keys), and
   computes a content hash per tool definition (for rug-pull diffing).
6. **RuleEngine** runs each detector over the inventory + captured responses.
   Each detector emits zero or more `Finding`s `{ ruleId, severity, title,
   evidence(redacted), explanation, remediation, location }`.
7. **Grader** maps findings → posture grade (A–F) by worst-severity + weighting.
8. **ReportService** persists the report + findings (secrets redacted to
   type+offset before write). `DiffService` compares against the prior report for
   the same endpoint and stores a diff summary.
9. The auth token is discarded — it never leaves the request scope.

## 4. Scanner core — detail

**SafeConnector** (`apps/api/src/scanner/safe-connector.ts`)
- Single chokepoint for all user-host egress. Exposes `createPinnedFetch(url)`.
- Scheme allow-list: `https` always; `http` permitted only because
  `protocol_support: "HTTPS only"` governs *Sentinel's own* serving — targets may
  legitimately be plaintext during dev. (Logged as ADR-004.) Default deny all
  other schemes.
- Resolves all A/AAAA records; denies if any is loopback, private (RFC1918),
  link-local (`169.254.0.0/16`, incl. cloud metadata), ULA (`fc00::/7`),
  link-local v6 (`fe80::/10`), unspecified, or IPv4-mapped-IPv6 of any denied
  range (normalized). Does NOT use the npm `ip` package (CVE-2024-29415).
- Connects via an undici `Agent` with a custom `connect` that pins to the
  validated IP. `maxRedirections: 0` at the transport; redirects surfaced to
  McpClient which re-invokes SafeConnector for the new URL (re-validation per
  hop, max 3 hops).
- Enforces connect timeout (5s) and total deadline (passed from ScanService).

**McpClient** (`apps/api/src/scanner/mcp-client.ts`)
- Uses `@modelcontextprotocol/sdk` v1.x `StreamableHTTPClientTransport` where
  possible, with the pinned fetch injected; a thin direct JSON-RPC path is used
  for enumeration to keep byte/time caps enforceable.
- Caps: total scan deadline default 10s (SPEC perf target); response byte ceiling
  5 MB per call (streamed, abort on exceed); JSON parse guarded by size + depth.
- Captures raw JSON-RPC envelopes (redacted) for the evidence viewer.

**RuleEngine** (`apps/api/src/scanner/rules/`)
- One module per detector; a registry lists them. Detectors are pure functions
  `(inventory, context) => Finding[]`. v1 risk classes:
  - `tool-poisoning` — injection patterns in tool/prompt descriptions (imperative
    override phrases, hidden-instruction markers, invisible/again-Unicode tags,
    data-exfil directives). Matching uses bounded input length + RE2-style
    linear-time matching (`re2` package) — never native backtracking on
    untrusted strings. (INV-9)
  - `excessive-scope` — over-broad permission/tier hints, wildcard scopes,
    filesystem/shell/network tools requesting unbounded access.
  - `secret-leak` — secrets/tokens echoed in tool outputs or resource contents
    (entropy + known-prefix detectors: `sk-`, `ghp_`, AWS AKIA, JWT, etc.).
  - `rug-pull` — tool definition content-hash changed vs the prior scan, or
    server declares `tools.listChanged` without auth (mutable-after-trust).
  - `missing-auth` — sensitive tools (write/delete/exec/exfil signals) reachable
    on an endpoint that returned no `401`/`WWW-Authenticate`/PRM challenge.
  - Plus transport-hygiene checks derived from the spec: missing `Origin`
    validation signal, accepts requests without `MCP-Protocol-Version`.
- Detectors are versioned (`ruleVersion`) so reports record which engine produced
  them.

**Grader** (`apps/api/src/scanner/grader.ts`) — deterministic: any CRITICAL → F;
any HIGH → D; MEDIUM-only → C; LOW-only → B; none → A. Recorded with the report.

## 5. Backend modules & orchestration

- `index.ts` — Hono app bootstrap, middleware chain order (security headers →
  request-id/logger → rate limiter → session → CSRF/Origin → router).
- `middleware/` — `auth` (session cookie → user), `csrf` (Origin + double-submit
  token for state-mutating verbs), `rateLimit` (per-IP + per-user, scan endpoint
  has a tighter cost-based limit), `error` (maps thrown `AppError` → RFC 9457
  `application/problem+json`).
- `services/` — business logic, the only callers of the data layer's scoped
  wrapper.
- `db/` — Drizzle schema, migrations, `withUserScope(userId, fn)` wrapper. **No
  service may import the raw table objects for user-scoped tables except through
  this wrapper.** (INV-5)
- `lib/` — `logger` (pino, structured JSON, token redaction), `crypto`
  (argon2id password hashing, session token gen), `redact` (secret redaction),
  `problem` (problem+json helper).

## 6. Frontend architecture

- React 18 + Vite + TypeScript; React Router; TanStack Query for server state;
  Tailwind configured from the **design-template `:root` tokens** (see §8).
- App shell = saas archetype: persistent left sidebar (224px) + contextual
  header + content region. Theme toggle (light/dark/system) persisted to
  `localStorage`, default follows `prefers-color-scheme`.
- Screens: Login, Register, Dashboard (recent scans + new-scan form),
  Scan-in-progress, Report (posture + findings list), Finding drill-down (with
  raw JSON-RPC viewer), Endpoint history (timeline + grade trend), Compare/diff.
- All MCP-origin strings rendered as escaped text (never `dangerouslySetInnerHTML`)
  to neutralize stored-XSS from poisoned definitions. (INV-8)

## 7. Persistence model (logical — full DDL in SPEC §3)

- `users` (id, email UNIQUE, password_hash, created_at)
- `sessions` (id, user_id, token_hash, expires_at)
- `endpoints` (id, user_id, url, url_normalized, created_at; UNIQUE
  (user_id, url_normalized))
- `scans` (id, endpoint_id, user_id, status, grade, started_at, finished_at,
  duration_ms, error, rule_version)
- `findings` (id, scan_id, rule_id, severity, title, explanation, remediation,
  evidence_redacted, location)
- `inventory_items` (id, scan_id, kind, name, definition_hash, definition_redacted)
- `scan_diffs` (id, scan_id, prev_scan_id, added, resolved, changed) — JSON summaries
- `audit_log` (id, user_id, action, target, ip, created_at) — structured audit trail

All user-scoped tables carry `user_id` and are queried only via `withUserScope`.

## 8. Design system (tokens are binding — from DESIGN-TEMPLATE.html)

The customer uploaded an HTML design template. Per DESIGN.md Part III its `:root`
tokens are copied **verbatim** as the canonical token set and override
`get_project_config` where they conflict. Saved to `DESIGN-TEMPLATE.html`; exact
copied `:root` block recorded in DECISIONS.md (ADR-009). Tokens are emitted once
as CSS custom properties + Tailwind theme and consumed everywhere; hard-coded
colors / off-scale spacing are a defect. (INV-10)

Key tokens (light): `--color-bg:#f6f8fb`, `--color-surface:#ffffff`,
`--color-accent:#2563eb`, fonts Manrope (display) / Inter (body) / JetBrains Mono;
spacing 4–96px scale; radius 6/12/20/999; three-tier shadows. Dark theme via
`[data-theme="dark"]` + `prefers-color-scheme`. Archetype: **saas**.

## 9. Architectural Invariants

Every rule below has a corresponding entry in `invariants.json` (same `id`).

| ID | Rule | Check type | Reference |
|----|------|-----------|-----------|
| INV-1 | All user-host egress goes through `SafeConnector`; no other module calls `fetch`/`undici.request`/`http(s).request` against a user-supplied URL. | forbidden-pattern | §4 SafeConnector |
| INV-2 | The SSRF connector module exists and is the single egress chokepoint. | required-file | §4 |
| INV-3 | User MCP auth tokens are never written to a persistent store: no DB column or migration stores `auth_token`/`authToken`. | forbidden-pattern | §1, §7 |
| INV-4 | No logging call emits a raw token/secret; logger is configured with redaction. | manual | §5 lib/logger |
| INV-5 | User-scoped DB access in the resource services (`*-service.ts`) goes through `withUserScope`. `identity.ts` (pre-auth register/login/session) is exempt by design. | boundary-order | §5, §7 |
| INV-6 | Every user/email-unique table has a UNIQUE constraint on email; idempotency-critical tables have their UNIQUE constraint. | required-unique-constraint | §7 |
| INV-7 | No LLM/model API call exists in the detection path (deterministic engine only). | forbidden-pattern | §1, §4 RuleEngine |
| INV-8 | MCP-origin content is never rendered via `dangerouslySetInnerHTML`. | forbidden-pattern | §6 |
| INV-9 | Detection regexes run on bounded input via a linear-time engine (`re2`); native `RegExp` is not used on untrusted MCP strings in the rules dir. | manual | §4 RuleEngine, RESEARCH T5 |
| INV-10 | Design tokens are defined once (CSS custom properties / Tailwind theme) and the token file exists; screens consume tokens, not ad-hoc hex. | required-file | §8 |
| INV-11 | Every SPEC §UI Surface screen route renders; every non-internal endpoint is UI-reachable; every Key Workflow has an e2e test to its terminal step. | ui-coverage | SPEC §UI Surface |
| INV-12 | Rate limiting is applied to every public endpoint (auth + scan). | manual | §5 middleware/rateLimit |
| INV-13 | State-mutating endpoints enforce an Origin/CSRF guard. | manual | §5 middleware/csrf |
| INV-14 | The MCP `initialize` handshake precedes any `tools/list` / `resources/list` / `prompts/list` enumeration call in the client. | boundary-order | §4 McpClient |

**Invariant count: 14.** Machine-checkable: INV-1, INV-2, INV-3, INV-5, INV-6,
INV-7, INV-8, INV-10, INV-11, INV-14. Manual (prose-audited at Drift Gate):
INV-4, INV-9, INV-12, INV-13.

## 10. Threat model (comprehensive — see RESEARCH §5)

The eight threats T1–T8 and their mitigations are carried forward as binding
design constraints. T1 (SSRF), T2 (bounded consumption), and T5 (ReDoS) are the
non-negotiable, GO-gating controls and are encoded as INV-1/INV-2/INV-9 plus the
caps in §4. T3/T6 (token & data handling) drive INV-3/INV-4 and the redaction
layer. T7 (tenant isolation) drives INV-5. T4/T8 drive INV-7/INV-8 and zod
response validation.

## 11. Key decisions & alternatives (recorded as ADRs in DECISIONS.md)

- ADR-001 Hono on Node (vs Bun) — Node LTS for ecosystem maturity; Hono is
  runtime-portable so a later Bun move is low-cost.
- ADR-002 Drizzle ORM (vs raw pg) — type-safe migrations + retains pg driver.
- ADR-003 Deterministic rule engine (vs LLM-as-judge like Cisco scanner) — avoids
  scanner-prompt-injection (T4), meets the <10s perf target, fully explainable.
- ADR-004 `http` targets allowed (Sentinel itself is HTTPS-only) — targets may be
  plaintext dev servers; SafeConnector still IP-validates.
- ADR-005 IP pinning via custom undici dispatcher (vs npm `ip`) — CVE-2024-29415.
- ADR-006 `re2` for untrusted matching (vs native RegExp) — ReDoS safety (T5).
- ADR-007 Session cookies (vs JWT) — server-side revocation, simpler for small scale.
- ADR-008 argon2id password hashing — NIST 800-63B aligned; HIBP k-anonymity
  breach screening on register.

## 12. Deployment & ops

- Local/prod via `docker-compose.yml`: `db` (Postgres 16), `api` (Hono/Node),
  `web` (static build served by the api or nginx). HTTPS-only policy: production
  terminates TLS at a reverse proxy with HTTP→HTTPS redirect + HSTS; local dev
  uses mkcert (documented in QUICKSTART §Protocol & TLS).
- Header-buffer sizing applied to the serving path.
- Secrets via environment variables only; `.env.example` enumerates every var.
- Structured JSON logs (pino) with request-id correlation; `audit_log` table for
  security-relevant actions (`audit_logging: true`).
