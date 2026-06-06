# RESEARCH.md — MCP Sentinel

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Security"

## Domain Signals

```yaml
domain_signals: ["has_webhooks"]
```

## Product Vision
**Problem:** ▎ Teams are wiring MCP servers into agents and developer tools
  ▎ without any way to vet what those servers are actually doing. An
  ▎ MCP server's tool descriptions are fed straight into an LLM's
  ▎ context, so a malicious or careless server can smuggle injection
  ▎ instructions, request far more permission than it needs, leak
  ▎ secrets in its responses, or silently change its tool definitions
  ▎ after you've granted trust. These are documented attack classes
  ▎ — tool poisoning, rug pulls, excessive scope — but there's no
  ▎ easy way to check a server for them. Developers either read the
  ▎ raw protocol output by hand or, more often, just trust the
  ▎ endpoint and hope.

**Who it affects:** ▎ Anyone who connects an agent or app to an MCP server they didn't
  ▎ write. Primarily:
  ▎ - Developers and AI engineers integrating third-party or
  ▎ community MCP servers into agents, IDEs, or products — they're
  ▎ the ones who inherit a poisoned tool description or an
  ▎ over-scoped server straight into their LLM's context.
  ▎ - Platform / DevSecOps and AppSec teams responsible for approving
  ▎ what gets connected to internal systems, who currently have no
  ▎ MCP-aware way to vet a server before signing off.
  ▎ - MCP server authors who want to self-audit and ship a server
  ▎ users can trust — running Sentinel on their own endpoint before
  ▎ publishing.
  ▎ - Technical founders and indie builders wiring MCP into a product
  ▎ fast, who need a quick "is this safe to depend on?" check
  ▎ without standing up a security review.

**Why existing solutions fall short:** ▎ Anthropic's official MCP Inspector is a local development tool
  ▎ for poking at a server's tools — it has no security analysis at
  ▎ all, no notion of tool poisoning or excessive scope, and no
  ▎ history. Generic JSON-RPC and Postman-style clients can hit an
  ▎ endpoint but are protocol-agnostic: they show raw envelopes and
  ▎ leave every judgment to the user. Traditional application
  ▎ security scanners (SAST/DAST, dependency scanners) don't
  ▎ understand MCP's threat model — that the danger is in tool
  ▎ descriptions and contracts being fed to an LLM, not in
  ▎ conventional code paths. Nothing on the market answers the
  ▎ question a team actually has: is it safe to connect my agent to
  ▎ this MCP server?

**Solution:** ▎ MCP Sentinel is a web tool that audits the security posture of
  ▎ Model Context Protocol servers. You point it at an MCP endpoint
  ▎ (URL + optional auth token), and it connects, enumerates the
  ▎ server's tools/resources/prompts, and scans their definitions for
  ▎ known MCP risk classes: prompt-injection payloads hidden in tool
  ▎ descriptions ("tool poisoning"), over-broad permission/tier
  ▎ requirements, secrets or tokens echoed back in responses, mutable
  ▎ tool definitions that can change after you trust them ("rug
  ▎ pulls"), and sensitive tools missing authentication. It produces
  ▎ a graded report — findings ranked by severity, each with an
  ▎ explanation and a concrete remediation — and keeps a history so
  ▎ you can re-scan and see whether a server's posture improved or
  ▎ regressed.


## Users & Outcomes
**Key Workflows:**
▎ 1. Scan a server. User pastes an MCP endpoint URL (and optional
  ▎ auth token), clicks Scan. Sentinel connects, enumerates
  ▎ tools/resources/prompts, runs the risk checks, and renders a
  ▎ graded report — overall posture grade plus findings ranked by
  ▎ severity, each with an explanation and a remediation.
  ▎
  ▎ 2. Drill into a finding. User expands a finding to see the exact
  ▎ evidence — the offending tool description, the over-broad
  ▎ permission, the response that echoed a secret — alongside the
  ▎ rule that flagged it and how to fix it. Raw JSON-RPC is one click
  ▎ away for verification.
  ▎
  ▎ 3. Re-scan and compare. User re-runs a scan on a previously
  ▎ inspected endpoint; Sentinel diffs against the last result and
  ▎ highlights what changed — new findings, resolved findings, and
  ▎ any tool definition that mutated since last time (the rug-pull
  ▎ signal).
  ▎
  ▎ 4. Review history. User opens an endpoint's history to see its
  ▎ posture grade over time and the timeline of scans, so a
  ▎ regression is obvious at a glance.
  ▎
  ▎ 5. Self-audit / share a result. A server author scans their own
  ▎ endpoint and exports or links the report as evidence the server
  ▎ is safe to depend on.

**Success Metrics:**
▎ Functional
  ▎ - Connects to any spec-compliant MCP endpoint (HTTP/SSE
  ▎ transport) and enumerates its tools, resources, and prompts
  ▎ without the user writing any client code.
  ▎ - Detects each of the v1 risk classes on a known-bad test server:
  ▎ tool-description prompt injection, over-broad permissions,
  ▎ secrets echoed in responses, mutable/changed tool definitions,
  ▎ and sensitive tools missing auth.
  ▎ - Produces a graded report where every finding has a severity, a
  ▎ plain-language explanation, the offending evidence, and a
  ▎ concrete remediation.
  ▎ - Re-scanning a previously inspected endpoint correctly diffs
  ▎ against the last result and flags new, resolved, and changed
  ▎ findings.
  ▎
  ▎ Quality / non-functional
  ▎ - A first scan of a typical server returns a complete report in
  ▎ under ~10 seconds.
  ▎ - User-supplied auth tokens are never persisted and are redacted
  ▎ everywhere they could surface (logs, stored reports, the raw
  ▎ JSON-RPC viewer).
  ▎ - Zero false positives on a curated known-good ("clean")
  ▎ reference server; zero false negatives on the known-bad test
  ▎ server.
  ▎ - The app itself passes the MDLC comprehensive
  ▎ security/compliance gate with no unresolved high-severity
  ▎ findings.
  ▎ - Graceful, clear handling of unreachable, slow, or non-compliant
  ▎ endpoints — no crashes, no hung scans.

**Non-Goals:**
▎ - Not a runtime firewall or proxy. Sentinel audits and reports;
  ▎ it does not sit inline between the agent and the server, and it
  ▎ does not block, filter, or rewrite live traffic.
  ▎ - No persistent storage of user-supplied auth tokens. Tokens are
  ▎ session-only, used for the active scan, never written to storage,
  ▎ logs, or reports.
  ▎ - Not a generic web/API vulnerability scanner. Scope is MCP
  ▎ contract risk only (tool/resource/prompt definitions and
  ▎ responses) — no SAST/DAST, no port scanning, no dependency-CVE
  ▎ analysis of the host.
  ▎ - No automated exploitation. Sentinel flags injection-prone or
  ▎ over-scoped definitions; it never executes attacks, fuzzes
  ▎ destructively, or invokes tools with malicious intent to "prove"
  ▎ a finding.
  ▎ - No scheduled or continuous monitoring in v1. All scans are
  ▎ user-initiated. Recurring/automated re-scanning and alerting are
  ▎ explicitly out of scope (that's a separate monitoring product).
  ▎ - No team/multi-user collaboration in v1. No orgs, shared
  ▎ workspaces, roles, or per-seat access — single-user scans and
  ▎ history only.
  ▎ - Not an MCP server registry or directory. Sentinel scans a
  ▎ server you give it; it does not host, list, rank, or help you
  ▎ discover servers.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"

# Data & Storage
database_preference: "PostgreSQL"

# Security & Compliance
security_baseline: ["OWASP Top 10"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables"

# Frontend
frontend_framework: "React"

# Backend
backend_framework: "Hono"
api_style: "REST"

# Performance & Quality
performance_requirements: ["first scan completes in under 10 seconds"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "small — under 1k users"
target_platforms: ["web"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, confid▎ Clear, precise, and calmly authoritative. Sentinel speaks like a
  ▎ senior security engineer giving you a straight read: direct about
  ▎ risk, specific about evidence, never alarmist or hype-driven. It
  ▎ explains why something is dangerous in plain language, then
  ▎ tells you exactly how to fix it. Confident but not cocky;
  ▎ technical but never gatekeeping. No fear-mongering, no
  ▎ emoji-laden urgency, no marketing fluff in the product surface —
  ▎ the findings speak for themselves.ent, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #a92fe1 | Accents, badges, highlights |
| Accent | #dcb04f | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #16181d | Headings, body text |
| Text Secondary | #5c6270 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #ac36e2 | Accents, badges, highlights |
| Accent | #ddb255 | Callouts, hover states |
| Background | #090a0c | Page background |
| Surface | #121317 | Cards, elevated containers |
| Text | #eaeaec | Headings, body text |
| Text Secondary | #838995 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

### Typography
- Heading: Manrope (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 14px, scale ratio: 1.2
- Scale: 9.7 / 11.7 / 14 / 16.8 / 20.2 / 24.2 / 29px

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Rounded
- Border radius: 6px (sm: 2px, lg: 10px, xl: 14px)
- Shadows: Subtle — `0 1px 3px rgba(0,0,0,0.1)`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

---

# Research (Stage 0 — comprehensive depth)

> Verification note: Sources fetched and cross-checked June 2026. Load-bearing
> technical claims (JSON-RPC method names, protocol revision, transport
> semantics) were confirmed against the vendor's own spec pages, not from
> training memory. GitHub stats are approximate (as GitHub displays) at fetch.

## §3 Source Categories

### §3.1 Official / vendor documentation

- **MCP spec — current stable revision `2025-11-25`** (a `2026-07-28` revision
  is release-candidate only; target `2025-11-25`). Changelog:
  https://modelcontextprotocol.io/specification/2025-11-25/changelog
- Spec overview — https://modelcontextprotocol.io/specification/2025-11-25
- **Transports (Streamable HTTP + stdio)** —
  https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
- Server → Tools — https://modelcontextprotocol.io/specification/2025-11-25/server/tools
- Server → Resources — https://modelcontextprotocol.io/specification/2025-11-25/server/resources
- Server → Prompts — https://modelcontextprotocol.io/specification/2025-11-25/server/prompts
- Lifecycle / initialize — https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle
- **Security Best Practices** —
  https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices
- Authorization (OAuth 2.1) —
  https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
- Official TS SDK — https://github.com/modelcontextprotocol/typescript-sdk
  (client guide: `docs/client.md`)
- JSON-RPC 2.0 — https://www.jsonrpc.org/specification

**VERIFIED JSON-RPC methods Sentinel issues for enumeration:** `initialize`
(handshake → capabilities + optional `MCP-Session-Id` header), `tools/list`
(paginated via `cursor`/`nextCursor`), `resources/list`,
`resources/templates/list`, `prompts/list`. Mutation signal for rug-pull
detection: `notifications/tools/list_changed` (emitted when `tools.listChanged`
capability is declared).

**VERIFIED Streamable HTTP transport (2025-11-25):** server MUST expose a single
MCP endpoint path supporting POST and GET. Client MUST send
`Accept: application/json, text/event-stream`. POST responses are either
`application/json` (one object) or `text/event-stream` (SSE) — client must
support both. `MCP-Protocol-Version: <version>` header MUST be sent post-init
(absent → server assumes `2025-03-26`). Spec requires servers to validate
`Origin` (DNS-rebinding defense, 403 on invalid) and SHOULD bind to localhost
when local — these become Sentinel detection rules. The old two-endpoint
HTTP+SSE transport (`2024-11-05`) is deprecated; back-compat probing is
documented (Sentinel supports fallback).

**VERIFIED JSON-RPC error codes:** `-32700` parse, `-32600` invalid request,
`-32601` method not found, `-32602` invalid params, `-32603` internal;
`-32000..-32099` server-defined. MCP distinguishes protocol errors (JSON-RPC
`error`) from tool-execution errors (`result.isError: true`).

### §3.2 GitHub repos

| Repo | Stars | Last active | License | Note |
|---|---|---|---|---|
| modelcontextprotocol/servers | ~86.7k | Jan 2026 | Apache-2.0 (legacy MIT) | Reference servers — corpus of real tool defs for rule tuning. |
| modelcontextprotocol/typescript-sdk | ~12.6k | v1.29.0 Mar 2026 | Apache-2.0/MIT | **Use v1.x** (`@modelcontextprotocol/sdk`); `StreamableHTTPClientTransport`. v2 pre-alpha. |
| modelcontextprotocol/inspector | ~9.7k | active 2026 | MIT/Apache-2.0 | Official debug UI — closest "competitor" (no security analysis). |
| snyk/agent-scan (ex invariantlabs-ai/mcp-scan) | ~2.5k | v0.5.8 Jun 2026 | Apache-2.0 | Tool poisoning, rug pulls, cross-origin escalation. Local-config oriented. |
| riseandignite/mcp-shield | ~556 | active 2026 | MIT | CLI scanner: tool poisoning, exfil channels. |
| cisco-ai-defense/mcp-scanner | smaller | active 2026 | OSS | Multi-engine (YARA + LLM judge). |
| invariantlabs-ai/mcp-injection-experiments | — | 2025 | OSS | PoC tool-poisoning fixtures — test corpus. |
| colinhacks/zod | ~38k | active | MIT | Validation. |
| vitest-dev/vitest | ~16.6k | active | MIT | Test framework. |
| drizzle-team/drizzle-orm | ~34k | active | Apache-2.0 | ORM. |

### §3.3 Video / tutorials

- Simon Willison — "My Lethal Trifecta talk" (annotated slides/notes) —
  https://simonwillison.net/2025/Aug/9/bay-area-ai/
- Heavybit Generationship Ep.39 — Simon Willison, "I Coined Prompt Injection" —
  https://www.heavybit.com/library/podcasts/generationship/ep-39-simon-willison-i-coined-prompt-injection
- HiddenLayer — "The Lethal Trifecta and How to Defend Against It" —
  https://www.hiddenlayer.com/research/the-lethal-trifecta-and-how-to-defend-against-it
- *(Gap: no single canonical recorded talk specifically on MCP tool poisoning — see §6.)*

### §3.4 Articles — attack classes (authoritative)

- **Invariant Labs — "Tool Poisoning Attacks"** (origin of the term, Apr 2025) —
  https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks
- Invariant — MCP injection experiments (rug-pull/exfil PoC) —
  https://github.com/invariantlabs-ai/mcp-injection-experiments
- **Simon Willison — "The lethal trifecta for AI agents"** —
  https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- **Trail of Bits — "Jumping the line"** (line-jumping: description injection
  fires before any tool call) —
  https://blog.trailofbits.com/2025/04/21/jumping-the-line-how-mcp-servers-can-attack-you-before-you-ever-use-them/
- Trail of Bits — "Insecure credential storage plagues MCP" —
  https://blog.trailofbits.com/2025/04/30/insecure-credential-storage-plagues-mcp/
- Trail of Bits — "We built the security layer MCP always needed" (TOFU pinning,
  rug-pull defense) —
  https://blog.trailofbits.com/2025/07/28/we-built-the-security-layer-mcp-always-needed/
- Snyk — "Building Secure MCP Servers" — https://snyk.io/articles/building-secure-mcp-servers/
- CyberArk — "Poison everywhere: No output from your MCP server is safe" —
  https://www.cyberark.com/resources/threat-research-blog/poison-everywhere-no-output-from-your-mcp-server-is-safe
- Authzed — "A Timeline of MCP Security Breaches" — https://authzed.com/blog/timeline-mcp-breaches
- arXiv — "Systematic Analysis of MCP Security" — https://arxiv.org/html/2508.12538v1

### §3.5 Standards / RFCs

- JSON-RPC 2.0 — https://www.jsonrpc.org/specification
- RFC 9457 (obsoletes RFC 7807) — Problem Details for HTTP APIs (`problem+json`)
  — https://www.rfc-editor.org/rfc/rfc9457.html (target 9457)
- OWASP Top 10 (2021) — A10 SSRF — https://owasp.org/Top10/
- OWASP Top 10 for LLM Applications (2025) — LLM01 Prompt Injection —
  https://genai.owasp.org/llm-top-10/
- **OWASP MCP Top 10 (2025, beta)** — most directly applicable —
  https://owasp.org/www-project-mcp-top-10/ (MCP01 Token/Secret Exposure, MCP02
  Excessive Permission Scope, MCP03 Tool Poisoning, MCP07 Insufficient Auth/Authz,
  MCP10 Context Injection & Over-Sharing)
- OAuth 2.1 (MCP auth basis; PKCE S256, RFC 9728 PRM, RFC 8707 resource
  indicators) — https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- NIST SSDF SP 800-218 — https://csrc.nist.gov/pubs/sp/800/218/final

### §3.6 Competing products — and where each falls short

- **Anthropic MCP Inspector** — interactive debug UI; *no* security rule engine,
  scoring, persistence, or SSRF-guarded scanning. A developer console, not an auditor.
- **Generic JSON-RPC / Postman / curl** — transport only; zero MCP semantics or detection logic.
- **mcp-scan / snyk/agent-scan** — strong but oriented to *locally installed* MCP
  configs and partly LLM-as-judge. Sentinel = remote endpoint, web-delivered,
  deterministic rules, persisted multi-tenant reports.
- **MCP-Shield** — CLI, overlapping detections, no web UI / DB history, smaller rule set.
- **Cisco mcp-scanner** — multi-engine incl. LLM judge; heavier, infra-dependent.
- **Sentinel's wedge:** remote HTTP/SSE endpoint audit + deterministic (non-LLM)
  pattern rules + persisted shareable reports + SSRF-hardened connector. No
  competitor combines all four.

### §3.7 Community threads

- HN — "The 'S' in MCP Stands for Security" — https://news.ycombinator.com/item?id=43600192
- GitHub — Cline issue: "MCP server security scanning before connection"
  (demand signal for Sentinel's exact use case) — https://github.com/cline/cline/issues/9786
- GitHub — incomplete `Accept` header breaks spec-compliant servers —
  https://github.com/OfficeDev/microsoft-365-agents-toolkit/issues/15421
- MCP spec releases / SEP tracker — https://github.com/modelcontextprotocol/modelcontextprotocol/releases

### §3.8 APIs / integrations

- **Hono** — https://hono.dev/docs/ · repo https://github.com/honojs/hono
- **hono-rate-limiter** — https://github.com/rhinobase/hono-rate-limiter (windowMs/limit, Memory/Redis store)
- **node-postgres (pg)** — https://node-postgres.com/
- **Drizzle ORM (Postgres)** — https://orm.drizzle.team/docs/get-started-postgresql (uses pg driver)
- **MCP TS SDK client** — `StreamableHTTPClientTransport` from
  `@modelcontextprotocol/sdk/client/streamableHttp.js`. **Caveat:** the SDK
  transport will NOT honor an IP allow/deny-list — Sentinel must validate the
  resolved IP itself around the fetch (see §5 T1).
- **HaveIBeenPwned Pwned Passwords Range (k-anonymity)** —
  `GET https://api.pwnedpasswords.com/range/{first5SHA1}` (only first 5 SHA-1
  hex chars leave the client; send `Add-Padding: true`) — https://haveibeenpwned.com/API/v3
- **zod** — https://github.com/colinhacks/zod

### §3.9 Patterns

- **SSRF prevention (CRITICAL — Sentinel dials arbitrary user URLs):** OWASP SSRF
  Prevention Cheat Sheet —
  https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
  Block at the *resolved-IP* layer (not URL string): deny `127.0.0.0/8`,
  `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16` (incl.
  `169.254.169.254` cloud metadata), `::1/128`, `fc00::/7`, `fe80::/10`,
  IPv4-mapped IPv6 (`::ffff:a9fe:a9fe`). Resolve DNS app-side, validate every
  resolved address, connect to the pinned IP (defeats DNS-rebinding/TOCTOU),
  reject redirects to private IPs, re-validate each hop. Avoid npm `ip` package
  (CVE-2024-29415 misclassifies private IPs). Normalize IPv6-mapped addresses.
- **Allow/deny-list outbound** — allowlist schemes (`https`, optionally `http`)
  and ports; deny everything else.
- **Secret redaction / structured logging** — pino `redact` paths; never log the
  user's MCP auth token; store finding *type* + offset, never the secret value.
- **Hono rate limiting** — per-IP/per-key windows to blunt scan-abuse / DoS amplification.

## §4 Stack candidates (confirmed)

- **Confirmed (all current, June 2026):** React (frontend) + Hono (Node/TS
  backend) + PostgreSQL + REST.
- **Runtime:** Node.js LTS (Hono also runs on Bun/Deno/Workers; Node is the safe default).
- **ORM:** **Drizzle ORM** (TS-first, type-safe migrations, lightweight; uses pg
  driver) over raw pg.
- **Test framework:** **Vitest** (fits Vite/TS toolchain; unit + integration).
- **MCP client:** official `@modelcontextprotocol/sdk` **v1.x**
  (`StreamableHTTPClientTransport`), wrapped with an SSRF-validating fetch/agent.
- **Validation:** zod for REST bodies + (untrusted) MCP-response parsing.

## §5 Risk register + Threat Model

**Trust boundary:** Sentinel accepts an arbitrary user-supplied URL, connects
outbound to it, and parses attacker-controllable JSON (tool/resource/prompt
definitions and responses). The endpoint, its DNS, its responses, and any pasted
auth token are all untrusted. The detection engine is **deterministic pattern
rules, not an LLM** — this materially shrinks the prompt-injection surface (the
scanner cannot be "instructed" by poisoned content) but does not eliminate ReDoS
or data-handling risk.

| # | Threat | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| T1 | **SSRF** via user URL → internal services / cloud metadata `169.254.169.254` / localhost / RFC1918; DNS-rebinding (TOCTOU), redirect-to-private, IPv4-mapped-IPv6 bypass | High | **Critical** | Resolve DNS app-side; validate *every resolved IP* against deny-list; **connect to the pinned IP**; block non-http(s) schemes & odd ports; reject private-IP redirects and re-validate each hop; avoid npm `ip`; normalize IPv6. **GO is conditional on this — Stage-1 invariant INV-1/INV-2.** |
| T2 | **DoS via slow/huge responses** (slowloris SSE, multi-GB body, deep JSON, decompression bomb) | High | Medium–High | Hard connect + total timeouts; cap response bytes (stream w/ ceiling, abort on exceed); cap JSON depth/size pre-parse; bound decompression; per-user concurrency + rate limiting; bounded worker; SSE read deadline. |
| T3 | **Auth-token leakage** (user-pasted MCP token persisted/logged/echoed, or sent cross-host after redirect) | Medium | High | **Never persist** — ephemeral per-scan only; never log (redact); never attach across cross-host redirect; scope to validated target; redact from reports/responses. |
| T4 | **Scanner prompt-injection** (poisoned descriptions trying to manipulate the analyzer) | Low (by design) | Low–Med | Deterministic rules, no LLM in detection path → poisoned content is inert data. Residual: escape all MCP strings on React render (XSS), CSP. |
| T5 | **ReDoS in detection regexes** (crafted description → catastrophic backtracking in Sentinel's own rules) | Medium | High | Audit every detection regex (no nested quantifiers on overlapping classes); linear-time engine (RE2) for untrusted matching; per-rule input-length caps; per-match timeout/isolation; fuzz rules in CI. |
| T6 | **Storage of sensitive scan data** (results may capture echoed secrets, internal hostnames) | Medium | High | Redact detected secrets before persistence (store finding *type* + offset, not value); least-privilege DB role; per-tenant isolation; retention/expiry + delete API; HIBP k-anonymity so secrets never leave in full. |
| T7 | **Confused-deputy / token passthrough / tenant isolation failure** | Low–Med | High | No ambient credentials on egress; strict per-request token binding; per-tenant auth on REST API. |
| T8 | **Malicious response parsing** (malformed JSON-RPC, oversized arrays, unexpected types) | Medium | Medium | zod-validate all MCP responses before use; defensive parsing; never eval/dynamic-require from a response. |

Framework mapping: T1↔OWASP A10 SSRF; T3/T6↔OWASP MCP01 + LLM02; T2/T5↔OWASP
LLM10 Unbounded Consumption; T4↔LLM01 / MCP03; T7↔MCP confused-deputy.

## §6 Research gaps

1. No single authoritative *recorded* talk specifically on MCP tool poisoning;
   OWASP MCP Top 10 + Invariant/ToB writeups substitute.
2. Invariant→Snyk transition: confirm original Invariant blog posts stay live or
   migrate; update citations if relocated.
3. OWASP MCP Top 10 is beta/living — re-pull before locking rule→framework mapping.
4. RFC 7807 vs 9457 — targeting 9457 (obsoletes 7807).
5. MCP TS SDK v1→v2 (Q3 2026) may change client/transport API — pin v1.x, track v2.
6. No off-the-shelf SSRF-safe outbound for the MCP SDK — Sentinel must own this
   (custom IP-pinning around fetch) with a dedicated security test suite.
7. `2026-07-28` spec RC exists — target `2025-11-25` stable for the build window.

## §7 Summary + GO/NO-GO

**Summary.** MCP is a real, fast-moving standard with a clearly-versioned spec
(`2025-11-25` stable; method names and Streamable HTTP semantics verified above),
a well-documented attack surface (tool poisoning, rug pulls, line jumping,
missing auth, over-broad scope, secret leakage), and authoritative coverage from
Invariant/Snyk, Trail of Bits, Simon Willison, and OWASP. The proposed stack
(React + Hono + PostgreSQL + REST, with Drizzle, Vitest, zod, official TS SDK
v1.x) is current and appropriate. Existing tools validate demand but leave a
clear gap: a hosted, remote-endpoint, deterministic-rule, report-persisting
auditor — Sentinel's niche.

**Decision: GO (conditional).** The single gating condition is **T1 (SSRF)**:
because Sentinel's core function is dialing arbitrary user-supplied URLs, an
SSRF-hardened outbound connector (app-side DNS resolution + resolved-IP
deny-listing + IP pinning + redirect re-validation) MUST be a Stage-1
architectural invariant, paired with response-size/time caps (T2) and a
ReDoS-safe matching strategy (T5). With those three controls treated as
non-negotiable invariants, proceed.
