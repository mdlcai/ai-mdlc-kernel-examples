# COMPLIANCE.md — PatchLAB

Maps every SPEC §4 Security Requirement (SEC-1…15) to implemented, audit-clean code.
Baseline: OWASP ASVS L1/L2 + Top-10. No named regulatory regime (HIPAA/PCI/GDPR/SOC2) was
declared in RESEARCH → only the OWASP floor applies (logged in DECISIONS.md).

Legend: ✅ implemented + audit-clean · ⚠ implemented with open note · ❌ gap.

| ID | Requirement | Status | Evidence | Notes |
|----|-------------|--------|----------|-------|
| SEC-1 | Argon2id password hashing, never plaintext | ✅ | `core/security.py` `hash_password`/`verify_password`; `test_foundation.py::test_password_hash_roundtrip_and_argon2id` | argon2-cffi defaults |
| SEC-2 | Length-based password policy, no composition/rotation (NIST 800-63B) | ✅ | `schemas/auth.py` (min 8, max 200, no rules) | |
| SEC-3 | JWT algorithm pinned, alg=none rejected, secret from env | ✅ | `core/security.py::decode_token`; `test_foundation.py::test_access_token_pins_algorithm_and_rejects_alg_none` | INV-13; prod fail-fast on weak secret (`config.assert_production_safe`) |
| SEC-4 | HttpOnly+Secure+SameSite cookies; refresh rotation + reuse detection | ✅ | `api/v1/auth.py::_set_session_cookies`; `services/auth_service.py::rotate_refresh` | `jti` UNIQUE (INV-6); reuse → revoke-all |
| SEC-5 | Auth anti-enumeration (shape + timing); register leaks nothing | ✅ | `services/auth_service.py` (dummy hash on collision); `api/v1/auth.py::register`; `test_api_auth.py` (3 tests) | H1 remediated: no Set-Cookie/PII/timing oracle |
| SEC-6 | Rate limiting (auth before hash, generate, packs, download) with key dims | ✅ | `core/ratelimit.py`; decorators in `api/v1/*`; `client_ip` hardened | H3 remediated: trusts X-Real-IP/last hop; download per-IP |
| SEC-7 | Object-level authz on every read/mutate/download incl. sub-resources; 404 not 403 | ✅ | `services/sound_service.py::get_owned/get_for_download`; `pack_service.py::get_owned`; `test_api_sounds.py::test_idor_cross_user_returns_404`, `test_api_packs.py::test_pack_ownership_404` | |
| SEC-8 | Path-traversal-safe asset access; opaque UUIDs; realpath confined | ✅ | `core/storage.py::resolve/new_asset_path`; `test_foundation.py::test_storage_resolver_rejects_traversal` | INV-11 |
| SEC-9 | Prompt parsed by allow-list, never eval/exec; params clamped | ✅ | `audio/prompt_parser.py`, `audio/params.py`; `test_audio.py` (parser/clamp/no-eval) | INV-2 |
| SEC-10 | Resource caps (duration/rate/variations/aggregate/pack-bytes) pre-allocation | ✅ | `services/sound_service.py` (aggregate guard), `audio/render.py`, `audio/pack.py`; `test_audio.py::test_render_rejects_oversized_request`, `test_pack_budget_aborts`, `test_api_sounds.py::test_aggregate_sample_cap_enforced` | H2 remediated |
| SEC-11 | Parameterized queries only; no SQL string interpolation | ✅ | `repositories/*.py` (SQLAlchemy select/bound params) | INV-1 |
| SEC-12 | Structured JSON logs redact secrets; no bodies | ✅ | `core/logging.py::RedactFilter` (message + structured fields); `core/errors.py` (500 logged w/ request id) | M4/M5 remediated |
| SEC-13 | HTTPS only: TLS, HTTP→HTTPS, HSTS | ✅ | `infra/nginx/nginx.conf` | INV-14; smoke test runs against https origin |
| SEC-14 | CSRF mitigation on mutating routes (SameSite + X-Requested-With) | ✅ | `api/deps.py::require_csrf`; `frontend/src/lib/api.ts`; `test_api_auth.py::test_csrf_header_required` | |
| SEC-15 | Per-user storage quota + anonymous TTL cleanup | ✅ | `services/sound_service.py::_check_quota`, `_cleanup_expired_anon`, anon TTL `expires_at` + download guard | H4 remediated |

## Posture
- A (✅): 15
- B (⚠): 0
- C (❌): 0

**All SPEC §4 security requirements implemented and audit-clean.** Two LOW audit items (L1 fail-closed bare-except; L4 16 MiB body limit) are accepted defense-in-depth notes, not requirement gaps — tracked in SECURITY-AUDIT.md.
