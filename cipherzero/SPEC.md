# SPEC.md — CIPHΞRZER0 // TRUST

Version 0.1.0 · Stage 2 · 2026-10-14 · build_depth comprehensive · archetype saas · amended Re-entry R-2 2026-11-10 (ADR-083, ADR-084) · Re-entry R-3 2026-11-11 (ADR-089)

## 1. Overview & scope

CIPHΞRZER0 is a multi-tenant cryptographic trust plane, delivered as a Next.js console, an Express REST gateway, a gRPC trust-core and a BullMQ worker. See ARCHITECTURE.md for the system design.

**In scope:**

| Module | What it does |
|---|---|
| COMMAND CENTER | Dashboard |
| CORE | Key lifecycle |
| VAULT | Protect / decrypt |
| TRUST | Signed policy bundles and simulation |
| Workloads & Agents | Identities, credentials and short-lived grants |
| Approvals | Separation of duties |
| EVIDENCE | Hash-chained audit, checkpoints, verify, export |
| QUANTUM | Inventory and hybrid migration; scanned external assets, risk score, CBOM export (R-2); monitored domains and clientless external discovery (R-3) |
| SCANNER (`apps/scanner`, R-2) | Customer-side CLI that discovers cryptography in use and uploads metadata only |
| PROBER (`apps/prober`, R-3) | Isolated egress service: DNS TXT verification, CT search and TLS handshakes against verified public hosts (ADR-089) |
| APEX | Analysis and proposals |
| Operations | Jobs and DLQ |
| SETTINGS | Profile, MFA, team, tenant, billing, plan usage (R-2) |

The public landing page is ported from DESIGN-TEMPLATE.html.

**Product modes (ADR-083).** `PRODUCT_MODE` selects what one build serves:

| Mode | Serves | Switched off |
|---|---|---|
| `platform` (default) | Everything in this spec | — |
| `readiness` | QUANTUM (scanned assets, plans, CBOM), SCANNER ingest, EVIDENCE, Approvals, APEX, Operations, SETTINGS, COMMAND CENTER | VAULT, customer key create/rotate/disable/enable/destroy, agent grants. Their routes answer 404 `NOT_FOUND`, and the console hides them. The tenant signing key still signs bundles and checkpoints |

In `readiness` mode no customer key material or plaintext enters the system.

**Plan tiers (ADR-084).** A tenant with no `active` or `past_due` subscription is on **Free**. Limits and features
per plan are defined in `packages/core/src/entitlements.ts` and enforced by the API (§4, §7 `PLAN_LIMIT`).

**Scope expansion (comprehensive).** Each item is logged as an ADR tagged `scope expansion (comprehensive)`:
- team invites and role management (ADR-019)
- operations console for jobs and the DLQ (ADR-020)
- evidence export plus checkpoint verification (ADR-021)
- the analytics dashboard (ADR-022)
- billing self-service (cancel and refunds) (ADR-023)
- policy simulation (ADR-024)

**Out of scope:**
- real HSM drivers (provider interface only)
- multi-region
- FIPS-validated operation
- email delivery (invites show a copyable link)
- password reset by email (an owner or admin can reset a member's password; ADR-025)
- cloud KMS discovery and source-code scanning in the scanner (R-2, deferred; cloud connectors planned for R-5, a
  repository app for R-4)
- STARTTLS probing, raw CT log monitoring and sub-daily external monitoring (R-3, deferred)
- per-plan audit retention (R-2, deferred; ADR-084)

## 2. Data model

All ids are UUIDv7 strings. Every tenant-owned table has `tenant_id` and indexes led by `(tenant_id, …)`, plus `created_at` and `updated_at` timestamps in UTC. Prisma models map to snake_case tables.

| Entity (table) | Fields (abridged) | Constraints |
|---|---|---|
| Tenant (`tenants`) | id, name, slug, retention_days | slug UNIQUE |
| User (`users`) | id, tenant_id, email_enc, email_hash, name_enc, password_hash, role, totp_secret_enc, totp_enabled, failed_logins, locked_until | email_hash UNIQUE; role ∈ owner, admin, security_officer, auditor, developer |
| Session (`sessions`) | id, user_id, tenant_id, token_hash, csrf_hash, mfa_at, expires_at, ip, ua | token_hash UNIQUE |
| AuthEvent (`auth_events`) | tenant_id, user_id?, kind, ip, success | — |
| Invite (`invites`) | tenant_id, email_hash, role, token_hash, expires_at, accepted_at | token_hash UNIQUE |
| Workload (`workloads`) | tenant_id, name, kind (service, agent, observer — ADR-080, scanner — ADR-083), spiffe_id, status | (tenant_id, name) UNIQUE |
| WorkloadCredential (`workload_credentials`) | workload_id, prefix, token_hash, revoked_at, last_used_at | token_hash UNIQUE |
| AccessGrant (`access_grants`) | tenant_id, workload_id, actions[], key_ids[], purpose, token_hash, status, expires_at, claimed_at, issued_by | token_hash UNIQUE |
| Key (`keys`) | tenant_id, name, purpose (encrypt, sign), profile, status, rotation_days, next_rotation_at, claimed_at, system | (tenant_id, name) UNIQUE |
| KeyVersion (`key_versions`) | key_id, tenant_id, version, status, wrapped_material, public_key, invocation_count | (key_id, version) UNIQUE |
| ProtectedObject (`protected_objects`) | tenant_id, key_id, key_version, label, classification, purpose, ciphertext, nonce, tag, wrapped_dek, kem_ct, digest, rewrapped_at, created_by | — |
| PolicyBundle (`policy_bundles`) | tenant_id, version, status, document (JSON), digest, sig_ed25519, sig_mldsa, author_id, approver_id, activated_at | (tenant_id, version) UNIQUE |
| PolicyDecision (`policy_decisions`) | tenant_id, bundle_version, principal, action, resource, effect, reason, rule_id, latency_us | — |
| CryptoOperation (`crypto_operations`) | tenant_id, op, key_id, key_version, principal_type, principal_id, purpose, result, latency_us, request_id | — |
| AuditHead (`audit_heads`) | tenant_id PK, seq, hash | — |
| AuditEvent (`audit_events`) | tenant_id, seq, at, actor, action, target, data (JSON), prev_hash, hash | (tenant_id, seq) UNIQUE |
| AuditCheckpoint (`audit_checkpoints`) | tenant_id, from_seq, to_seq, root, sig_ed25519, sig_mldsa | (tenant_id, to_seq) UNIQUE |
| ApprovalRequest (`approval_requests`) | tenant_id, kind, target_id, payload, status, requested_by, approved_by, decided_at, expires_at | CHECK approved_by <> requested_by |
| ApexFinding (`apex_findings`) | tenant_id, fingerprint, detector, severity, score, title, evidence (JSON ids), status | (tenant_id, fingerprint) UNIQUE |
| ApexProposal (`apex_proposals`) | tenant_id, finding_id, action, params, status, approval_id, executed_at, validation | — |
| CryptoAsset (`crypto_assets`) | tenant_id, fingerprint, kind, algorithm, quantum_vulnerable, source, key_id?, last_seen_at; R-2: locator?, key_size?, protocol?, not_after?, risk_score INT 0–100, scan_report_id? | (tenant_id, fingerprint) UNIQUE |
| ScanReport (`scan_reports`, R-2) | tenant_id, workload_id (R-3: nullable), scan_id, scanner_name, scanner_version, started_at, finished_at, finding_count, accepted_count, rejected_count, scope[], digest; R-3: origin ∈ cz-scan, external; domain_id? | (tenant_id, scan_id) UNIQUE |
| MonitoredDomain (`monitored_domains`, R-3) | tenant_id, name (IDNA, lower case), status, verification_token, ports INT[] (allowlist), verified_at?, last_checked_at?, consecutive_misses INT, last_scan_at?, removed_at? | (tenant_id, name) UNIQUE |
| ExternalHost (`external_hosts`, R-3) | tenant_id, domain_id, hostname, origin ∈ apex, ct, manual, status ∈ pending, reachable, unreachable, private, wildcard, first_seen_at, last_seen_at, last_probe_at? | (tenant_id, hostname) UNIQUE |
| MigrationPlan (`migration_plans`) | tenant_id, name, target_profile, asset_ids[], status, phase, interop_result; R-2: asset_source ∈ managed, scanned, migrating_at? | — |
| Plan (`plans`) | code, name, amount_minor BIGINT, currency CHAR(3), interval | code UNIQUE |
| Subscription (`subscriptions`) | tenant_id, plan_code, status, current_period_end | — |
| Payment (`payments`) | tenant_id, subscription_id, amount_minor BIGINT, currency CHAR(3), status, provider_ref, checkout_id | provider_ref UNIQUE |
| IdempotencyKey (`idempotency_keys`) | tenant_id, key, request_hash, response_status, response_body, created_at | (tenant_id, key) UNIQUE |
| WebhookEvent (`webhook_events`) | event_id, type, payload, received_at, claimed_at, processed_at | event_id UNIQUE |
| JobRun (`job_runs`) | job, tenant_id ('*' for global), window_start, status, processed, skipped, failed, error, started_at, finished_at | (job, tenant_id, window_start) UNIQUE |

### 2.1 State machines

Each status a transition admits is written by a writer surface (invariants INV-20..INV-49).

| Entity | Transitions |
|---|---|
| Payment | `pending → succeeded \| failed`; `succeeded → refunded \| disputed \| charged_back`; `disputed → succeeded \| charged_back` (webhook processor only) |
| Subscription | `pending → active`; `active → past_due \| canceled`; `past_due → active \| canceled` |
| PolicyBundle | `draft → pending_approval → active → superseded`; `pending_approval → rejected` |
| Key | `active ⇄ disabled`; `active \| disabled → destroyed` (via approval) |
| KeyVersion | `active → deactivated` (rotation) `→ retired` (zero dependents); `* → destroyed` |
| Workload | `active ⇄ suspended` |
| AccessGrant | `active → revoked \| expired` |
| ApprovalRequest | `pending → approved → executed`; `pending → rejected \| expired` |
| ApexFinding | `open → dismissed \| resolved` |
| ApexProposal | `proposed → pending_approval → executed → validated`; `pending_approval → rejected` |
| MigrationPlan | `draft → testing → migrating → validated` |
| MonitoredDomain (R-3) | `pending → verified`; `verified → lapsed` (two consecutive missed daily checks); `lapsed → verified`; `pending \| verified \| lapsed → removed`; `removed → pending` (re-add: same row, new verification_token, misses and verified_at reset; ADR-091) |
| JobRun | `running → succeeded \| failed` |

## 3. API surface

The API is REST/JSON under `/api`, version prefix `/v1`. Paths below are relative to `/api`.

**Auth column legend:**

| Value | Meaning |
|---|---|
| S | Session cookie plus CSRF header on mutations |
| B | Workload bearer `czw_…` |
| A | Agent grant `cza_…` |
| O | Observer workload bearer `czw_…` (kind `observer`): only the read-only observe routes (ADR-080) |
| C | Scanner workload bearer `czw_…` (kind `scanner`): only `POST /v1/quantum/ingest` (ADR-083) |
| — | Public |

Role lists use the role names from §2. "MFA" means step-up TOTP within 15 minutes.

**Conventions:**
- Lists return `{data, nextCursor}` (`?cursor=&limit=` ≤100, default 25).
- Mutations creating billable or crypto state accept the `Idempotency-Key` header.
- Errors are §7 problem+json.
- **Mode-gated (ADR-083).** With `PRODUCT_MODE=readiness`, every `/v1/vault/*` route, `POST /v1/keys`,
  `POST /v1/keys/:id/{rotate,disable,enable,destroy}` and `POST /v1/workloads/:id/grants` answer 404 `NOT_FOUND`,
  and `POST /v1/workloads` refuses kind `agent` (400 `VALIDATION_FAILED`).
- **Plan-gated (ADR-084).** Routes marked "+ plan" call `assertEntitlement` and answer 403 `PLAN_LIMIT` when the
  tenant's plan lacks the feature or the create would exceed the limit.

| Method + path | Auth / roles | Input | Output | internal |
|---|---|---|---|---|
| POST /v1/auth/register | — | {orgName, email, password, displayName} | 201 {user, tenant}; sets session | no |
| POST /v1/auth/login | — | {email, password, totp?} | 200 {user}; sets session | no |
| POST /v1/auth/logout | S | — | 204 | no |
| GET /v1/auth/session | S | — | {user, tenant, csrfToken, mfaFresh, mode} (mode: ADR-083) | no |
| POST /v1/auth/mfa/setup | S | {currentPassword} (ADR-069, SEC-3) | {otpauthUrl, secret} | no |
| POST /v1/auth/mfa/enable | S | {code} | 204 | no |
| POST /v1/auth/mfa/verify | S | {code} | {mfaFresh:true} | no |
| POST /v1/auth/password | S | {currentPassword, newPassword} | 204 | no |
| PATCH /v1/me | S | {displayName} | {user} | no |
| GET /v1/tenant | S | — | {tenant} | no |
| PATCH /v1/tenant | S owner, admin | {name, retentionDays} | {tenant} | no |
| GET /v1/team/members | S | cursor | {data, nextCursor} | no |
| PATCH /v1/team/members/:id | S owner, admin | {role} | {member} | no |
| DELETE /v1/team/members/:id | S owner, admin | — | 204 | no |
| POST /v1/team/members/:id/password | S owner, admin | {newPassword} | 204 | no |
| GET /v1/team/invites | S owner, admin | — | {data} | no |
| POST /v1/team/invites | S owner, admin + plan (members) | {email, role} | 201 {invite, link} | no |
| DELETE /v1/team/invites/:id | S owner, admin | — | 204 | no |
| GET /v1/invites/:token | — | — | {orgName, role, email} | no |
| POST /v1/invites/:token/accept | — | {password, displayName} | 201 {user}; sets session | no |
| GET /v1/dashboard/summary | S | — | {kpis (incl. users, activeSessions, quantum counts, pqcReadiness), opsSeries, principals, keys, quantum, evidence, decisions, findings, jobs} — ADR-077 | no |
| GET /v1/keys | S | cursor, status | {data, nextCursor} | no |
| POST /v1/keys | S owner, admin, security_officer + plan (keys) | {name, purpose, profile, rotationDays} | 201 {key} | no |
| GET /v1/keys/:id | S | — | {key, versions[], dependents} | no |
| POST /v1/keys/:id/rotate | S owner, admin, security_officer | — | {key, version} | no |
| POST /v1/keys/:id/disable | S owner, admin, security_officer | — | {key} | no |
| POST /v1/keys/:id/enable | S owner, admin, security_officer | — | {key} | no |
| POST /v1/keys/:id/destroy | S owner, admin, security_officer + MFA | {reason} | 202 {approval} | no |
| GET /v1/vault/objects | S, B | cursor, keyId | {data, nextCursor} | no |
| POST /v1/vault/objects | S, B, A + plan (vaultObjects) | {keyId, label, classification, purpose, plaintextB64} | 201 {object} | no |
| GET /v1/vault/objects/:id | S, B | — | {object metadata} | no |
| POST /v1/vault/objects/:id/decrypt | S, B, A | {purpose} | {plaintextB64}; no-store | no |
| DELETE /v1/vault/objects/:id | S owner, admin, security_officer | — | 204 | no |
| GET /v1/policies | S | — | {data} | no |
| POST /v1/policies | S owner, admin, security_officer | {document} | 201 {bundle draft} | no |
| GET /v1/policies/:id | S | — | {bundle, diffToActive} | no |
| PATCH /v1/policies/:id | S author | {document} | {bundle} | no |
| POST /v1/policies/:id/submit | S owner, admin, security_officer | — | {bundle pending_approval, approval} | no |
| POST /v1/policies/simulate | S | {document?, principal, action, resource, context} | {effect, reason, ruleId, latencyUs} | no |
| GET /v1/policies/decisions | S | cursor, effect | {data, nextCursor} | no |
| GET /v1/workloads | S | cursor, kind | {data, nextCursor} | no |
| POST /v1/workloads | S owner, admin, developer + plan (workloads; kind scanner: scanners) | {name, kind ∈ service, agent, observer, scanner} | 201 {workload} | no |
| GET /v1/workloads/:id | S | — | {workload, credentials[], grants[]} | no |
| POST /v1/workloads/:id/suspend | S owner, admin, security_officer | — | {workload} | no |
| POST /v1/workloads/:id/activate | S owner, admin, security_officer | — | {workload} | no |
| POST /v1/workloads/:id/credentials | S owner, admin, developer | — | 201 {credential, token (once)} | no |
| DELETE /v1/workloads/:id/credentials/:credId | S owner, admin, developer | — | 204 | no |
| POST /v1/workloads/:id/grants | S owner, admin, security_officer + MFA | {actions, keyIds, purpose, ttlSeconds≤900} | 201 {grant, token (once)} | no |
| GET /v1/grants | S | cursor, status | {data, nextCursor} | no |
| POST /v1/grants/:id/revoke | S owner, admin, security_officer | — | {grant} | no |
| GET /v1/approvals | S | status | {data} | no |
| POST /v1/approvals/:id/approve | S (role per kind) + MFA, not requester | — | {approval} | no |
| POST /v1/approvals/:id/reject | S (role per kind), not requester | {reason} | {approval} | no |
| GET /v1/evidence/events | S | cursor, action | {data, nextCursor} | no |
| GET /v1/evidence/verify | S | — | {ok, events, checkpoints, brokenAt?} | no |
| GET /v1/evidence/checkpoints | S | — | {data} | no |
| GET /v1/evidence/export | S owner, admin, auditor + plan (evidenceExport) | — | application/x-ndjson | no |
| GET /v1/evidence/operations | S | cursor, op | {data, nextCursor} | no |
| GET /v1/quantum/assets | S | cursor, vulnerable, source (managed, scanned), minRisk | {data (incl. riskScore, locator), nextCursor, exposure} | no |
| POST /v1/quantum/scan | S owner, admin, security_officer | — | 202 {jobRun} | no |
| POST /v1/quantum/ingest | C (internal: customer-side cz-scan) + Idempotency-Key + plan (assets) | {scanId, scanner {name, version}, startedAt, finishedAt, scope?[≤5000], findings[≤5000] {source ∈ tls, certificate, dependency, locator, algorithm, keySize?, protocol?, notAfter?}} — strict, metadata only (ADR-083) | 202 {scanReport, accepted, rejected, created, updated, replay} | yes |
| GET /v1/quantum/scans | S | cursor, origin (cz-scan, external) | {data (scan reports, incl. origin and domain), nextCursor} | no |
| GET /v1/quantum/cbom | S owner, admin, security_officer, auditor + plan (cbomExport) | — | application/vnd.cyclonedx+json (CycloneDX 1.6) | no |
| GET /v1/quantum/plans | S | — | {data} | no |
| POST /v1/quantum/plans | S owner, admin, security_officer + plan (migrationPlans) | {name, targetProfile, assetIds} (managed or scanned assets, not both) | 201 {plan} | no |
| POST /v1/quantum/plans/:id/advance | S owner, admin, security_officer | — | {plan}; a scanned-asset plan reaches `validated` only when a scan after `migrating` began shows every asset on a PQC/hybrid algorithm or no longer present | no |
| GET /v1/quantum/domains | S | — | {data [{id, name, status, ports, txtRecord {name, value}, verifiedAt, lastScanAt, hostCount}]} (R-3) | no |
| POST /v1/quantum/domains | S owner, admin, security_officer + plan (domains) | {name, ports?[≤8] ⊆ 443, 8443, 465, 993, 995, 636, 853, 5061} | 201 {domain, txtRecord {name, value}} (R-3, ADR-089) | no |
| PATCH /v1/quantum/domains/:id | S owner, admin, security_officer | {ports} | {domain} | no |
| POST /v1/quantum/domains/:id/verify | S owner, admin, security_officer | — | 202 {jobRun} (interactive `domain-verify`) | no |
| POST /v1/quantum/domains/:id/scan | S owner, admin, security_officer | — | 202 {jobRun} (interactive `external-discovery`); 409 INVALID_TRANSITION unless `verified`; 429 within 1 h of the last on-demand scan | no |
| DELETE /v1/quantum/domains/:id | S owner, admin, security_officer | — | 204; status `removed`, probes stop, assets kept | no |
| GET /v1/quantum/domains/:id/hosts | S | cursor, status | {data [{hostname, origin, status, lastProbeAt}], nextCursor} | no |
| POST /v1/quantum/domains/:id/hosts | S owner, admin, security_officer | {hostname} (the domain or under it on a label boundary; no wildcard) | 201 {host} | no |
| GET /v1/apex/findings | S | status | {data} | no |
| GET /v1/apex/findings/:id | S | — | {finding, evidence[], proposals[]} | no |
| POST /v1/apex/scan | S owner, admin, security_officer | — | 202 {jobRun} | no |
| POST /v1/apex/findings/:id/dismiss | S owner, admin, security_officer | {reason} | {finding} | no |
| POST /v1/apex/findings/:id/propose | S owner, admin, security_officer | — | 201 {proposal} | no |
| POST /v1/apex/proposals/:id/submit | S owner, admin, security_officer + plan (apexRemediation) | — | {proposal, approval} | no |
| GET /v1/ops/jobs | S owner, admin | cursor, job | {data, nextCursor, schedulers, dlq} | no |
| POST /v1/ops/jobs/:job/run | S owner, admin | — | 202 {jobRun} | no |
| POST /v1/ops/dlq/:id/retry | S owner, admin | — | 202 | no |
| GET /v1/billing/plans | S | — | {data} | no |
| GET /v1/billing/subscription | S | — | {subscription, payments[]} | no |
| GET /v1/billing/entitlements | S | — | {plan (free, sovereign-team, sovereign-enterprise), mode, limits {name: max \| null}, features {name: bool}, usage {name: count}, tiers [{plan, name, limits, features}]} (ADR-084) | no |
| POST /v1/billing/checkout | S owner + Idempotency-Key | {planCode} | 201 {checkoutId, url} | no |
| GET /v1/billing/checkout/:id | S owner | — | {checkout, plan, amount} | no |
| POST /v1/billing/checkout/:id/simulate | S owner (BILLING_MODE=simulated) | {outcome: succeeded, failed} | 202 | no |
| POST /v1/billing/subscription/cancel | S owner | — | {subscription} | no |
| POST /v1/billing/payments/:id/refund | S owner + Idempotency-Key | — | 202 | no |
| POST /v1/billing/webhook | Stripe-Signature (internal: processor callback) | raw body | 200 | yes |
| GET /health | — (internal: probes) | — | {status, version, commit, uptimeSec, checks} | yes |
| GET /metrics | — (internal: Prometheus scrape, not routed by edge) | — | text/plain | yes |
| GET /v1/observe/snapshot | O (internal: STΛRGΛZΞR observer) | — | {contract: cipherzero.observation/v1, observedAt, source, tenantId, assets[], services[], findings[], evidence, truncated} — ADR-080 | yes |
| GET /v1/observe/events | O (internal: STΛRGΛZΞR observer) | after (seq), limit ≤100 | {contract: cipherzero.observation.event/v1, data[{id, seq, observedAt, type, target, actor, basis, provenance}], nextAfter} — ADR-080 | yes |

**gRPC.** `TrustCore` (`proto/trust.proto`) is reachable only from api and worker over mTLS:
- Encrypt, Decrypt, CreateKey, RotateKey, Rewrap, DestroyKeyVersion
- Sign, Verify
- EvaluatePolicy, ActivateBundle
- Health

## 4. Security requirements

- **Authentication.**
  - Passwords: argon2id (m=19 MiB, t=2, p=1). Minimum 15 characters, up to 128 accepted. No composition rules.
  - Breached-password check: HIBP k-anonymity when `HIBP_MODE=online`; otherwise a bundled top-100k list.
  - Login limiter before hash verification (INV-14). Lockout after 20 consecutive failures for 15 minutes.
  - Sessions: 12-hour absolute and 30-minute idle lifetime. Rotated on login and MFA. Stored hashed.
- **Cookies.**
  - `__Host-cz_session` with Secure, HttpOnly, SameSite=Strict, Path=/.
  - `__Host-cz_csrf`, readable by JS, echoed in `x-csrf-token` on every mutation. The server compares it with the stored hash.
- **Authorization.**
  - Deny by default at three layers: the route role guard, the tenant-scoped repository, and the trust-core PDP for any key operation.
  - Cross-tenant ids → 404.
  - Separation of duties: approver ≠ requester, plus MFA.
- **Cryptography.** See ARCHITECTURE §5.1.
  - AEAD with per-object DEKs and random 96-bit nonces.
  - Invocation counters with `KEY_MAX_INVOCATIONS`.
  - Hybrid PQC profiles.
  - `TRUST_ROOT_KEY` exists only in trust-core.
- **Transport.**
  - HTTPS only via Caddy (`:80` → 308).
  - mTLS for gRPC with a SPIFFE allow-list.
  - HSTS only with `HSTS_ENABLED=true`.
- **Headers.**
  - CSP Profile N (nonce plus strict-dynamic) set in `proxy.ts`.
  - nosniff; Referrer-Policy strict-origin-when-cross-origin; Permissions-Policy (camera, microphone and geolocation disabled); X-Frame-Options DENY; COOP same-origin; no X-Powered-By.
- **Rate limits.**
  - Per IP: 300 per minute. Per tenant: 3,000 per minute. Login: 10 per 15 minutes per IP plus email hash. Crypto: 600 per minute per principal.
  - Exceeding a limit returns 429 with `Retry-After`.
- **Input.**
  - zod on every body, query and params.
  - Body limit 1 MB. Vault plaintext ≤ 256 KiB.
- **Logging.** pino redaction covers authorization, cookie, set-cookie, password, newPassword, currentPassword, totp, code, plaintextB64, token, secret, `*.wrappedDek` and `TRUST_ROOT_KEY`.
- **Payments.**
  - PAN and CVV are never accepted.
  - Webhook signature is verified before any DB read (INV-15).
  - Idempotency rows are kept for at least 72 hours (default 720).
- **AI isolation.** APEX code never imports crypto (INV-4). Proposals execute only through approvals.
- **Scanner ingest (ADR-083).**
  - A scanner bearer reaches only `POST /v1/quantum/ingest`. It cannot read any route, and no other bearer kind
    can call ingest.
  - The body is a strict zod schema (unknown fields → 400). Every string is checked for secret material before
    any DB access: PEM `PRIVATE KEY` blocks (including PGP `PRIVATE KEY BLOCK` and PuTTY key files), OpenSSH
    private keys, JWK private members (`"d"`, `"p"`, `"q"`), base64 runs longer than 512 characters, and base64
    runs of 96 or more characters with at least 5.0 bits per character of Shannon entropy. Percent-encoded strings
    are checked in decoded form too. A hit refuses the whole report with 422 `SECRET_MATERIAL_REJECTED` and logs
    only the finding index (INV-50). Before that, strings with C0/C1 control characters (other than tab, CR, LF)
    or bidi overrides are refused with 400, and a non-scanner caller gets 403 before any validation
    (ADR-085).
  - `locator` is at most 512 characters (host:port, a file path or a package name). Payloads are capped at 5,000
    findings and 1 MB.
  - Reports are idempotent by `Idempotency-Key` and `(tenant_id, scan_id)`. Every accepted report writes one audit
    event (`quantum.scan.ingested`) with its digest and counts.
- **Scanner CLI (ADR-083).** It never opens files matching private-key patterns (`*.key`, `id_*`, `*.p12`, `*.pfx`,
  `*.jks`), uploads only the fields of the ingest schema, offers `--dry-run` to print the exact payload, and sends
  the token only over HTTPS (`--insecure` is refused unless `--api` is loopback). The scanner never imports
  `@cz/core` (INV-52).
- **External discovery (R-3, ADR-089).**
  - Only hosts under a domain in status `verified` are probed (INV-56). Verification is a DNS TXT record
    `_cipherzero-challenge.<domain>` = `cz-domain-verification=<token>`, re-checked daily. Two consecutive misses
    lapse the domain.
  - Domain names are refused (400) when they are IP literals, public suffixes, longer than 253 characters, or under
    `localhost`, `local`, `internal`, `arpa`, `test`, `example` or `invalid`.
  - All outbound DNS, CT and TLS traffic goes through `apps/prober`. The prober has no route to Postgres, Redis,
    trust-core, the API or the edge, holds no database or Redis credentials, and never imports `@cz/core` (INV-55).
    Only the scanner and the prober open raw sockets or DNS (INV-53).
  - The prober resolves each host once and refuses it if any A/AAAA address is non-public (IANA special-purpose
    registries, including IPv4-mapped, NAT64 and 6to4 embedded addresses and CGNAT). It then connects to that exact
    address with SNI, completes a TLS handshake and closes, sending no application data (INV-54). Ports come from the
    allowlist only. Refused hosts are stored as `private` and never probed.
  - Worker → prober is HTTPS with mTLS. The prober accepts only the worker's SPIFFE id.
  - External findings pass the same secret-material guard as `cz-scan` reports, inside `ingestFindings()` before
    any DB access (INV-57).
  - Rate: one on-demand scan per domain per hour, 8 concurrent probes per prober, 2 per domain, a 10 s timeout and
    1,000 hosts per domain. One run spends at most 20 minutes probing (the rest is probed next run), and one worker
    runs at most 2 external-discovery jobs at once (ADR-095). Every domain add, verify, lapse, remove and external scan is audited.
- **Entitlements (ADR-084).** Limits are checked inside the creating transaction, so concurrent creates cannot
  overshoot by more than the in-flight request. Downgrades never delete data: over-limit tenants keep read access
  and cannot create more until they are under the limit again.
- **Compliance.** HIPAA, PCI-DSS, SOC 2 and ISO 27001 are **mapped** in COMPLIANCE.md. No certification is claimed.

## 5. Features & Workflows

### 5.1 Features

| ID | Feature | Workflows / steps served |
|---|---|---|
| F-1 | Service floor & data layer: monorepo, `@cz/core` env/logger/problem/db, Prisma schema + migrations, health, Docker, compose, Caddy | all |
| F-2 | trust-core: gRPC + mTLS, crypto provider (AEAD, hybrid KEM/sig), PDP, key versions | W1.3–W1.5, W2.3–W2.5, W3.2, W3.4 |
| F-3 | Design system, app shell, landing, PWA/offline | all UI |
| F-4 | Authentication: register, login, logout, session, CSRF, password policy, HIBP, limiter, TOTP, step-up | W1.1, W2.1 |
| F-5 | Team & tenant settings: invites, roles, member admin, profile | — |
| F-6 | TRUST policy engine: bundles, signatures, two-person activation, simulation, decisions | W1.2, W2.3, W6.5 |
| F-7 | Workloads & AI-agent grants | W4.1–W4.6 |
| F-8 | CORE keys: create, rotate, disable, destroy via approval | W3.1–W3.2 |
| F-9 | VAULT protect & authorized decrypt | W1.1–W1.6, W2.1–W2.6 |
| F-10 | EVIDENCE: hash chain, checkpoints, verify, export, operations log | W1.5, W2.6, W4.5, W6.7 |
| F-11 | Approvals inbox (separation of duties) | W3.5, W6.5–W6.6 |
| F-12 | Scheduled work: rotation, rewrap, grant expiry, checkpoints, retention, approval expiry; ops console + DLQ | W3.1–W3.5, W4.6 |
| F-13 | QUANTUM: inventory, exposure, migration plans, interop test | W5.1–W5.6 |
| F-14 | APEX: detectors, findings, proposals, validation | W6.1–W6.7 |
| F-15 | Billing: plans, checkout, signed webhooks, subscription, refunds, cancel | — |
| F-16 | COMMAND CENTER dashboard & analytics | all (read) |
| F-17 | Delivery & observability: Prometheus, Grafana, CI, web baseline, load test, smoke | — |
| F-21 | Product modes: `PRODUCT_MODE`, route gating, console nav gating (ADR-083) | all |
| F-22 | Entitlements: plan map, `assertEntitlement`, `PLAN_LIMIT`, `GET /v1/billing/entitlements`, usage and upgrade UI (ADR-084) | — |
| F-23 | Scanner ingest and inventory: `scanner` workload kind, secret-material guard, `scan_reports`, risk score, CBOM export, scanned-asset migration plans (ADR-083) | W7.1, W7.3–W7.6 |
| F-24 | Scanner CLI `apps/scanner` (`cz-scan`): tls, certificate and dependency discovery, dry-run, upload (ADR-083) | W7.2–W7.3 |
| F-25 | Prober `apps/prober`: isolated egress container, egress guard, mTLS from the worker, `resolve-txt` / `ct-search` / `probe-tls` (ADR-089) | W8.2–W8.4 |
| F-26 | Monitored domains: `monitored_domains`, add/verify/remove API, `domain-verify` job, `domains` plan limit (ADR-089) | W8.1–W8.2, W8.5 |
| F-27 | External discovery: `external_hosts`, `external-discovery` job (apex, www, CT, manual), `ingestFindings()` in core, `scan_reports.origin` (ADR-089) | W8.3–W8.6 |
| F-28 | Console: Domains tab on Quantum (add, TXT record, verify, hosts, scan now), origin column on scan reports, private-host hint (ADR-089) | W8.1–W8.6 |

### 5.2 Workflows

**W1 — Protect data** (RESEARCH workflow A)
- W1.1 An app (session user or workload bearer) authenticates to `POST /v1/vault/objects`.
- W1.2 trust-core evaluates `key.encrypt` against the active signed bundle.
- W1.3 The key authority is selected: the key's active version (profile AES256_GCM or HYBRID_X25519_MLKEM768).
- W1.4 Data is encrypted: fresh DEK, AES-256-GCM, AAD bound, DEK wrapped.
- W1.5 Metadata, crypto operation and audit event are recorded atomically.
- W1.6 **Terminal:** ciphertext reference returned (201, object id, key version, digest), shown in VAULT.

**W2 — Authorized decryption** (RESEARCH workflow B)
- W2.1 Workload, user or agent identity verified.
- W2.2 Tenant boundary checked (foreign object → 404).
- W2.3 Purpose and policy evaluated (`key.decrypt` with purpose; grant scope for agents).
- W2.4 Key operation authorized and DEK unwrapped in trust-core.
- W2.5 Plaintext released only to the requesting principal (`no-store`).
- W2.6 **Terminal:** decryption event audited and visible in EVIDENCE.

**W3 — Key rotation** (RESEARCH workflow C)
- W3.1 Key aging (`next_rotation_at` due) or risk (APEX or a manual rotate) detected.
- W3.2 New key version generated in trust-core.
- W3.3 New writes use the new version.
- W3.4 Existing ciphertext rewrapped by the `rewrap` job.
- W3.5 **Terminal:** old version retired after dependency verification (zero objects on it), audited.

**W4 — AI-agent access** (RESEARCH workflow D)
- W4.1 Agent workload registered and authenticated.
- W4.2 Capability evaluated (requested actions and keys versus policy).
- W4.3 Short-lived scoped grant issued (TTL ≤ 900 s, step-up MFA by the issuer).
- W4.4 Operation executed with the `cza_` token, within scope only.
- W4.5 Output and access evidence recorded.
- W4.6 **Terminal:** privilege expires (TTL plus `grant-expiry` job) or is revoked. Later use → 401.

**W5 — Quantum migration** (RESEARCH workflow E)
- W5.1 Dependencies discovered (`quantum-inventory` scan of keys, profiles and usage).
- W5.2 Algorithms inventoried (crypto assets with algorithm and vulnerability flag).
- W5.3 Exposure assessed (vulnerable share, weighted by usage).
- W5.4 Approved hybrid or PQC profile tested (migration plan `testing`, round-trip interop test).
- W5.5 Phased migration governed (`migrating`: rotate assets to the hybrid profile, rewrap).
- W5.6 **Terminal:** interoperability validated (plan `validated` with the interop result).

**W6 — APEX analysis** (RESEARCH workflow F)
- W6.1 Telemetry collected (operations, decisions, auth events, grants).
- W6.2 Anomaly identified by deterministic detectors.
- W6.3 Risk scored (0–100, severity).
- W6.4 Remediation proposed (an `apex_proposal`, with no side effects).
- W6.5 Policy gate evaluates (`apex.remediate`) and an approval request is raised.
- W6.6 Approved action executed by the api service (not by APEX).
- W6.7 **Terminal:** outcome validated (the detector re-runs) and logged to EVIDENCE.

**W7 — External quantum readiness** (RESEARCH re-entry R-2)
- W7.1 An owner, admin or developer registers a workload of kind `scanner` and issues its credential (token shown once).
- W7.2 The customer runs `cz-scan` inside their network against TLS targets, certificate directories and
  dependency manifests. `--dry-run` shows the exact payload.
- W7.3 The scanner posts the metadata to `POST /v1/quantum/ingest`. Secret material is refused, and an accepted
  report is audited.
- W7.4 Findings are upserted as scanned crypto assets with a 0–100 risk score, and exposure is recomputed.
- W7.5 A migration plan is created over scanned assets and advanced to `testing`, then `migrating`.
- W7.6 **Terminal:** a later scan shows every plan asset on a PQC or hybrid algorithm, or gone. The plan reaches
  `validated` with the scan report id as evidence, and the CBOM exports from QUANTUM.

**W8 — Clientless external discovery** (RESEARCH re-entry R-3)
- W8.1 An owner, admin or security officer adds a domain in QUANTUM → Domains. The console shows the TXT record to
  publish.
- W8.2 The customer publishes the record and selects "Verify". The `domain-verify` job asks the prober to resolve
  it, and the domain becomes `verified` (audited).
- W8.3 `external-discovery` builds the host list from the apex, `www`, CT names and manual hosts. Wildcards are
  stored as `wildcard`, and hosts that resolve to non-public addresses as `private`. Neither is probed.
- W8.4 The prober handshakes with each public host:port. Findings are ingested as scanned assets (origin
  `external`) with risk scores. Unreachable hosts stay out of scope.
- W8.5 Daily: the record is re-checked (a domain lapses after two misses, and probes stop) and verified domains are
  re-scanned.
- W8.6 **Terminal:** the external assets appear in QUANTUM, merged with any `cz-scan` asset for the same locator.
  They count toward exposure, export in the CBOM, and a migration plan over them reaches `validated` when a later
  external scan shows them on a PQC or hybrid algorithm (the W7.6 rule).

## 6. UI Surface

| Screen | Route | Example | Roles | Binds | Workflow steps | User controls | States |
|---|---|---|---|---|---|---|---|
| Landing | / | / | public | — | — | Nav links, "Request access" → /register, "Sign in" → /login, theme toggle | default |
| Sign in | /login | /login | public | POST /v1/auth/login | W1.1, W2.1 | email, password, TOTP (conditional), submit, link to register | idle, submitting, error (401/429), mfa-required |
| Register | /register | /register | public | POST /v1/auth/register | — | org name, display name, email, password (live length meter), submit | idle, submitting, field errors, breached-password error |
| Accept invite | /invite/[token] | /invite/demo-token | public | GET /v1/invites/:token, POST /v1/invites/:token/accept | — | display name, password, accept | loading, invalid/expired, idle, submitting, error |
| Offline | /offline | /offline | public | — | — | retry | offline |
| Overview (Command Center) | /dashboard | /dashboard | all | GET /v1/dashboard/summary | W1.6, W6.2 | range (24h/7d/30d), drill-in links, key lifecycle, APEX proposals, quantum inventory, evidence stream (ADR-076, ADR-077) | loading, empty (new tenant onboarding), populated, error |
| Keys | /keys | /keys | all (write: owner, admin, security_officer) | GET /v1/keys, POST /v1/keys | W3.1 | filter by status, "New key" dialog (name, purpose, profile, rotation days), row → detail | loading, empty, populated, error, forbidden-action |
| Key detail | /keys/[id] | /keys/{seed.keyId} | all | GET /v1/keys/:id, POST /v1/keys/:id/rotate, POST /v1/keys/:id/disable, POST /v1/keys/:id/enable, POST /v1/keys/:id/destroy | W3.2, W3.3, W3.5 | rotate, disable/enable, destroy (reason + confirm → approval), versions table | loading, not-found, populated, pending-approval, error |
| Vault | /vault | /vault | all (write: owner, admin, security_officer, developer) | GET /v1/vault/objects, POST /v1/vault/objects | W1.1–W1.6 | "Protect data" form (key, label, classification, purpose, text or file), filter by key | loading, empty, populated, submitting, policy-denied, error |
| Vault object | /vault/[id] | /vault/{seed.objectId} | all | GET /v1/vault/objects/:id, POST /v1/vault/objects/:id/decrypt, DELETE /v1/vault/objects/:id | W2.1–W2.6 | purpose input, "Decrypt" (reveal with copy and auto-hide after 30 s), delete with confirm | loading, not-found, populated, decrypted, policy-denied, error |
| Policies | /policies | /policies | all | GET /v1/policies, POST /v1/policies, POST /v1/policies/simulate, GET /v1/policies/decisions | W1.2, W2.3 | "New draft" (clone active), simulator (principal, action, resource, purpose), decisions table with effect filter | loading, populated, simulating, error |
| Policy bundle | /policies/[id] | /policies/{seed.policyId} | all | GET /v1/policies/:id, PATCH /v1/policies/:id, POST /v1/policies/:id/submit | W6.5 | JSON rule editor (validated), save, submit for approval, diff vs. active | loading, draft-editing, invalid-json, pending-approval, active, error |
| Workloads | /workloads | /workloads | all | GET /v1/workloads, POST /v1/workloads, GET /v1/grants | W4.1, W7.1 | kind filter, "Register workload" (name, kind service/agent/observer/scanner; agent hidden in readiness mode), active grants panel (platform mode) | loading, empty, populated, plan-limit, error |
| Workload detail | /workloads/[id] | /workloads/{seed.workloadId} | all | GET /v1/workloads/:id, POST /v1/workloads/:id/credentials, DELETE /v1/workloads/:id/credentials/:credId, POST /v1/workloads/:id/grants, POST /v1/grants/:id/revoke, POST /v1/workloads/:id/suspend, POST /v1/workloads/:id/activate | W4.2–W4.6 | issue credential (token shown once), revoke credential, issue grant (actions, keys, purpose, TTL; MFA prompt), revoke grant, suspend/activate | loading, not-found, populated, mfa-required, token-reveal, error |
| Approvals | /approvals | /approvals | all (decide: per kind) | GET /v1/approvals, POST /v1/approvals/:id/approve, POST /v1/approvals/:id/reject | W3.5, W6.5, W6.6 | status tabs, approve (MFA prompt), reject (reason) | loading, empty, populated, mfa-required, self-approval-blocked, error |
| Evidence | /evidence | /evidence | all | GET /v1/evidence/events, GET /v1/evidence/verify, GET /v1/evidence/checkpoints, GET /v1/evidence/export, GET /v1/evidence/operations | W1.5, W2.6, W4.5, W6.7 | tabs (events / operations / checkpoints), action filter, "Verify chain", "Export NDJSON", load more | loading, empty, populated, verifying, verified, broken, error |
| Quantum | /quantum | /quantum | all | GET /v1/quantum/assets, POST /v1/quantum/scan, GET /v1/quantum/scans, GET /v1/quantum/cbom, GET /v1/quantum/plans, POST /v1/quantum/plans, POST /v1/quantum/plans/:id/advance | W5.1–W5.6, W7.2, W7.4–W7.6, W8.1–W8.6 | "Run discovery" (managed keys), Domains tab (R-3: "Add domain", TXT record with copy, "Verify", status, hosts with status filter, "Add host", "Scan now", private-host hint linking to the cz-scan setup), source filter (managed/scanned), vulnerable filter, risk column with sort, scan reports tab with the `cz-scan` command and an origin column (R-3), "Export CBOM", select assets → "Plan migration" (name, target profile), "Advance phase" | loading, empty (no scans: setup steps for cz-scan and for a domain), scanning, populated, domain pending, domain lapsed, plan-limit, error |
| APEX | /apex | /apex | all | GET /v1/apex/findings, POST /v1/apex/scan | W6.1–W6.3 | "Run analysis", status filter | loading, empty, scanning, populated, error |
| APEX finding | /apex/[id] | /apex/{seed.findingId} | all | GET /v1/apex/findings/:id, POST /v1/apex/findings/:id/dismiss, POST /v1/apex/findings/:id/propose, POST /v1/apex/proposals/:id/submit | W6.4–W6.7 | dismiss (reason), "Propose remediation", "Submit for approval" | loading, not-found, populated, proposed, pending-approval, executed, validated, error |
| Operations | /operations | /operations | owner, admin | GET /v1/ops/jobs, POST /v1/ops/jobs/:job/run, POST /v1/ops/dlq/:id/retry | W3.1, W3.4, W4.6 | job filter, "Run now" per scheduler, DLQ retry | loading, empty, populated, forbidden, error |
| Settings | /settings | /settings | all | GET /v1/auth/session, PATCH /v1/me, POST /v1/auth/password, POST /v1/auth/mfa/setup, POST /v1/auth/mfa/enable, POST /v1/auth/mfa/verify, POST /v1/auth/logout, GET /v1/tenant, PATCH /v1/tenant | — | display name, change password, MFA setup (QR, secret, code), theme (system/light/dark), org name and retention (admin), sign out | loading, populated, saving, saved, error |
| Team | /settings/team | /settings/team | all (manage: owner, admin) | GET /v1/team/members, PATCH /v1/team/members/:id, DELETE /v1/team/members/:id, POST /v1/team/members/:id/password, GET /v1/team/invites, POST /v1/team/invites, DELETE /v1/team/invites/:id | — | invite (email, role) → copy link, change role, remove member, reset password, revoke invite | loading, populated, error, forbidden-action |
| Billing | /settings/billing | /settings/billing | all (manage: owner) | GET /v1/billing/plans, GET /v1/billing/subscription, GET /v1/billing/entitlements, POST /v1/billing/checkout, POST /v1/billing/subscription/cancel, POST /v1/billing/payments/:id/refund | — | current plan with usage meters (used / limit per resource), plan comparison (Free, Team, Enterprise), choose plan → checkout, cancel subscription (confirm), refund payment (confirm), payment history | loading, free, active, past-due, canceled, over-limit, error |

**Plan-limit state (ADR-084).** On any screen, a 403 `PLAN_LIMIT` shows a Banner naming the limit ("Free plan:
50 of 50 crypto assets") with an "Upgrade" link to `/settings/billing`. Nav entries for mode-disabled modules are
not rendered (ADR-083).
| Checkout | /settings/billing/checkout/[id] | /settings/billing/checkout/{seed.checkoutId} | owner | GET /v1/billing/checkout/:id, POST /v1/billing/checkout/:id/simulate | — | (simulated mode) "Pay" / "Decline"; (Stripe) redirect | loading, awaiting-webhook, succeeded, failed, error |

### 6.1 Design System

- **Tokens.** ADR-000 `:root` as amended by template v2 (ADR-076: success green #3DDC97 separate from the cyan accent, amber #F5A524, violet #8B7CFF, chart blue #3D6BFF) and the brand lockup (ADR-078), dark by default, light via `[data-theme="light"]`, plus `prefers-color-scheme` when the theme is `system`.
  - Theme strategy: attribute `data-theme` on `<html>`, stored under the localStorage key `theme`.
  - The theme is applied before first paint by a nonce'd inline script.
- **Fonts.** Michroma (display), Sora 300–600 (body) and IBM Plex Mono 400/500 (data), self-hosted with `font-display: swap`.
- **Primitives** (`apps/web/src/components/ui`): Button (primary / secondary-dashed / ghost / danger), Input, Select, Textarea, Field (label, hint, error), Table, Badge (status tone), Card, Dialog (focus-trapped, Esc), Tabs, Toast (aria-live), EmptyState, Skeleton, StatTile, CodeBlock, Banner.
- **Shell.** saas sidebar (`nav[aria-label="Primary"]`) with dot indicators, listing Overview, Keys, Vault, Access policy, Workloads & agents, Approvals, Audit & evidence, Quantum, APEX insights, Operations and Settings (ADR-076), and a live platform-health box from `/api/health`.
  - Mobile: the sidebar becomes a drawer below 768px.
  - The top bar shows the tenant name, a jump-to search over the nav destinations, the MFA freshness indicator, a theme toggle and the initials avatar menu.
- **Accessibility and layout.**
  - Status is never conveyed by colour alone (icon plus text).
  - Visible focus ring: 2px accent.
  - Hit targets ≥ 44px.
  - Zero horizontal scroll at 375px.
  - Tables become stacked cards below 768px.
- **Data honesty.** Dashboards show only persisted tenant data. Empty states explain how to produce data. There is no invented telemetry and no certification badges (art direction).

## 7. Error contract

Every error is `application/problem+json` (RFC 9457), with these fields:
- `type`: `https://cipherzero.dev/problems/<code>`
- `title`
- `status`
- `detail`
- `instance`: the request path
- `code`: a stable UPPER_SNAKE value
- `requestId`
- `errors[]`: present for validation, as `{path, message}`

| Status | code examples |
|---|---|
| 400 | VALIDATION_FAILED, INVALID_JSON, IDEMPOTENCY_KEY_REQUIRED |
| 401 | UNAUTHENTICATED, GRANT_EXPIRED, INVALID_CREDENTIAL |
| 403 | FORBIDDEN, POLICY_DENIED, CSRF_FAILED, MFA_REQUIRED, SELF_APPROVAL_FORBIDDEN, PLAN_LIMIT (adds `limit`, `plan`, `used`, `max`; ADR-084) |
| 404 | NOT_FOUND (also every cross-tenant id, and mode-disabled routes; ADR-083) |
| 409 | CONFLICT, INVALID_TRANSITION, IDEMPOTENCY_KEY_REUSED |
| 413 | PAYLOAD_TOO_LARGE |
| 422 | BREACHED_PASSWORD, PASSWORD_TOO_SHORT, INTEGRITY_FAILURE, SECRET_MATERIAL_REJECTED (adds `findingIndex`; ADR-083) |
| 429 | RATE_LIMITED (with Retry-After) |
| 503 | TRUST_CORE_UNAVAILABLE, POLICY_UNAVAILABLE (fail closed), PROBER_UNAVAILABLE (R-3: on-demand verify or scan while the prober is down or external discovery is disabled) |
| 500 | INTERNAL_ERROR (no stack, requestId only) |

## 8. Environment & Configuration

`packages/core/src/env.ts` defines one zod schema per process, except the prober: INV-55 bars it from `@cz/core`, so its schema is `apps/prober/src/env.ts`, with the same rules (ADR-090). On failure the process exits with a message naming the missing or invalid variable. An empty string is coerced to undefined for optional variables. Production secrets have no defaults. `.env.example` lists every variable with a placeholder, never a real value. Ports come from env vars, and `scripts/ports.mjs` probes for free ones.

| Variable | Used by | Required | Default | Secret | Notes |
|---|---|---|---|---|---|
| NODE_ENV | all | no | development | no | production in containers |
| APP_VERSION | all | no | 0.0.0 | no | baked at build |
| GIT_COMMIT | all | no | unknown | no | baked at build |
| LOG_LEVEL | all | no | info | no | |
| DATABASE_URL | api, worker, trust-core, migrate | yes | — | yes | primary |
| DATABASE_REPLICA_URL | api | no | (primary) | yes | read replica |
| REDIS_URL | api, worker, trust-core | yes | — | yes | |
| SESSION_SECRET | api | yes | — | yes | ≥32 bytes |
| PII_ENCRYPTION_KEY | api | yes | — | yes | base64 32 bytes |
| PII_BLIND_INDEX_KEY | api | yes | — | yes | base64 32 bytes |
| TRUST_ROOT_KEY | trust-core | yes | — | yes | base64 32 bytes; trust-core only (INV-3) |
| TRUST_CORE_ADDR | api, worker | no | trust-core:50051 | no | |
| TRUST_CORE_PORT | trust-core | no | 50051 | no | |
| MTLS_CA_PATH / MTLS_CERT_PATH / MTLS_KEY_PATH | api, worker, trust-core, prober | yes | — | key yes | from the service's own pki-<service> volume (ADR-094) |
| TRUST_ALLOWED_SPIFFE_IDS | trust-core | no | spiffe://cipherzero.local/ns/default/sa/api,…/worker | no | |
| API_PORT | api | no | 4000 | no | |
| WEB_PORT | web | no | 3000 | no | |
| METRICS_PORT | api, worker, trust-core | no | 9100 / 9465 / 9464 | no | |
| API_INTERNAL_URL | web (server) | no | http://api:4000 | no | |
| PUBLIC_BASE_URL | api, web | yes | — | no | e.g. https://localhost:8443 |
| SITE_ADDRESS | edge | no | localhost | no | domain enables ACME |
| EDGE_HTTP_PORT / EDGE_HTTPS_PORT | edge | no | 80 / 443 | no | probed |
| HSTS_ENABLED | web, api | no | false | no | true only with a trusted certificate |
| BILLING_MODE | api, worker | no | simulated | no | simulated, stripe |
| STRIPE_SECRET_KEY | api | when stripe | — | yes | |
| BILLING_WEBHOOK_SECRET | api, worker | yes | — | yes | whsec_… |
| HIBP_MODE | api | no | offline | no | offline, online |
| RATE_LIMIT_SCALE | api | no | 1 | no | load test only |
| KEY_MAX_INVOCATIONS | trust-core | no | 1073741824 | no | |
| IDEMPOTENCY_RETENTION_HOURS | worker | no | 720 | no | min 72 |
| SEED_DEMO | migrate | no | false | no | seeds the demo tenant and baseline user |
| WEB_BASELINE_USER / WEB_BASELINE_PASSWORD | web-baseline, e2e | no | — | password yes | |
| POSTGRES_PASSWORD / REPLICATION_PASSWORD | postgres | yes | — | yes | |
| GRAFANA_ADMIN_PASSWORD | grafana | yes | — | yes | |
| PRODUCT_MODE | api, worker | no | platform | no | platform, readiness (ADR-083). The console reads it from `GET /v1/auth/session` |
| EXTERNAL_DISCOVERY_ENABLED | api, worker | no | true | no | R-3. false skips the domain sweeps, and on-demand calls answer 503 |
| PROBER_URL | worker | no | https://prober:4100 | no | R-3, mTLS (ADR-089) |
| PROBER_PORT | prober | no | 4100 | no | |
| PROBER_ALLOWED_SPIFFE_IDS | prober | no | spiffe://cipherzero.local/ns/default/sa/worker | no | |
| PROBER_DNS_SERVERS | prober | no | (system) | no | comma-separated resolvers |
| PROBER_CONCURRENCY | prober | no | 8 | no | the per-domain cap is 2 |
| PROBER_TIMEOUT_MS | prober | no | 10000 | no | per handshake or lookup; 1000–15000 (ADR-095) |
| CT_SEARCH_URL | prober | no | https://crt.sh | no | empty disables CT. The CT provider sees the tenant's domain names |
| PROBER_ALLOW_CIDRS | prober | no | — | no | **test only**: lets e2e probe loopback. The prober refuses to boot with it when NODE_ENV=production |

**Scanner CLI** (customer side; not part of the server env schema): `CZ_API_URL` (https base URL), `CZ_SCANNER_TOKEN`
(secret, the scanner workload's `czw_` credential). Flags: `--tls host:port…`, `--certs <dir>…`, `--deps <dir>…`,
`--dry-run`, `--out <file>`, `--scan-id <id>`.

## 9. Performance

| Metric | Target | Measured by |
|---|---|---|
| Policy evaluation p95 | < 25 ms | `cz_policy_eval_seconds`; trust-core unit benchmark |
| Key operation p95 (encrypt/decrypt 4 KiB) | < 100 ms | `cz_key_op_seconds`; k6 |
| API read p95 at 1,000 concurrent VUs | < 300 ms; error rate < 1% | `load/k6.js` (ramp to 1,000 VUs, 5 min sustained) |
| `/api/health` | ≤ 500 ms | smoke |
| Revocation propagation | ≤ 60 s (target ≤ 1 s via pub/sub) | grant revoke test |
| Time to first encrypted object | < 15 min | QUICKSTART walkthrough |
| Lighthouse (web baseline) | ≥ 90 | `web:baseline` |

**Headroom.** The load test reports the max sustained VUs before p95 exceeds 300 ms. The 50k figure is an extrapolation from stateless replicas (ADR-007).

## 10. Test plan

- **Unit and integration** (vitest; real Postgres and Redis via compose for integration; ≥ 80% coverage on `packages/core`, `apps/api`, `apps/trust-core`, `apps/worker`):
  - crypto: round-trip, tamper → INTEGRITY_FAILURE, nonce uniqueness over 10k, hybrid KEM and signatures, PII encryption and blind index
  - policy: deny default, explicit deny wins, purpose/MFA/hours conditions, digest and signature tamper → fail closed
  - auth: limiter before verify, lockout, breached password, minimum 15, CSRF, session rotation, TOTP
  - tenancy: every resource route returns 404 for a foreign id
  - jobs: one effect test per job, plus cron, manual and catch-up triggers (INV-11)
  - billing: signature failure (no DB touch), duplicate event no-op, every transition, idempotency replay and reuse conflict
  - approvals: self-approval forbidden, MFA required, expiry
  - audit: chain verify, tamper detection, checkpoint signature
  - modes (R-2): every mode-gated route → 404 under `readiness` and unchanged under `platform`; `agent` kind refused
  - entitlements (R-2): each limit at max → 403 `PLAN_LIMIT`; each feature off on Free; downgrade keeps reads; usage counts
  - ingest (R-2): every secret-material pattern → 422 with no DB write; non-scanner bearer → 403; scanner bearer on
    any other route → 403; idempotent replay; strict schema; audit event; risk-score table; CBOM validates against the
    CycloneDX 1.6 schema; scanned-plan validation by re-scan (pass, still vulnerable, asset gone)
  - scanner CLI (R-2): TLS probe against a local test server, certificate parsing (RSA, EC, Ed25519), private-key
    files never opened, dry-run output equals the upload body, HTTPS required
  - egress guard (R-3): every IANA special-purpose IPv4/IPv6 range, IPv4-mapped, NAT64 and 6to4 embedded, mixed
    public and private A/AAAA → refused; connect uses the resolved address with SNI; a non-allowlisted port is
    refused; `PROBER_ALLOW_CIDRS` is refused at boot in production; a non-worker client certificate is refused
  - domains (R-3): name validation table (IP literal, public suffix, reserved TLDs, IDNA); TXT match and miss; two
    misses → `lapsed`; re-verify; remove keeps assets; `domains` limit → `PLAN_LIMIT`; scan of an unverified domain
    → 409; a second on-demand scan within 1 h → 429; cross-tenant id → 404
  - external discovery (R-3): CT fixture → hosts under the domain only (label boundary, wildcard stored and not
    probed, 1,000 cap); a CT outage still probes known hosts; a private host is never probed; findings merge with a
    `cz-scan` asset of the same locator; an unreachable host is out of scope; a scanned plan is validated by an
    external re-scan
- **E2E** (Playwright, `e2e/`): `test('W1 …')` through `test('W8 …')`, each walking its workflow through the UI to the terminal step, plus auth and billing flows (W7 ingests a fixture report with a scanner token, then plans, re-ingests and validates; W8 adds a domain, verifies it against a test resolver and probes a loopback TLS server under `PROBER_ALLOW_CIDRS`).
- **Web baseline:** `npm run web:baseline` over every §6 screen × 3 viewports × 2 themes.
- **Load:** `load/k6.js`.
- **Smoke:** `scripts/smoke.mjs` against `https://` runs every W1–W6 workflow via the API, triggers a rotation job and asserts its effect, then logs to `smoke-test.log`.
- **Invariants:** `npm run lint:invariants`.
