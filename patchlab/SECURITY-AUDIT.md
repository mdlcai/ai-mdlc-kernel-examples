# SECURITY-AUDIT.md — PatchLAB

**Build:** 0.1.1 · **Date:** 2026-06-19 · **Depth:** comprehensive · **Gate:** Stage 3 Security Audit (3-pass)
**Method:** automated tooling (pip-audit, bandit, npm audit) + a fresh-context adversarial code review against SPEC §4 (SEC-1…15) and OWASP ASVS/Top-10.

## Trust boundaries
1. **Internet → nginx** — TLS termination, HTTP→HTTPS 301, HSTS, security headers, large header buffers. Only nginx is internet-facing.
2. **nginx → FastAPI `/v1`,`/api`** — internal network; API trusts `X-Real-IP`/last XFF hop from nginx only (`TRUSTED_PROXY`).
3. **FastAPI → PostgreSQL** — parameterized queries only (no string SQL); credentials from env.
4. **FastAPI → asset store (disk volume)** — every path built from server UUIDs via a traversal-safe resolver; no client path input.
5. **Principals:** anonymous (rate-limited, capped, TTL capability assets, no library) vs authenticated (JWT cookie, owns library/packs).

## Data-flow risks & controls
- **Prompt → synth params:** untrusted text parsed by a static allow-list; unknown tokens ignored; **no eval/exec**; all params clamped (NaN/Inf rejected). (SEC-9, INV-2)
- **Generation → render:** aggregate sample budget (`variations × duration × rate × channels`) enforced **before allocation**; per-variation cap too. (SEC-10, H2)
- **Pack assembly → ZIP:** running uncompressed-byte budget aborts before overflow. (SEC-10)
- **Asset retrieval:** ownership enforced; cross-owner → 404; anon capability assets expire (TTL) and are cleaned up. (SEC-7, SEC-15)

## AuthN/Z edge cases reviewed
- JWT: algorithm pinned, `alg=none` rejected, `exp/iat/sub` required, token `type` checked, secret from config (fail-fast in prod). (SEC-3, INV-13)
- Sessions: HttpOnly+Secure+SameSite=Lax cookies; refresh rotation with `jti` UNIQUE + **reuse detection revokes all**. (SEC-4)
- Anti-enumeration: login identical envelope + dummy-hash timing equalization; **register issues no cookie, runs a real hash on collision, echoes only the submitted display_name** (no Set-Cookie / PII / timing oracle). (SEC-5)
- CSRF: SameSite=Lax + mandatory `X-Requested-With` on mutating verbs. (SEC-14)
- IDOR matrix: B cannot read/download/refine/preset/delete A's sound or pack (tested). (SEC-7)

---

## Pass 1 — Initial scan (findings)

| ID | Sev | SEC/OWASP | Finding | File |
|----|-----|-----------|---------|------|
| H1 | HIGH | SEC-5 / API3 | Register anti-enumeration broken (Set-Cookie oracle, stored display_name leak, no-hash timing) | api/v1/auth.py, services/auth_service.py |
| H2 | HIGH | SEC-10 / A05 | Per-request **aggregate** sample cap not enforced (only per-variation) | services/sound_service.py |
| H3 | HIGH | SEC-6 / A04 | Rate-limit IP key used spoofable first XFF entry | core/ratelimit.py |
| H4 | HIGH | SEC-15 / A04 | Anon asset TTL never enforced; no cleanup; anon path unquota'd | services/sound_service.py |
| DEP1 | HIGH | A06 | `python-multipart` 0.0.20 — 6 CVEs | requirements.txt |
| DEP2 | MED | A06 | `pydantic-settings` 2.14.1 — GHSA-4xgf-cpjx-pc3j | requirements.txt |
| M1 | MED | A03 | `sample_rate`/`bit_depth` unvalidated at schema boundary | schemas/sound.py |
| M2 | MED | SEC-3 / A02 | Default/weak JWT secret allows silent insecure boot | core/config.py |
| M3 | MED | SEC-6 | Download rate-limit not keyed per-IP as specified | api/v1/sounds.py, packs.py |
| M4 | MED | SEC-12 | Log redaction shallow (structured `extra` not scrubbed) | core/logging.py |
| M5 | MED | SEC-12 | Unhandled 500 not logged with request id | core/errors.py |
| M6 | MED | A05 | CORS `allow_headers=["*"]` with credentials | main.py |
| L1 | LOW | — | `verify_password` bare `except Exception` | core/security.py |
| L2 | LOW | — | refine ignores parent's anon origin caps (owner-only; benign) | services/sound_service.py |
| L3 | LOW | — | health `or True` defeats asset-store probe | main.py |
| L4 | LOW | — | nginx `client_max_body_size 16m` (generous, bounded) | nginx.conf |
| L5 | LOW | — | logout cookie-clear (verified correct) | api/v1/auth.py |

Tooling: **bandit** — 0 findings (exit 0). **pip-audit** — 7 vulns in 2 packages (DEP1/DEP2). **npm audit** — clean at build.

## Pass 2 — Auto-remediation (applied, no user prompt)

- **H1** → register no longer auto-logins; identical 201 shape with no Set-Cookie either path; real Argon2 hash on collision (timing); echoes submitted display_name only. Frontend `authStore.register` now logs in explicitly afterward.
- **H2** → aggregate `n × duration × rate × channels` checked against `MAX_TOTAL_SAMPLES` before any render. Regression test added.
- **H3** → `client_ip()` now trusts `X-Real-IP` (nginx-set) or the **last** XFF hop, never the spoofable first entry.
- **H4** → expired anon capability URLs return 404; opportunistic `_cleanup_expired_anon()` runs on anon generate (deletes expired rows + files).
- **DEP1** → `python-multipart` **removed** (no form parsing used) — eliminates all 6 CVEs. **DEP2** → `pydantic-settings` bumped to 2.14.2.
- **M1** → `sample_rate: Literal[44100,48000]`, `bit_depth: Literal[16,24]` at the schema (422 on bad input). Test added.
- **M2** → `Settings.assert_production_safe()` fails fast in production on weak/default JWT secret or SQLite DB.
- **M3** → both download routes pinned to `key_func=client_ip`.
- **M4** → `RedactFilter` now scrubs structured `extra={}` keys, not just the message.
- **M5** → catch-all handler `logger.exception(...)` with request id.
- **M6** → CORS `allow_headers` and `allow_methods` enumerated explicitly.
- **L3** → health asset-store probe fixed.
- VERSION bumped 0.1.0 → 0.1.1.

## Pass 3 — Re-scan (residual)

- **Re-ran:** pytest **33 passed**, ruff **0**, pip-audit **0 vulns**, bandit **0**, invariant-lint **12/12** machine-checkable. Frontend `tsc --noEmit` 0.
- **Residual CRITICAL: 0 · HIGH: 0 · MEDIUM: 0.**
- **Deferred (LOW, accepted):** L1 (fail-closed bare-except — acceptable, logs nothing sensitive), L2 (refine caps — owner-only, benign), L4 (16 MiB body limit — generous but bounded; JSON bodies are tiny). Tracked here.
- No new CRITICAL/HIGH introduced by remediation.

## Verdict
**PASS.** Zero CRITICAL/HIGH/MEDIUM residual; all MEDIUM+ dispositioned FIXED; LOW deferred with rationale; full suite green; standalone audit file present (comprehensive-depth requirement).
