# ARCHITECTURE.md — CIPHΞRZER0 // TRUST

Version 0.1.0 · Stage 1 · 2026-10-14 · build_depth comprehensive · scale tier **large** · amended Re-entry R-2 2026-11-10 (§5.8, §5.9; ADR-083, ADR-084)

## 1. Problem, users and scope

**Problem.** CIPHΞRZER0 is a sovereign cryptographic trust plane. Today, key custody, policy, workload and AI-agent identity, and audit evidence live in separate products. CIPHΞRZER0 unifies them so that every sensitive cryptographic operation is:
- authorized by an explicit, signed, versioned policy;
- executed inside an isolated crypto service;
- recorded as tamper-evident evidence that can be anchored externally.

AI analysis (APEX) proposes changes but never enforces them.

**Personas.**

| Role | Can do | Cannot do |
|---|---|---|
| `owner` | Everything an admin can, plus billing | — |
| `admin` | Team, policy, key lifecycle | — |
| `security_officer` | Keys, vault, approvals, APEX | Team, billing |
| `auditor` | Read-only access to evidence, policy and inventory | Any write |
| `developer` | Workloads, credentials and vault use, within policy | — |

Non-human principals:
- **workloads**: service identities holding bearer credentials;
- **AI agents**: workloads of kind `agent` that act only through short-lived grants.

**Key workflows.** RESEARCH Step 3 workflows A–F map to SPEC §5 W1–W6.
- W1 Protect
- W2 Authorized decryption
- W3 Rotation
- W4 AI-agent access
- W5 Quantum migration
- W6 APEX analysis

**Non-goals.** RESEARCH.md names none. Out of scope by ADR:
- real HSM/PKCS#11 drivers (a provider interface only);
- multi-region active-active;
- FIPS-validated module operation;
- issuing certifications.

## 2. System architecture

```
                 Internet / LAN (HTTPS only)
                          │
                ┌─────────▼─────────┐  :80 → 308 redirect only
                │  edge (Caddy 2.11)│  :443 TLS 1.2–1.3, tls internal (local) / ACME (prod)
                │  rate limit, HSTS*│  *HSTS gated on HSTS_ENABLED (trusted cert only)
                └──┬────────────┬───┘
         /api/*    │            │  everything else
        ┌──────────▼───┐   ┌────▼──────────────┐
        │ api (Express5)│   │ web (Next.js 16)  │ standalone, proxy.ts CSP nonce
        │ REST /api/v1  │   │ Zustand UI state  │ PWA shell (sw.js)
        │ authn, tenancy│   └───────────────────┘
        │ billing, apex │
        └──┬─────┬──────┘
   gRPC mTLS│     │ Prisma (primary / replica)        ┌──────────────┐
 (SPIFFE SAN)     │                                    │ worker       │ BullMQ schedulers,
   ┌────────▼───┐ │  ┌──────────┐  ┌───────────────┐   │ DLQ, jobs    │ gRPC mTLS → trust-core
   │ trust-core │ └─►│ postgres │─►│postgres-replica│◄──┤              │ (rotation/rewrap/sign)
   │ gRPC :50051│───►│ primary  │  │ (streaming)    │   └──────┬───────┘
   │ policy PDP │    └──────────┘  └───────────────┘          │
   │ AEAD, KEM, │         ▲                                    │
   │ signatures │    ┌────┴─────┐                              │
   └────────────┘    │  redis   │◄─────────────────────────────┘ queues, rate-limit buckets,
                     └──────────┘                                 revocation pub/sub, cache
   prometheus ◄── scrapes /metrics on api :9100, trust-core :9464, worker :9465
   grafana    ◄── provisioned datasource + "CIPHΞRZER0 Overview" dashboard
```

### 2.1 Layers and modules (monorepo, npm workspaces)

| Path | Responsibility | Owns |
|---|---|---|
| `packages/core` | Shared library, no network listeners | `env.ts` (zod env schemas), `logger.ts` (pino plus redaction), `problem.ts` (RFC 9457), `db/` (Prisma client factory, primary/replica, tenant-scoped repos), `crypto/` (aead, hybrid-kem, hybrid-sig, pqc-provider, hkdf, blind-index, totp, password), `policy/` (evaluator, bundle schema, canonical digest), `audit/` (hash chain, merkle), `trust-client.ts` (gRPC client, mTLS), `queue.ts` (BullMQ names) |
| `packages/core/prisma` | `schema.prisma` and migrations | All tables (§4) |
| `proto/trust.proto` | gRPC contract | `TrustCore` service |
| `apps/trust-core` | **Crypto plane**. The only process holding `TRUST_ROOT_KEY`. It is the policy decision and enforcement point for key operations. It writes `crypto_operations` and audit events for every key operation | Key versions, wrap/unwrap, encrypt/decrypt, sign/verify, policy evaluation |
| `apps/api` | **Control plane gateway**. Handles authentication (sessions, workload/agent bearer tokens), CSRF, tenancy, rate limits, REST `/api/v1`, billing, webhooks, APEX proposals, approvals and evidence queries. Calls trust-core for every crypto operation | Routes, services |
| `apps/worker` | **Async plane**. Runs BullMQ job schedulers and processors (§5.4) | Jobs, DLQ, heartbeats |
| `apps/web` | **Experience plane**. Next.js App Router: public landing, auth screens and the console. Browser to api goes through the same origin | Screens, `components/ui` primitives |
| `apps/scanner` | **Customer-side discovery (R-2)**. The `cz-scan` CLI runs in the customer's network, discovers TLS, certificate and dependency cryptography, and posts metadata to `POST /api/v1/quantum/ingest` over HTTPS. Standalone: it depends only on Node built-ins and zod, never on `@cz/core` (INV-52) | Discovery probes, payload builder |
| `apps/prober` | **Isolated egress (R-3, ADR-089)**. A stateless HTTPS/mTLS service that only the worker can call. It answers `resolve-txt`, `ct-search` and `probe-tls`, and every connection passes `assertPublicAddress()` first (INV-54). It is on the `probe-ctl` and `egress` networks only, holds no DB or Redis credentials, and never imports `@cz/core` (INV-55). It imports `probeTls` from `cz-scan`, so external and customer-side findings are identical | Egress guard, DNS/CT/TLS clients |
| `deploy/` | Caddyfile, Prometheus and Grafana provisioning, mTLS bootstrap script | Delivery |
| `load/` | k6 scripts | Load test |
| `e2e/` | Playwright workflows W1–W6 | E2E |

### 2.2 Data flow — W1 Protect (encrypt)

1. The client (a user session, or a workload holding `Authorization: Bearer czw_…`) calls `POST /api/v1/vault/objects` with `{keyId, label, classification, purpose, plaintextB64}`.
2. api authenticates and resolves the principal `{tenantId, type, id, roles, mfa}`, applies the rate limit, and validates with zod.
3. api calls `TrustCore.Encrypt` over mTLS, passing the principal, request id and payload.
4. trust-core:
   - loads the active, signature-verified policy bundle (cached ≤ 30 s, invalidated by pub/sub);
   - evaluates `key.encrypt`. A deny returns `PERMISSION_DENIED`, and api maps it to 403 `POLICY_DENIED`;
   - loads the active key version;
   - increments its invocation counter atomically;
   - generates a DEK and a 96-bit random nonce;
   - computes AES-256-GCM with AAD `tenant|object|key|version|classification`;
   - wraps the DEK under the KEK, or under the hybrid KEM for PQC profiles;
   - persists `protected_objects`, `crypto_operations` and an audit event in one transaction.
5. api returns `{id, keyId, keyVersion, algorithm, digest}` with 201. The ciphertext stays in VAULT, so plaintext never comes back on encrypt.

### 2.3 Data flow — W2 Authorized decryption

1. api authenticates the workload, user or agent.
2. trust-core verifies the tenant boundary. A foreign object id returns NOT_FOUND, which api maps to 404.
3. trust-core evaluates `key.decrypt` with the stated purpose against the policy and, for agents, the grant scope.
4. It unwraps the DEK, decrypts and verifies the tag. A tag failure is `INTEGRITY_FAILURE`.
5. It records the operation and audit event.
6. Plaintext returns only to the requesting principal, as a `no-store` response.

## 3. Agent and tool orchestration (APEX and AI-agent principals)

**APEX** is a deterministic analysis engine in `apps/worker/src/apex` and `apps/api/src/apex`. It reads only persisted records (`crypto_operations`, `policy_decisions`, `auth_events`, `access_grants`). It computes graph and statistical features:
- deny-rate spikes per principal;
- off-hours decrypt bursts;
- grants unused or excessive versus actual usage;
- keys past their cryptoperiod;
- quantum-vulnerable assets in active use;
- repeated authentication failures.

These feed a weighted risk score, and APEX emits `apex_findings` with the source record ids as evidence. A finding may produce an `apex_proposal`. Proposals:
1. are evaluated by the policy gate (`apex.remediate`);
2. require human approval through `approval_requests` when the risk is high;
3. are executed by api through normal service calls (revoke a grant, suspend a workload, schedule a rotation), never by APEX itself;
4. are validated post-execution by re-running the detector.

**Isolation invariant.** No APEX module imports the trust-core client or crypto modules (INV-4). The `apex` service principal has no `approve` permission.

**AI agents** are workloads of kind `agent`.
- They authenticate with a short-lived grant token `cza_…` with a TTL of 15 minutes or less.
- Grants are issued by a human with the step-up MFA role, scoped to `{actions[], keyIds[], purpose}`.
- Every agent request is checked against both the grant scope and the policy.
- Grants expire automatically through the TTL and the `grant-expiry` job, and can be revoked immediately (DB plus pub/sub, effective in under 60 s).

## 4. Data layer

**Engine and access.**
- PostgreSQL 17 primary with a streaming hot-standby replica.
- Prisma 7.10.0 with `@prisma/adapter-pg`.
- Two clients: `db` (primary, every write and read-after-write) and `dbRead` (replica, used by list and report endpoints: audit, crypto ops, dashboard aggregates, inventory). `DATABASE_REPLICA_URL` is optional; it falls back to the primary.

**Tables and conventions.**
- Every tenant-owned table carries `tenant_id` with a composite index led by `tenant_id`.
- Money is `BIGINT amount_minor` plus `CHAR(3) currency`. Floats are forbidden (INV-8).

| Table | Purpose | Key constraints |
|---|---|---|
| `tenants` | Organisation | `slug` UNIQUE |
| `users` | Human principals; PII encrypted | `email_hash` UNIQUE; `email_enc`, `name_enc` AES-GCM |
| `sessions` | Server sessions | `token_hash` UNIQUE |
| `auth_events` | Login success/failure and MFA events (feeds APEX) | — |
| `invites` | One-time team invites | `token_hash` UNIQUE |
| `workloads` | Service and agent identities | (`tenant_id`, `name`) UNIQUE |
| `workload_credentials` | Hashed bearer credentials | `token_hash` UNIQUE |
| `access_grants` | Agent grants | `token_hash` UNIQUE; `claimed_at` |
| `keys` | Logical keys | (`tenant_id`, `name`) UNIQUE; `claimed_at` (rotation claim) |
| `key_versions` | Wrapped material per version | (`key_id`, `version`) UNIQUE |
| `protected_objects` | VAULT ciphertexts | — |
| `policy_bundles` | Signed bundles | (`tenant_id`, `version`) UNIQUE |
| `policy_decisions` | Every PDP decision | — |
| `crypto_operations` | Every key operation | — |
| `audit_heads` | Per-tenant chain head (row lock) | PK `tenant_id` |
| `audit_events` | Hash-chained log | (`tenant_id`, `seq`) UNIQUE |
| `audit_checkpoints` | Signed Merkle roots | (`tenant_id`, `to_seq`) UNIQUE |
| `approval_requests` | Two-person rule | — |
| `apex_findings`, `apex_proposals` | APEX | — |
| `crypto_assets` | QUANTUM inventory (managed keys and scanned assets; `risk_score`) | (`tenant_id`, `fingerprint`) UNIQUE |
| `scan_reports` | Accepted scanner reports (R-2) | (`tenant_id`, `scan_id`) UNIQUE |
| `migration_plans` | QUANTUM migrations | — |
| `plans` | Billing catalogue | `code` UNIQUE |
| `subscriptions` | Billing state | — |
| `payments` | Billing payments | `provider_ref` UNIQUE |
| `idempotency_keys` | API idempotency | (`tenant_id`, `key`) UNIQUE |
| `webhook_events` | Inbound webhook dedupe | `event_id` UNIQUE; `claimed_at` |
| `job_runs` | Job observability | — |

**Migrations.**
- Prisma Migrate, run by the compose `migrate` one-shot service before api, trust-core and worker start.
- Append-only grants on `audit_events` come from a migration-created `REVOKE UPDATE, DELETE` for the app role. With a single local role this is ⚠ delegated, and documented.

**Tenancy.**
- Repositories take a `TenantContext`. Every query includes `tenantId`.
- Lookups by id use `findFirst({where:{id, tenantId}})`, and a miss returns 404.
- Postgres row-level security is deferred (RESEARCH §6, R2).

## 5. Key subsystems

### 5.1 Cryptography (trust-core)

**Key hierarchy.**
- `TRUST_ROOT_KEY` (32 bytes, base64; never leaves trust-core; never logged).
- Key versions are wrapped under it. Material is random, and private keys are generated per version.
- Wrap: AES-256-GCM, AAD `cz:kv|tenant|key|version`.
- An HSM/KMS provider interface `KeyProvider { wrap, unwrap }` exists. Only the `software` provider is implemented (⚠ delegated).

**Profiles.**

| Profile | Algorithms |
|---|---|
| `AES256_GCM` | Symmetric KEK wraps the DEK |
| `HYBRID_X25519_MLKEM768` | The DEK is wrapped under `HKDF-SHA256(ss_x25519 ‖ ss_mlkem, info="cz-hybrid-v1"‖ctx)`. The KEM and X25519 ciphertexts are stored with the object |
| `HYBRID_ED25519_MLDSA65` | Signing key. Both signatures are required to verify |

**PQC provider.**
- Uses `node:crypto` `ml-kem-768` / `ml-dsa-65` when available (Node 24 / OpenSSL 3.5).
- Otherwise uses `@noble/post-quantum` 0.7.1 (ADR-009).
- Detected once at boot and reported in `/api/health` `checks.pqc`.

**Nonce discipline.**
- Each object gets a fresh DEK, so each DEK sees exactly one GCM invocation.
- KEK wraps use a random 96-bit nonce and a per-version `invocation_count` (`UPDATE … SET invocation_count = invocation_count + 1 RETURNING`).
- At `KEY_MAX_INVOCATIONS` (default 2^30) a version refuses further wraps, and the key is flagged `rotation_due`.

**Signing.** Every tenant has a system signing key `__tenant_signing__` with the `HYBRID_ED25519_MLDSA65` profile, created at tenant bootstrap. It signs policy bundles and audit checkpoints.

### 5.2 PII at rest

- `users.email` and `users.display_name` are encrypted with AES-256-GCM under `PII_ENCRYPTION_KEY` (api env), with AAD `cz:pii|table|column|rowId`.
- `email_hash` is `HMAC-SHA256(PII_BLIND_INDEX_KEY, lower(email))` and is used for login lookup.
- Tests: round-trip, tamper and nonce uniqueness.

### 5.3 Policy engine

**Bundle.**
- A canonical JSON document: `{schema:1, defaultEffect:"deny", rules:[{id, effect, principals:{types,roles,ids}, actions, resources:{keyIds, classifications}, conditions:{purposes, requireMfa, hoursUtc}}]}`.
- Its digest is SHA-256 over canonical JSON (sorted keys).
- It is signed by the tenant signing key: Ed25519 plus ML-DSA-65.

**Evaluation.**
- An explicit deny wins.
- Otherwise the first matching allow wins.
- Otherwise the result is deny, with reason `no_matching_rule`.
- Evaluation is pure, synchronous and in-process: under 1 ms typical, budget p95 under 25 ms.

**Lifecycle.**
- `draft` → `pending_approval` (signature produced) → `active` (a second approver who is not the author, with step-up MFA) → `superseded`.
- A `rejected` state is also possible.
- Exactly one bundle per tenant is active.
- trust-core refuses a bundle whose signature or digest fails (fail closed: no verified active bundle means deny everything, `POLICY_UNAVAILABLE`).

**Bootstrap.** Version 1 is created and activated at tenant creation, with the two-person exemption logged as an audit event.

### 5.4 Background jobs (has_scheduled_work)

All jobs run on BullMQ 6 job schedulers (`upsertJobScheduler`). The worker has concurrency 4. Failed jobs retry 3 times with exponential backoff, then move to the `cz-dead-letter` queue. Each run records a `job_runs` row with `{processed, skipped, failed}`. The worker heartbeat key `cz:worker:heartbeat` (TTL 30 s) is surfaced in `/api/health` `checks.worker`.

The clock is injected (`Clock` interface; **UTC**).

| Job | Trigger | Per-tenant function | Claim | Effect asserted by test |
|---|---|---|---|---|
| `key-rotation` | cron `*/5 * * * *` UTC; manual `POST /api/v1/ops/jobs/key-rotation/run` | `runKeyRotationForOrg(orgId)` | `UPDATE keys SET claimed_at=now() WHERE id=? AND claimed_at IS NULL AND next_rotation_at<=now()` | New active version; old version `deactivated`; `next_rotation_at` advanced |
| `rewrap` | enqueued by rotation; manual | `runRewrapForOrg(orgId)` | per object `rewrapped_at IS NULL` batch | Objects reference the new version; old version `retired` when there are zero dependents |
| `grant-expiry` | every 30 s | `runGrantExpiryForOrg(orgId)` | `claimed_at IS NULL AND expires_at<=now()` | Grant `expired`; Redis revocation published |
| `audit-checkpoint` | cron `*/10 * * * *` | `runAuditCheckpointForOrg(orgId)` | (`tenant_id`, `to_seq`) UNIQUE | Signed checkpoint row that verifies |
| `apex-scan` | cron `*/15 * * * *` | `runApexScanForOrg(orgId)` | dedupe by finding fingerprint | Findings created from seeded anomalies |
| `quantum-inventory` | cron `0 * * * *` | `runQuantumInventoryForOrg(orgId)` | upsert by fingerprint | Assets reflect managed keys |
| `retention-purge` | cron `30 3 * * *` | `runRetentionPurge()` | — | `webhook_events` and `idempotency_keys` older than `IDEMPOTENCY_RETENTION_HOURS` (default 720, minimum 72) deleted |
| `approval-expiry` | cron `*/5 * * * *` | `runApprovalExpiryForOrg(orgId)` | `status='pending' AND expires_at<=now()` | Request `expired` |
| `domain-verify` | cron `15 4 * * *`; manual `POST /api/v1/quantum/domains/:id/verify` (one domain) | `runDomainVerifyForOrg(orgId)` | `job_runs` (job, tenant, day) UNIQUE; per domain `UPDATE … WHERE status=<read status>` | TXT match → `verified`; second consecutive miss on a verified domain → `lapsed`; a prober lookup failure changes nothing (ADR-091) |
| `external-discovery` | cron `45 4 * * *`; manual `POST /api/v1/quantum/domains/:id/scan` (one domain, one per hour) | `runExternalDiscoveryForOrg(orgId)` | `job_runs` (job, tenant, day) UNIQUE; `scan_reports` (tenant, scan_id) UNIQUE with scan id `ext:<domain>:<window>`; hosts (tenant, hostname) UNIQUE | Each `verified` domain (re-read, INV-56): apex, `www` and CT hosts upserted (≤ 1,000), non-wildcard non-private hosts probed 2 at a time, one `external` report ingested via `ingestFindings()` (INV-57), `last_scan_at` set (ADR-092) |

**Catch-up policy.**
- At startup, the worker runs each sweep once immediately. Every sweep is idempotent and claim-based, so missed windows are caught up.
- On SIGTERM, the worker closes and stops taking jobs. The current job finishes, or is returned to wait by BullMQ's stalled-job recovery.

### 5.5 Evidence (audit)

**Chain.**
- `appendAudit(tx, event)` locks `audit_heads` with `SELECT … FOR UPDATE` and computes `seq = last+1` and `hash = SHA256(prev_hash ‖ canonical(event))`.
- Genesis `prev_hash` is 64 zeros.

**Checkpoints.**
- A Merkle root per RFC 9162 (leaf `0x00`, node `0x01`) over the event hashes in `(from_seq, to_seq]`, signed with the hybrid signature.
- `GET /api/v1/evidence/verify` recomputes the chain and the checkpoints.
- `GET /api/v1/evidence/export` streams NDJSON plus the checkpoints, for external anchoring.

### 5.6 Billing (has_payments)

**Catalogue and amounts.**
- Plans are seeded: `sovereign-team` 49_000 minor units USD per month, and `sovereign-enterprise` 249_000.
- Amounts are `BIGINT amount_minor` with `CHAR(3) currency`.

**Checkout.**
- `POST /api/v1/billing/checkout` requires `Idempotency-Key`. It creates a `pending` subscription and payment.
- With Stripe configured it returns the processor session URL.
- In `BILLING_MODE=simulated` it returns a `simulate` URL. The simulator endpoint signs a webhook with `BILLING_WEBHOOK_SECRET` exactly as Stripe does and posts it.

**Webhook.**
- `POST /api/v1/billing/webhook` takes the raw body and verifies the `Stripe-Signature` (t, v1; 300 s tolerance) **before** any DB read.
- It inserts `webhook_events` (`event_id` UNIQUE, so duplicates return 200 and do nothing), then enqueues processing.
- Transitions: `pending → succeeded → refunded | disputed | charged_back`, plus `pending → failed`.
- Payment state changes only in the webhook processor (INV-15).

**Card data.** PAN and CVV are never accepted or stored. Checkout is hosted by the processor.

### 5.7 Approvals (separation of duties)

`approval_requests` has kind `policy_activate | key_destroy | apex_remediation` (ADR-067), requester, approver and `expires_at` (24 h).
Executors are idempotent: the `policy_activate` executor resumes a bundle trust-core already activated and writes its
`policy.activate` audit event exactly once (ADR-088).

**Approval rule.** The approver must:
- not be the requester;
- hold a role allowed for the kind;
- have passed step-up MFA (TOTP verified within 15 minutes).

The rule is enforced in the service and by a DB CHECK (`approved_by <> requested_by`).

### 5.8 Quantum readiness: scanner, ingest, risk, CBOM (R-2, ADR-083)

**Flow.**

```
customer network                              │  CIPHΞRZER0
 cz-scan ──probe──► TLS endpoints             │
        ──read───► cert files (no key files)  │
        ──read───► lockfiles / manifests      │
        ──build──► payload (metadata only) ─HTTPS, czw_ scanner bearer─► edge ─► api
          --dry-run prints it instead          │   authenticate (kind=scanner → ingest only)
                                               │   zod strict → rejectSecretMaterial() → tx:
                                               │     scan_reports + crypto_assets upsert + appendAudit
```

**Ingest.**
- `rejectSecretMaterial(report)` runs before any DB access (INV-50). It is a pure function in
  `apps/api/src/lib/secret-guard.ts`. The scanner runs its own copy of the same patterns before upload (it cannot
  import platform code, INV-52), and one shared fixture file (`test/fixtures/secret-material.json`) is asserted by
  both test suites, so the two copies cannot drift.
- One transaction inserts the `scan_reports` row (the `(tenant_id, scan_id)` UNIQUE makes replays a no-op that
  returns the first result), upserts each finding by fingerprint `sha256(tenant|source|locator|algorithm)`, sets
  `last_seen_at` and `scan_report_id`, and appends one audit event.
- `headroom(tx, tenant, 'assets', now)` runs in the same transaction (§5.9). It takes the per-tenant `assets` lock
  and returns the room left. Findings for new assets beyond it are refused with `PLAN_LIMIT`, and the rest of the
  report is accepted (partial accept).

**Risk score.** `riskScore(asset, now)` in `packages/core/src/quantum/risk.ts` is pure and integer-only, clamped to
0–100:

| Component | Points |
|---|---|
| Algorithm class: Shor-vulnerable public key (RSA, DSA, DH, ECDSA, ECDH, EdDSA, X25519/X448 alone) | 60 |
| Algorithm class: classically weak (SHA-1, MD5, 3DES, RC4, RSA < 2048) | +25 on top of its class |
| Algorithm class: symmetric/hash at 128-bit security (AES-128) | 15 |
| Algorithm class: PQC or hybrid (ML-KEM, ML-DSA, SLH-DSA, X25519MLKEM768), AES-256, SHA-256+ | 0 |
| Exposure: key exchange on TLS (harvest now, decrypt later) | +20 |
| Lifetime: certificate `not_after` after 2030-12-31 (NIST IR 8547 deprecation) | +10 |
| Lifetime: certificate `not_after` after 2035-12-31 (disallowed) | +10 more |

`quantum_vulnerable` stays the boolean "Shor-vulnerable" flag. Exposure on `GET /v1/quantum/assets` becomes the
usage-weighted share for managed keys and the count-weighted share for scanned assets.

**CBOM.** `GET /v1/quantum/cbom` streams CycloneDX 1.6 JSON. Each asset is a component of type
`cryptographic-asset` with `cryptoProperties.assetType` `algorithm`, `certificate` or `protocol`, and properties
`cz:riskScore`, `cz:source` and `cz:locator`. The BOM metadata names the tenant slug and the latest scan report.

**Scanned-asset migration plans.** A plan holds either managed or scanned assets. Managed plans keep the W5 path.
A scanned plan advances `draft → testing → migrating` by an operator action. `migrating → validated` needs a
`scan_reports` row finished after `migrating` began in which every plan asset is either seen with a risk class of 0,
or not seen while its locator was in scope. Otherwise advance answers 409 `INVALID_TRANSITION` with the blocking
asset ids. `interop_result` records the validating scan report id.

### 5.9 Product modes and entitlements (R-2, ADR-083, ADR-084)

**Modes.** `PRODUCT_MODE` (env, default `platform`) is read once at boot.
- api: `modeGate(mode)` (`apps/api/src/lib/mode.ts`) is mounted on `/api` before authentication. It matches the
  method and the lower-cased path against a fixed list (the vault router, key create/rotate/disable/enable/destroy
  and grant issue) and answers 404 `NOT_FOUND` in `readiness`.
- api, indirect paths (ADR-086): a managed migration plan never gives a customer key a successor, re-seals its data
  or retires it in `readiness` (the step is `failed`), and the `key_destroy` executor refuses with 409, so an
  approval raised in `platform` cannot destroy a key after the switch. `rotateKey` refuses a non-system key in
  `readiness` for every caller, and APEX does not propose `rotate_key` for one, so an approved APEX remediation
  cannot rotate a customer key either (ADR-087).
- web: the shell reads `mode` from `GET /v1/auth/session` and drops the nav entries.
- worker: in `readiness`, `rewrap` and `key-rotation` only touch system keys.
- trust-core is unchanged. It still holds the root key for the tenant signing key. In `readiness` the API asks it to
  wrap, unwrap or destroy customer key material only through those paths, and all of them are closed.

**Entitlements.** `packages/core/src/entitlements.ts` exports `PLAN_ENTITLEMENTS`, keyed by plan code with `free`
as the fallback. Each entry has `limits` (an integer or `null` for unlimited) and `features` (booleans).
`assertEntitlement(tx, tenantId, limit, { now })` resolves the plan from the tenant's `active` or `past_due`
subscription, takes the per-tenant lock for that limit, counts current usage with a tenant-scoped `count()`, and
throws `problems.planLimit(...)` (403 `PLAN_LIMIT`) when the next row would exceed the maximum.
`assertFeature(db, tenantId, feature)` throws the same problem when a plan feature is off. `headroom(tx, tenantId,
limit, now)` returns the room left for a batch (scanner ingest, §5.8). Every creating service calls it inside its own
transaction (INV-51). The plan map is code, not data, so changing a limit is a reviewed commit.

### 5.10 Clientless external discovery (R-3, ADR-089)

**Flow.**

```
 default network (no internet route needed)        │ probe-ctl (internal) │ egress (internet)
 api ── POST /quantum/domains[/:id/verify|scan] ─►  │                      │
        enqueue interactive job (ADR-062)           │                      │
 worker  domain-verify ───── mTLS ─────────────────►│ prober ──resolve TXT─┼─► DNS
         external-discovery ─ mTLS ────────────────►│   ct-search ─────────┼─► CT search (CT_SEARCH_URL)
           assertDomainVerified() (INV-56)          │   probe-tls:         │
           hosts = apex + www + CT + manual         │    resolve A/AAAA    │
           ◄── findings[] ──────────────────────────│    assertPublicAddress() (INV-54)
           ingestFindings(tx, origin=external)      │    tls.connect(ip, SNI=host) ─┼─► customer host:port
             rejectSecretMaterial() first (INV-57)  │    handshake, close  │
             upsert assets · scan_report · audit    │                      │
```

**Prober.** `apps/prober` is a small `node:https` server with `requestCert: true`. It checks the client
certificate's SPIFFE SAN against `PROBER_ALLOWED_SPIFFE_IDS` (§6.5). Its three endpoints:
- `POST /resolve-txt {name}` returns the TXT strings.
- `POST /ct-search {domain}` returns the distinct DNS names from the CT source. The response is capped at 10 MB and
  1,000 names, and the request times out. The CT source is called through `fetch` with `redirect: "error"`; it is a
  fixed configured URL, not a tenant input. Its host still goes through `assertPublicAddress()` before each search,
  and a refusal is `502 CT_UPSTREAM`. `fetch` resolves the name again, so this is a check-then-fetch with a
  rebinding window; the guard is defence in depth here, and the network placement (no route to `default`) is the
  control (ADR-090).
- `POST /probe-tls {host, ports[]}` returns `{status: reachable | unreachable | private, findings[]}`. The only TLS
  connect is `connectChecked(pub: PublicAddress, …)`, which dials the guard's checked address with SNI set to the
  host; a structure test keeps it the single `probeTls` call site.

Errors are RFC 9457 `application/problem+json` with the SPEC §7 fields (`type`, `title`, `status`, `detail`,
`instance`, `code`, `requestId`, and `errors[]` for `VALIDATION_FAILED`), so the worker parses them like api errors.
The prober adds `INVALID_TARGET` (400), `CT_UPSTREAM` (502), `UNRESOLVABLE` (502) and `BUSY` (503).

The prober keeps no state between calls. Concurrency is a semaphore (`PROBER_CONCURRENCY`) plus a per-domain cap of
2 supplied by the worker. `PROBER_ALLOW_CIDRS` (tests only, refused at boot in production) adds CIDRs to the allow
side of the guard.

**Egress guard.** `assertPublicAddress(host)` in `apps/prober/src/egress.ts`:
1. Refuses IP literals and names that are not valid DNS names.
2. Resolves A and AAAA with `dns.promises.Resolver` (`PROBER_DNS_SERVERS`).
3. Unwraps embedded IPv4 (`::ffff:0:0/96`, `64:ff9b::/96`, `2002::/16`).
4. Checks every address against a `net.BlockList` built from the IANA IPv4 and IPv6 special-purpose registries, plus
   `100.64.0.0/10` and `0.0.0.0/8`.
5. Returns one public address, or throws `PrivateAddress` if any address is blocked. Mixed answers are refused as a
   whole.

`probeTls(target, timeout, { address })` then connects to that address with `servername: host`. A rebinding answer
after the check is therefore never used. Ports outside the allowlist are refused before resolution.

**Domains.** `monitored_domains.status` moves `pending → verified → lapsed ⇄ verified`, and any of them `→ removed`.
`domain-verify` claims one window per tenant per day (INV-46):
- For each non-removed domain it asks the prober for `_cipherzero-challenge.<name>` and matches
  `cz-domain-verification=<token>` exactly.
- A match sets `verified`, `verified_at` and `consecutive_misses = 0`.
- A miss increments `consecutive_misses`. At 2 a `verified` domain becomes `lapsed`, and `pending` stays `pending`.
- Each status change appends an audit event (`quantum.domain.verified`, `.lapsed`, `.removed`).

The interactive variant runs the same code for one domain.

**Discovery and ingest.** `external-discovery` claims one window per tenant per day:
1. For each `verified` domain (`assertDomainVerified()` re-reads the status in the job, INV-56), upsert hosts: the
   apex, `www`, CT names that end in `.<domain>` on a label boundary, and manual hosts. Wildcards go in as `wildcard`.
   The total is capped at 1,000 per domain, newest CT entries first.
2. Probe each non-wildcard host on the domain's ports, hosts probed longest ago first. Set the host `status` and
   `last_probe_at`. A tenant's run starts no probe after its 20-minute budget (`RUN_BUDGET_MS`); it records and ingests
   what it probed, and the next run picks up the rest. Domains scanned longest ago go first. One worker runs at most
   2 external-discovery jobs at once (`EXTERNAL_DISCOVERY_SLOTS`); a job past that is delayed, not failed (ADR-095).
3. Build one report per domain: `scanId = ext:<domainId>:<windowStart>`, scope = locators of reachable hosts only,
   and the findings.
4. Call `ingestFindings(tx, tenantId, { origin: "external", domainId, workloadId: null }, report, now)`.

`ingestFindings()` (`packages/core/src/quantum/ingest.ts`, moved from the API service with the secret guard) runs
`rejectSecretMaterial()` before its first DB call (INV-57). It then upserts by the R-2 fingerprint, applies
`headroom('assets')`, writes `scan_reports` and appends `quantum.scan.ingested`. A host seen by both `cz-scan`
and the prober is one asset, because the locator `tls://host:port` and the algorithm are the same.

Validating scanned migration plans (§5.8) needs no change: an external report is a scan report with a scope.

**Compose.** Two new networks: `probe-ctl` (`internal: true`; worker and prober) and `egress` (prober only).
`pki-init` issues `prober.crt` with SAN `spiffe://cipherzero.local/ns/default/sa/prober`. The worker gains
`PROBER_URL` and joins `probe-ctl`. The prober runs as a non-root user with a read-only root filesystem, no
volumes besides `/pki:ro`, and a memory limit.

## 6. Delivery, security and operations

### 6.1 Runtime

- `node:24-alpine` multi-stage images, run as non-root `node`, with `tini` as PID 1 and a `HEALTHCHECK`.
- Node runs with `--max-http-header-size=32768`.
- `GIT_COMMIT` and `APP_VERSION` are build args baked into the image and surfaced by `/api/health`.

### 6.2 Edge and TLS

**Caddy (`deploy/caddy/Caddyfile`).**
- `{$SITE_ADDRESS:localhost}` with `tls internal` locally. Production uses a real domain, which gets an ACME (Let's Encrypt) certificate automatically.
- `:80` redirects to `:443` with a 308.
- Routing: `/api/*` goes to `api:4000`; everything else goes to `web:3000`.
- Caddy strips `Server` and `X-Powered-By`.

**HSTS.**
- Emitted only when `HSTS_ENABLED=true` (trusted certificate). Never sent on `tls internal` (reference/security-headers.md; ADR-017).

**CSP.**
- Profile N (nonce plus `strict-dynamic`), minted in `apps/web/src/proxy.ts` on both request and response.
- The api sends `helmet` headers on JSON responses.

### 6.3 Rate limiting and quotas

**Edge.** Caddy request-body limit of 1 MB and timeouts.

**Application.** A Redis token bucket (`packages/core/src/ratelimit.ts`) keyed per IP, per principal and per tenant, with cost weights:

| Path | Cost |
|---|---|
| Login | 10 |
| Crypto operations | 2 |
| Reads | 1 |

**Login limiter.** Applied **before** the argon2 compare: 10 attempts per 15 minutes per IP and email hash. Lockout after 20 consecutive failures for 15 minutes (well under the NIST ceiling of 100).

**Exceeding a limit.** Returns 429 problem+json with `Retry-After`.

**Load testing.** `RATE_LIMIT_SCALE` (default 1) multiplies the bucket capacity. It is restored and proven after the load test.

### 6.4 Observability

- **Logs.** pino JSON logs carry `requestId` (inbound `x-request-id` or a new UUID), `tenantId` and `principal`. The redaction list covers the service-floor list plus `plaintext`, `plaintextB64`, `ciphertext`, `wrappedDek`, `totpSecret` and `rootKey`.
- **Metrics (prom-client).** `http_request_duration_seconds`, `cz_policy_eval_seconds`, `cz_key_op_seconds`, `cz_jobs_total{job,result}`, `cz_queue_depth`, `cz_revocation_latency_seconds`.
- **STΛRGΛZΞR (ADR-080).** An `observer` workload's bearer reads the tenant's observation snapshot and the audit-chain event stream from the replica (`/v1/observe/*`), with provenance and an observed/inferred basis per item. It never receives key material, ciphertext, PII or audit payloads.
- **Grafana.** A provisioned dashboard shows p95 policy evaluation against the 25 ms target and p95 key operations against the 100 ms target.

### 6.5 mTLS and workload identity

- `deploy/pki/bootstrap.mjs`, run by the compose `pki-init` one-shot service, generates a private CA (ECDSA P-256) and leaf certificates for trust-core, api and worker. Each carries a SAN URI `spiffe://cipherzero.local/ns/default/sa/<service>`. The full set, with the CA key, stays in the `pki` volume, which only pki-init mounts. pki-init exports each service's `ca.crt`, certificate and key into its own `pki-<service>` volume (`--export`), and each service mounts only that one, read-only (ADR-094).
- The trust-core gRPC server requires client certificates (`checkClientCertificate=true`) and authorizes the peer SPIFFE ID against an allow-list (`api`, `worker`).
- Development falls back to an in-repo generated dev CA under `.pki/`, which is git-ignored.

### 6.6 Offline and PWA

- `public/manifest.webmanifest` and `public/sw.js` precache the shell and `/offline`, cache-first for `/_next/static`.
- `/api/*` is network-only and never cached.
- When offline, mutation buttons are disabled and a banner reads "Crypto operations are unavailable offline (fail-closed)".

## 7. Threat model (STRIDE) and alternatives

The full STRIDE table is in RESEARCH.md §5.1. Summary of the controls, with the assets they protect:

| Asset | Threats | Controls (where) |
|---|---|---|
| Root key | Exfiltration, logging | Only in trust-core env; redaction; INV-3 forbids `TRUST_ROOT_KEY` outside trust-core and core env |
| Key material | Misuse, cross-tenant use | PDP in trust-core; AAD binds tenant, key and version; 404 for foreign ids |
| Plaintext | Leakage | `no-store`, never logged, returned only to the authorised principal |
| Policy | Forgery, unilateral change | Hybrid signature; two-person activation; fail closed |
| Audit | Tamper, repudiation | Hash chain, signed checkpoints, verify and export |
| Sessions | Theft, CSRF | `__Host-` cookie, HttpOnly, SameSite=Strict, CSRF double-submit, session rotation on login and MFA |
| Agent grants | Escalation, persistence | Scope plus policy; TTL of 15 minutes or less; revocation in under 60 s |
| APEX | Autonomous harm | Proposal-only; policy gate; human approval; no crypto imports (INV-4) |
| Scanner ingest (R-2) | Customer secrets uploaded by mistake; spoofed reports; flooding | Secret-material guard before DB (INV-50); scanner kind reaches ingest only; revocable credential; strict schema, 5,000-finding and 1 MB caps; tenant rate limit; idempotent by scan id; audited |
| Scanner CLI (R-2) | Reading private keys; token leakage | Key-file patterns never opened; HTTPS-only upload; dry-run review; standalone, with no access to platform code (INV-52) |
| Plan limits (R-2) | Bypass via concurrency or downgrade | Check inside the creating transaction (INV-51); downgrade keeps reads and blocks creates |
| External prober (R-3) | SSRF into the internal network (rebinding, embedded IPv4, mixed records, redirects); using the platform to scan third parties; probe flooding | Network isolation: the prober is not on the default network and holds no DB/Redis credentials (INV-55); egress guard before every connect (INV-54); only scanner and prober open sockets (INV-53); connect by resolved IP with SNI; port allowlist; handshake only, no redirects; verified domains only (INV-56); per-domain and global concurrency; hourly on-demand cap |
| Domain verification (R-3) | Claiming a domain you do not control; keeping access after a domain changes hands | DNS TXT with a per-tenant random token; daily re-check, lapse after two misses; audited status changes |
| CT search provider (R-3) | Leaks which domains a tenant monitors; malformed or huge responses | Documented in SPEC §8 and can be disabled; response size and name caps; names filtered to the verified domain on a label boundary |
| Availability | Floods | Edge limit plus token buckets; queue DLQ; health-gated restarts |

### Alternatives considered (per major decision)

| Decision | Chosen | Alternatives | Why |
|---|---|---|---|
| Browser API | REST gateway | gRPC-Web with Envoy; pure gRPC | Browsers lack HTTP/2 trailers. Envoy adds a hop and CSP/cookie friction (ADR-001) |
| Policy engine | Embedded signed-JSON PDP | OPA sidecar; Cedar WASM | Latency budget, sovereignty and zero dependencies |
| Crypto placement | Separate trust-core process | In-process in api | Isolation from AI and gateway code; root key confined |
| Audit integrity | Hash chain plus signed Merkle checkpoints | DB triggers only; external Rekor | Sovereign and offline-verifiable, exportable for anchoring |
| PQC | Hybrid (classical plus ML-KEM/ML-DSA) | PQC-only; classical-only | NIST transition guidance favours hybrid during migration |
| Scale | Replica, cache, stateless replicas, DLQ on one host | Kubernetes HPA | A local deploy is the shipped artifact. Topology is portable to k8s (documented) |
| Sessions | Server-side opaque sessions | JWT | Instant revocation; no token replay after logout |
| Billing | Stripe with a simulated mode | Build our own processor | PCI scope reduction |

## 8. Dependencies (pinned, verified 2026-10-14)

| Package | Version |
|---|---|
| next | 16.4.1 |
| react / react-dom | 19.x (peer of Next 16) |
| zustand | 5.0.15 |
| express | 5.3.0 |
| helmet | 8.x |
| @grpc/grpc-js | 1.14.6 |
| @grpc/proto-loader | 0.8.x |
| prisma / @prisma/client / @prisma/adapter-pg | 7.10.0 |
| pg | 8.x |
| bullmq | 6.3.15 |
| ioredis | 6.0.0 |
| zod | 4.6.7 |
| pino | 10.4.0 |
| prom-client | 15.1.3 |
| argon2 | 0.45.1 |
| @noble/post-quantum | 0.7.1 |
| @noble/curves / @noble/hashes | latest stable at install |

Tooling: vitest 5.0.4, @playwright/test 1.64.1, eslint 10.12.0 with typescript-eslint 8.71.2, eslint-plugin-jsx-a11y and eslint-plugin-react-hooks, prettier 3, tsup 8, k6 (container `grafana/k6:0.57.0`).

Images: `node:24-alpine`, `postgres:17-alpine`, `redis:7-alpine`, `caddy:2.11-alpine`, `prom/prometheus:v3.5.0`, `grafana/grafana:12.1.0`.

## 8.1 Risks and implementation notes

See RESEARCH.md §5 (R1–R12). Implementation notes:
- Prisma 7 does not auto-load `.env`; every entrypoint imports `packages/core/src/env.ts` first.
- BullMQ `repeat` is removed in v6; only `upsertJobScheduler` is used.
- Next 16 uses `proxy.ts`, not `middleware.ts`.
- Express 5 `req.query` is read-only, so validated copies go in `res.locals`.

## 9. Architectural Invariants

Each rule maps 1:1 to `invariants.json` by id.

| ID | Rule | Reference | Check |
|---|---|---|---|
| INV-1 | Governance ledger is append-only and covers every post-plan commit | GOVERN.md | governance |
| INV-2 | Web Delivery Baseline report passes and is fresh | WEB.md | web-baseline |
| INV-3 | `TRUST_ROOT_KEY` is read only by trust-core and the env schema | §5.1 | forbidden-pattern outside `apps/trust-core/**`, `packages/core/src/env.ts`, docs, compose/env example |
| INV-4 | APEX and web code never import the trust-core client or crypto primitives | §3 | forbidden-pattern `from .@cz/core/(trust-client\|crypto)\b` outside the key-operation surfaces (trust-core, core, api services/routes/lib, worker jobs) |
| INV-5 | Health route exists | service floor | required-file `apps/api/src/routes/health.ts` |
| INV-6 | `.env.example` exists | service floor | required-file |
| INV-7 | Env schema module exists | service floor | required-file `packages/core/src/env.ts` |
| INV-8 | No floating-point money columns in migrations | §5.6 | forbidden-pattern `\b(FLOAT\|REAL\|DOUBLE PRECISION)\b` |
| INV-9 | Idempotency keys unique per tenant | §5.6 | required-unique-constraint `idempotency_keys(tenant_id,key)` |
| INV-10 | Webhook events deduplicated by event id | §5.6 | required-unique-constraint `webhook_events(event_id)` |
| INV-11 | Every scheduled trigger has an effect test | §5.4 | manual |
| INV-12 | Prisma client is constructed only in `packages/core/src/db` | §4 | forbidden-pattern `new PrismaClient\(` outside `packages/core/src/db/**` |
| INV-13 | Audit events are written only through `appendAudit` | §5.5 | forbidden-pattern `auditEvent\.create\(` outside `packages/core/src/audit/**` |
| INV-14 | Login limiter runs before password verification | §6.3 | boundary-order `first: loginLimiter`, `second: verifyPassword\(` in `apps/api/src/services/auth.ts` |
| INV-15 | Webhook signature verified before any DB access in the webhook route | §5.6 | boundary-order `first: verifyWebhookSignature\(`, `second: (db\|prisma)\.` in `apps/api/src/routes/billing-webhook.ts` |
| INV-16 | No `Math.random` in crypto, auth or token code | §5.1 | forbidden-pattern `Math\.random\(` outside web UI and tests |
| INV-17 | No CSP disabled / `unsafe-inline` scripts | §6.2 | forbidden-pattern `script-src[^;\n]*'unsafe-inline'` |
| INV-18 | UI coverage of SPEC §6 screens, endpoints and W1–W6 | SPEC §6 | ui-coverage |
| INV-19 | Web baseline runner pinned | WEB.md | required-file `scripts/web-baseline.mjs` |
| INV-20..INV-42 | Every status a SPEC §2 transition admits is written by a writer surface: `succeeded`, `failed`, `refunded`, `disputed`, `charged_back`, `past_due`, `canceled`, `pending_approval`, `superseded`, `rejected`, `deactivated`, `retired`, `disabled`, `destroyed`, `revoked`, `expired`, `approved`, `executed`, `validated`, `testing`, `migrating`, `suspended`, `active` | SPEC §2 state machines | required-pattern `status:\s*['"]<s>['"]` in `apps/*/src/**` (routes, services, workers, trust-core), tests and web excluded |
| INV-43 | Approver differs from requester | §5.7 | required-pattern `approved_by"?\s*<>\s*"?requested_by` in `packages/core/prisma/migrations/**` |
| INV-44 | gRPC server requires client certificates (mTLS) | §6.5 | required-pattern `createSsl\(…, true)` in `apps/trust-core/src/**` |
| INV-45 | HSTS only behind the `HSTS_ENABLED` flag | §6.2 | boundary-order `first: HSTS_ENABLED`, `second: Strict-Transport-Security` in `apps/web/src/proxy.ts` |
| INV-46 | Job runs claimed once per job, tenant and window (job-claim table) | §5.4 | required-unique-constraint `job_runs(job, tenant_id, window_start)` |
| INV-47 | No Prisma `Float` fields | §5.6 | forbidden-pattern `^[ \t]*\w+[ \t]+Float\??(\s\|$)` outside app code and docs |
| INV-48 | ApexFinding status `dismissed` is written by a writer surface | SPEC §2.1 | required-pattern in `apps/*/src/**` |
| INV-49 | ApexFinding status `resolved` is written by a writer surface | SPEC §2.1 | required-pattern in `apps/*/src/**` |
| INV-50 | Scanner ingest checks for secret material before any DB access (R-2) | §5.8 | boundary-order `first: rejectSecretMaterial\(`, `second: \b(db\|prisma\|tx)\.` in `apps/api/src/routes/quantum-ingest.ts` |
| INV-51 | Plan limits are enforced by the API (R-2) | §5.9 | required-pattern `assertEntitlement\(` in `apps/api/src/services/**`, tests excluded |
| INV-52 | The scanner CLI never imports `@cz/core` (R-2) | §2.1 | forbidden-pattern `from ['"]@cz/core` outside every workspace except `apps/scanner` |
| INV-53 | Only the scanner and the prober open raw sockets or DNS (R-3) | §5.10 | forbidden-pattern `(from \|require\(\|import\()['"](node:)?(tls\|net\|dns\|dgram)(/promises)?['"]` outside `apps/scanner/**`, `apps/prober/**`, tests, the two dev scripts that probe ports (`scripts/ports.mjs`, `scripts/web-baseline.mjs`) and the `Dockerfile` HEALTHCHECK (ADR-090) |
| INV-54 | The prober checks the address before every connect (R-3) | §5.10 | boundary-order `first: assertPublicAddress\(`, `second: probeTls\(` in `apps/prober/src/routes.ts` |
| INV-55 | The prober never imports `@cz/core` or Prisma (R-3) | §2.1 | forbidden-pattern `from ['"](@cz/core\|@prisma/client)` outside the core consumers (`apps/api`, `apps/trust-core`, `apps/worker`, `apps/web`, `apps/scanner`, `packages/**`) and dev tooling. It covers the prober's tests too (ADR-090) |
| INV-56 | External discovery probes only verified domains (R-3) | §5.10 | required-pattern `assertDomainVerified\(` in `apps/worker/src/jobs/external-discovery.ts` |
| INV-57 | `ingestFindings()` checks for secret material before any DB access (R-3) | §5.10 | boundary-order `first: rejectSecretMaterial\(`, `second: \b(db\|prisma\|tx)\.` in `packages/core/src/quantum/ingest.ts` |
| INV-58 | MonitoredDomain status `verified` is written by a writer surface (R-3) | SPEC §2.1 | required-pattern in `apps/*/src/**` |
| INV-59 | MonitoredDomain status `lapsed` is written by a writer surface (R-3) | SPEC §2.1 | required-pattern in `apps/*/src/**` |
| INV-60 | MonitoredDomain status `removed` is written by a writer surface (R-3) | SPEC §2.1 | required-pattern in `apps/*/src/**` |

Manual entries: one (INV-11), within the cap of 3 entries or 20 %.

**R-2 note.** INV-50 to INV-52 are added to `invariants.json` by the feature that creates their target (F-23
INV-50, F-22 INV-51, F-24 INV-52), so each is proven the moment it lands and the 1:1 mapping holds at every
commit.

**R-3 note.** INV-53 to INV-60 follow the same rule: F-25 adds INV-53 to INV-55, F-26 INV-58 to INV-60,
and F-27 INV-56 and INV-57. They enter `invariants.json` with their targets and are proven with `--prove` at that
commit.

`node scripts/invariant-lint.mjs --prove` at Stage 2 close: `49 total / 48 machine (48 proven, 0 unproven) / 1 manual (skipped -- Reviewer Gate item 8)` (47 at Stage 1, +2 status entries from SPEC §2.1 at Stage 2).
