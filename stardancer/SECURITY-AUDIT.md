# STARDANCER — Security Audit

Comprehensive-depth Security Audit Gate (BUILD.md Stage 3). Pass 1 scanned commit b13b14f (0.1.4) with a fresh-context
subagent; full findings, tool output and exploit paths are in `artifacts/security/pass-1.md` (Semgrep, Gitleaks,
`npm audit` JSON alongside). Pass 2 re-scans the 0.1.5 remediation diff: `artifacts/security/pass-2.md`.

## Tools

| Tool | Version | Result (pass 1) |
|---|---|---|
| npm audit | npm 10.9.3 | 11 across the full tree; 4 high with `--omit=dev` (S-1) |
| Semgrep | 1.95.0 (`p/default`, `p/typescript`, `p/nodejs`) | 13 findings, all triaged false positive |
| Gitleaks | v8.21.2 | 2 history hits, both test fixtures (false positive); working tree 0 |
| Live probes | https://localhost:8643 | cookie flags, CSRF/Origin 403, cross-tenant 404 — all held |

## Findings and disposition

| ID | Severity | Finding | Disposition |
|---|---|---|---|
| S-1 | HIGH | `npm audit --omit=dev` showed 4 high (Prisma CLI chain: `deepmerge-ts`, `mysql2`), failing SC-18 | **Remediated 0.1.5** — `prisma` moved to devDependencies (only the migrate stage needs it); root `overrides` pin `deepmerge-ts` 8.0.2 and `mysql2` 3.24.5. `npm audit --omit=dev`: 0. `prisma validate` passes with the overrides. |
| S-2 | MEDIUM | Open redirect after login: `next=/%09/evil.example` survived `safeNext` (confirmed live) | **Remediated 0.1.5** — `web/src/lib/session.ts` refuses control characters and backslashes, resolves the value as the browser will and requires the same origin; `loginHref` uses the same guard. Unit tests cover TAB/LF/CRLF. |
| S-3 | MEDIUM | Baseline fixture seeded the public `demo-reset-token` for the baseline org owner and an analyst `demo-invite-token`, 30-day expiry | **Remediated 0.1.5** — the reset now targets a throwaway account that owns nothing, the invite is viewer-only, both expire 2 h after a run; re-applied to the live stack. |
| S-4 | MEDIUM | SSE concurrency had no per-user cap; one member could hold all 200 org slots per replica | **Remediated 0.1.5** — `MAX_STREAMS_PER_USER = 6` per replica, a new stream closes that user's oldest (`server/src/services/realtime.ts`); test in `server/test/overview.test.ts`. |
| S-5 | LOW | Evidence export 10/h/org vs SPEC SC-5 5/h/org | **Remediated 0.1.5** — 5/h; test asserts the 6th export is 429. |
| S-6 | LOW | SSR session check from the web container counts every user against the web container's IP bucket | Deferred — the check is one cheap call per navigation and the global limit is 600/min; forwarding the client IP needs a trusted internal header contract between web and api (ARCH §4), recorded for the next release. |
| S-7 | LOW | Redis has no authentication | Deferred — in `compose.yaml` Redis is on the internal network with no published port (the separate `compose.dev.yaml` publishes it on 127.0.0.1 only, for development); adding `requirepass` changes every `REDIS_URL` consumer and is recorded for the next release. |
| S-8 | LOW | Third-party containers (postgres, redis, prometheus, grafana) run as root without `cap_drop` | Deferred — upstream images; app services and Caddy are hardened per SC-17. Hardening the vendor images needs per-image volume-ownership work, recorded. |
| S-9 | LOW | No email-ownership verification at signup | Deferred — v1 has no outbound mail: SPEC §3 returns the invite and reset links to the admin once, to hand over out of band, so there is no channel to verify ownership; an invite binds to an existing account by email by design. |
| S-10 | LOW | `.env.example` `change-me` placeholders passed validation | **Remediated 0.1.5** — startup refuses `change-me` in SESSION_SECRET, EVIDENCE_HMAC_KEY, STRIPE_WEBHOOK_SECRET and DATABASE_URL (test in `platform.unit.test.ts`). The Grafana admin password is read by Grafana itself, not the app — deferred, documented in QUICKSTART. |

Pass 1: 0 CRITICAL · 1 HIGH · 3 MEDIUM · 6 LOW. Auto-remediated: 6 (all HIGH/MEDIUM plus S-5, S-10). Deferred: 4 LOW.

## Pass 2

Fresh-context re-scan of the remediation commit d6b3b07 against b13b14f (`artifacts/security/pass-2.md`): Semgrep 0,
Gitleaks 0, `npm audit --omit=dev` 0 (full tree 7, dev and load tooling only, unchanged). Live probes confirmed S-2
(28 bypass attempts held), S-4 (16 streams on one session → 12 open across 2 replicas) and that a stream ends about 2 s
after its session logs out while the user's other session keeps its own.

| ID | Severity | Finding | Disposition |
|---|---|---|---|
| S-3 | LOW (residual) | The viewer invite still lets the holder of the public demo token join the baseline org read-only; each fixture run re-arms it for 2 h | Deferred — the baseline org holds only simulated data, the invite is viewer-only and short-lived, and the web baseline needs a live invite to audit the accept screen. |
| S-10 | LOW (residual) | `GRAFANA_ADMIN_PASSWORD=change-me` is still accepted | Deferred — Grafana reads it, not the app; QUICKSTART tells the operator to set it and Grafana is bound to loopback. |
| N-1 | MEDIUM | `GET /v1/entities` scanned the whole org per request with no cost limit (cross-tenant DoS on a very large org) | **Remediated af9f645** — each 5 000-row chunk read is charged to a per-org budget (240 chunks/min, 429 with Retry-After when spent) and one request reads at most 40 chunks; test in `server/test/entities.list.test.ts`. |
| N-2 | LOW | Each entity-creating ingest batch retries up to 1 000 waiting relationships on the shared workers | Deferred — bounded per batch and per org, and ingest keys are limited to 120 batches/min; targeting retries at the externalIds a batch created is recorded for the next release. |

S-6, S-7, S-8 and S-9 are unchanged and remain LOW (see above).

**Security** — Pass 1 findings: 10 (0 CRITICAL · 1 HIGH · 3 MEDIUM · 6 LOW) · Auto-remediated: 7 (S-1, S-2, S-3→LOW, S-4,
S-5, S-10 partial, N-1) · Pass 2 residual: 0 CRITICAL · 0 HIGH · 0 MEDIUM · 7 LOW (S-3, S-6, S-7, S-8, S-9, S-10, N-2;
all deferred with reasons).
