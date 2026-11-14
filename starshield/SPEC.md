# SPEC.md — STΛRSHΞILD

Project e5214c3a-f03e-4631-8cff-5ab54215fa87 · Stage 2 · build_depth **comprehensive** · version 0.1.0

This is the behavioral contract. ARCHITECTURE.md describes how the system is built, and this document describes what it does. The canonical section numbering is fixed (§1–§10).

---

## 1. Overview & scope

**Problem.** Distributed SOC teams investigate signals across fragmented tools. Context switching, weak evidence, noisy alerts and ungoverned automation raise response time and risk.

**Product.** STΛRSHΞILD is a vendor-neutral cyber-defense control plane. It ingests signed telemetry, correlates it into alerts and incidents, and ranks bounded response proposals with an explainable deterministic engine (QAI). Each proposal is independently validated against versioned policy and authorized by distinct humans, with two approvers for high-impact actions. It is executed with single-use just-in-time action tokens and verified after the fact. Every step is recorded in a tamper-evident hash-chained evidence ledger.

There are two portal experiences over one API:
- **COMMAND** is the executive/analyst overview.
- **TACTICAL** is the SOC operations view: globe, feed, attack path, timeline and posture radar.

**Personas → roles.**

| Persona | Role | Primary screens |
|---|---|---|
| SOC analyst / threat hunter | `analyst` | TACTICAL, Alerts, Incidents |
| Incident commander / responder | `responder` | Incidents, Governance (execute) |
| Response approver | `approver` | Governance (approve/reject) |
| Security engineer / platform admin | `admin` | Sources, Policies, Users, Jobs, all of the above |
| CISO / director | `viewer` | COMMAND, Analytics |
| Auditor / compliance | `auditor` | Ledger, Audit log, exports (read-only) |

A user has exactly one role per org. `admin` holds every permission except approving their own proposals.

**In scope (v1).**
- Signed generic ingest API and a SYNTHETIC connector.
- Normalization and correlation.
- Alerts, incidents and attack-path hypotheses.
- QAI proposals, independent validation, and dual-control approvals.
- JIT action tokens.
- A simulated action adapter (badged SIMULATED, ADR-022).
- Post-action verification and rollback.
- Hash-chained ledger with verify and export.
- COMMAND and TACTICAL portals.
- Analytics, assets, policies (policy-as-code versions), sources, users and RBAC.
- Scheduled jobs and an operations view.
- Audit log with export.
- Settings (password change, sessions, theme).

**Non-Goals (restated from RESEARCH.md; Non-Goals win over scope expansion).**
- Replace all existing SIEM, EDR, firewall, IAM or vulnerability management products in v1.
- Grant an LLM unrestricted privileges or allow it to self-approve response actions. QAI is deterministic and propose-only.
- Automatically isolate business-critical systems, disable privileged identities or change production network policy without explicit authorization and guardrails.
- Offensive intrusion, exploit development, persistence or destructive testing.
- Promise complete threat prevention, perfect attribution, zero false positives or fully autonomous incident closure.
- Claim quantum advantage or real-time predictive certainty.
- Present illustrative globe locations, counts or scores as factual. Synthetic data is always badged SYNTHETIC.
- General-purpose ITSM, CMDB or observability replacement.
- Vendor-specific connectors (CrowdStrike, Okta, Splunk, AWS, …) in v1.
- SSO/SAML and an external secret vault. These are deferred as recorded in DECISIONS. Email delivery is no longer a Non-Goal: F-23 brings it into scope (ADR-071), and WebSocket push is no longer one: F-28 brings live updates into scope (ADR-076).

**Scope expansion (comprehensive)**, each logged as an ADR in DECISIONS.md:
- F-13 Analytics
- F-14 Audit log + export
- F-15 Admin users & roles
- F-16 Jobs operations view
- F-17 Assets inventory
- F-18 Data export (ledger / incidents / audit)
- F-19 Settings & sessions

## 2. Data model

PostgreSQL 16 via Prisma 6. Physical names are snake_case (ADR-020).

The following conventions apply to every table unless a table says otherwise:
- `id UUID PK default gen_random_uuid()`.
- `created_at timestamptz default now()`.
- `org_id UUID NOT NULL FK orgs` on every **tenant** table.
- Every tenant query filters by `org_id = actor.orgId`.

| Table | Key fields (type, constraint) | Ownership |
|---|---|---|
| `orgs` | `name text 2..80`, `slug text unique`, `retention_days int default 400 CHECK 30..3650`, `ledger_status text default 'ok' CHECK in ('ok','broken')`, `demo bool default false`, F-27 `copilot_enabled bool default false` (admin-set, ledgered) | tenant root |
| `users` | `email citext unique` (lower-cased), `name text 1..80`, `password_hash text` (argon2id), `must_change_password bool default false`, `active bool default true`, `last_login_at` | global identity; PII: email, name |
| `memberships` | `org_id`, `user_id`, `role text CHECK in (admin,responder,approver,analyst,auditor,viewer)`; **unique(org_id,user_id)** | tenant |
| `sessions` | `token_hash text unique` (SHA-256 of a 32-byte random token), `user_id`, `org_id`, `csrf_token text`, `ip inet`, `user_agent text`, `expires_at` (absolute 12 h), `last_seen_at` (idle 30 min), `revoked_at` | user |
| `sources` | `org_id`, `name text 2..60`, `kind text default 'generic'` CHECK (`generic`/`firewall`/`edr`/`identity`/`synthetic`) (F-20), `secret_enc text` (AES-256-GCM with `SOURCE_SECRET_KEY`), `enabled bool`, `last_seen_at`, `silence_after_min int default 60 CHECK 5..1440`, `health text default 'unknown'` CHECK (`unknown`/`healthy`/`degraded`/`silent`), `health_changed_at NULL`; unique(org_id,name) | tenant |
| `ingest_nonces` | `source_id`, `nonce text 16..64`, `received_at`; **unique(source_id, nonce)**; purged after 24 h by the retention sweep | tenant (via source) |
| `ingest_batches` | `org_id`, `source_id`, `status text CHECK in (queued,processed,failed)`, `job_id text`, `count int`, `accepted int`, `duplicates int` (F-20), `rejected int` (invalid events only; duplicates are not rejections, ADR-068), `errors jsonb default '[]'` (`[{index, reason}]`), `test bool default false` (F-20 test event), `error text`, `processed_at`; index (source_id, created_at desc) | tenant |
| `events` | `org_id`, `source_id`, `batch_id`, `external_id text`, `observed_at timestamptz`, `category text` (CSF-mapped: `identity, endpoint, network, cloud, application, vulnerability`), `severity text CHECK (critical,high,medium,low,info)`, `title text ≤200`, `summary text ≤2000`, `asset_id`, `lat numeric(5,2) NULL CHECK (lat BETWEEN -90 AND 90)`, `lng numeric(6,2) NULL CHECK (lng BETWEEN -180 AND 180)`, `country char(2) NULL`, `confidence int 0..100`, `synthetic bool`, `raw_hash text` (SHA-256 of the raw event), F-21 network fields `src_ip inet`, `dst_ip inet`, `domain text ≤253`, `url text ≤2048`, `file_hash text`, `user_name text`, enrichment `asn int`, `as_org text`, `geo_source text NULL CHECK (sender, dataset)`, `indicator_ids uuid[] default '{}'`; index (org_id, src_ip), F-24 indexes (org_id, dst_ip), (org_id, domain), (org_id, file_hash), (org_id, user_name); unique(org_id, source_id, external_id); index (org_id, observed_at desc) | tenant |
| `assets` | `org_id`, `external_id text`, `kind text CHECK (host,identity,service,network,cloud)`, `name text`, `criticality int 1..5 default 3`, `business_critical bool default false`, `owner text NULL`, `last_seen_at`, F-25 `software text[] default '{}'` (normalized `vendor:product` keys, ≤ 50), `internet_facing bool default false`; unique(org_id, external_id) | tenant |
| `alerts` | `org_id`, `rule text` (correlation rule id), `severity`, `title`, `status text CHECK (new,linked,suppressed)`, `suppress_reason text NULL CHECK (false_positive,benign,duplicate)`, `incident_id NULL`, `asset_id NULL`, `event_count int`, `first_seen_at`, `last_seen_at`, `synthetic bool`, `dedupe_key text`, F-22 `risk_score int 0..100 default 0`, `mitre_technique text NULL`, `rule_id uuid NULL` (FK rules, ON DELETE SET NULL), `rule_version int NULL` CHECK (`rule = 'R-CUSTOM'` ⇔ `rule_version` set), F-23 `escalation_level int 0..10 default 0`, `escalated_at NULL`; index (org_id, rule_id); unique(org_id, dedupe_key) | tenant |
| `alert_events` | `alert_id`, `event_id`; pk(alert_id,event_id) | tenant (via alert) |
| `incidents` | `org_id`, `number int` (per-org sequence), `title 3..160`, `severity`, `status text CHECK (open,triaged,contained,eradicated,recovered,closed)`, `csf_function text` (derived from status), `owner_id NULL`, `sla_due_at`, `escalated_at NULL`, `closed_at NULL`, `resolution text NULL`; unique(org_id, number) | tenant |
| `attack_path_nodes` / `attack_path_edges` | `incident_id`, node `{key, kind (asset/identity/technique/external), label, ref_id}`; edge `{from_key, to_key, technique text (ATT&CK id), confidence}` | tenant |
| `policies` | `org_id`, `version int`, `rules jsonb` (validated by Zod `PolicyRules`), `note text`, `published_by`; **unique(org_id, version)** | tenant |
| `proposals` | `org_id`, `incident_id`, `proposer_id`, `action text` (catalog key), `target_asset_id`, `impact text CHECK (low,high)`, `score numeric(6,3)`, `score_breakdown jsonb`, `rationale text`, `status text CHECK (proposed,validated,validation_failed,approved,rejected,executed,execution_failed,verified,rolled_back,expired)`, `required_approvals int` (1 or 2), `policy_version int`, `expires_at`, `verify_after NULL`, `claimed_at NULL` | tenant |
| `validations` | `proposal_id unique`, `verdict text (pass,fail)`, `checks jsonb [{key, ok, detail}]`, `policy_version`, `validated_at` | tenant |
| `approvals` | `proposal_id`, `approver_id`, `decision text (approve,reject)`, `note text ≤500`; **unique(proposal_id, approver_id)** | tenant |
| `action_tokens` | `jti text unique`, `proposal_id`, `org_id`, `action`, `target_asset_id`, `expires_at` (≤ 15 min), `used_at NULL`, `revoked_at NULL` | tenant |
| `action_executions` | `proposal_id unique`, `executor_id`, `adapter text default 'simulated'`, `request jsonb`, `result jsonb`, `rollback jsonb`, `executed_at`, `verified_at NULL`, `verification jsonb NULL`, `claimed_at NULL` | tenant |
| `evidence_entries` | `org_id`, `seq bigint`, `type text` (e.g. `incident.transition`, `proposal.approved`), `actor_id NULL` (null = system), `subject_type`, `subject_id`, `payload jsonb`, `prev_hash char(64)`, `hash char(64)`; **unique(org_id, seq)**; **trigger `evidence_entries_immutable` BEFORE UPDATE OR DELETE raises** | tenant, append-only |
| `audit_logs` | `org_id NULL`, `actor_id NULL`, `action text` (e.g. `auth.login`, `auth.login_failed`, `user.role_changed`, `export.ledger`, `authz.denied`), `target text`, `ip`, `meta jsonb` (redacted), `request_id` | tenant |
| `job_runs` | `job_key text`, `window_start timestamptz`, `org_id NULL`, `status text (running,completed,errored)`, `processed int`, `skipped int`, `failed int`, `error text NULL`, `started_at`, `finished_at`; **unique(job_key, window_start)** | system |
| `posture_snapshots` | `org_id`, `taken_at`, `threat_level int 0..100`, `kpis jsonb`, `radar jsonb` (`identify, protect, detect, respond, recover, govern` 0..100), F-25 `score smallint NULL 0..100`, `grade text NULL CHECK (A,B,C,D,F)`, `factors jsonb NULL` (NULL on snapshots taken before F-25); unique(org_id, taken_at) | tenant |
| `notifications` | `org_id`, `kind text` (`sla_escalation`, `verification_failed`, `ledger_broken`, `source_health`, `alert_escalation`), `subject_id`, `message`, `read_at NULL`; unique(org_id, kind, subject_id) | tenant |
| `indicators` (F-21) | `org_id`, `type text CHECK (ip, domain, url, hash, email)`, `value text 1..2048` (normalized), `threat_type text 1..60 default 'unknown'`, `confidence int 0..100 default 50`, `source text 1..80 default 'manual'` (`manual`/`csv`), `tags text[]`, `expires_at NULL`, `hits int`, `last_hit_at NULL`, `created_by NULL`, `updated_at`; unique(org_id, type, value) | tenant |
| `rules` (F-22) | `org_id`, `name text 3..120` (unique per org on lower(name) among live rules), `enabled bool default false`, `current_version int ≥ 1`, `created_by NULL`, `updated_at`, `deleted_at NULL` (soft delete; deleted ⇒ disabled) | tenant |
| `rule_versions` (F-22) | `org_id`, `rule_id`, `version int ≥ 1`, `definition jsonb` (the rule DSL), `severity`, `confidence int 0..100 default 70`, `mitre_technique text NULL` (`T####[.###]`), `required_evidence int 1..100 default 1`, `suppression_min int 0..1440 default 30`, `change_note text ≤ 500 NULL`, `created_by NULL`; unique(rule_id, version); **immutable** (an UPDATE trigger rejects every change) | tenant |
| `notify_policies` (F-23) | `org_id`, `name text 2..80`, `enabled bool default true`, `channel text CHECK (email, webhook)`, `min_severity text CHECK (low, medium, high, critical)`, `recipients citext[] ≤ 20` (email), `role_recipients text[]` (email; active members with those roles), `endpoint_id NULL` (FK delivery_endpoints, ON DELETE RESTRICT; required for webhook, forbidden for email), `escalate_after_min int 5..10080 NULL`, `escalate_to citext[] ≤ 20`, `created_by NULL`, `updated_at` | tenant; PII: recipient emails |
| `delivery_endpoints` (F-23) | `org_id`, `name text 2..80`, `url text ≤ 2048` (https; validated as a public address at save and again before every send), `enabled bool default true`, `secret_enc text` (AES-256-GCM, AAD `delivery-endpoint:<id>`), `secret_hint text` (last 4), `created_by NULL`, `updated_at` | tenant; SECRET: signing secret (shown once) |
| `outbound_messages` (F-23) | `org_id`, `policy_id NULL`, `alert_id NULL`, `endpoint_id NULL` (all ON DELETE SET NULL), `channel`, `kind text CHECK (alert.created, alert.escalated, report.ready)` (F-26: `report.ready` is email only), F-26 `report_id NULL` (ON DELETE SET NULL), `business_ref text`, `target text`, `target_key text`, `payload jsonb` (allowlisted fields only), `signature text NULL` + `signed_at NULL` (required for webhook), `status text CHECK (pending, sent, delivered, failed, dlq, blocked) default pending`, `attempts int 0..20`, `next_attempt_at`, `claimed_at NULL`, `completed_at NULL`, `error text NULL`, `last_status_code int NULL`, `checks jsonb` (validator results); unique(org_id, channel, target_key, kind, business_ref); index (status, next_attempt_at); index (org_id, created_at desc, id) | tenant; PII: target email |
| `vulnerabilities` (F-25) | `org_id`, `cve_id text CHECK ^CVE-\d{4}-\d{4,}$`, `vendor`, `product`, `vendor_key`, `product_key` (normalized for matching), `name`, `description`, `required_action`, `date_added date`, `due_date date NULL`, `known_ransomware bool`, `catalog_version text`, `imported_at`; unique(org_id, cve_id); index (org_id, vendor_key, product_key), (org_id, date_added desc). The CISA KEV catalogue, imported per org (ADR-073) | tenant |
| `vuln_findings` (F-25) | `org_id`, `asset_id`, `vulnerability_id` (both ON DELETE CASCADE), `status text CHECK (open, resolved)`, `first_seen_at`, `last_seen_at`, `resolved_at NULL`; unique(asset_id, vulnerability_id); index (org_id, status). Re-matched on import and on every software change; a removed match resolves, a returning one reopens the same row | tenant |
| `report_schedules` (F-26) | `org_id`, `name text 3..80`, `cadence text CHECK (daily, weekly)`, `hour_utc int 0..23`, `weekday int 0..6 NULL` (required for weekly, NULL for daily; CHECK), `recipients text[]` (1..20, active members at save and at run), `enabled bool default true`, `next_run_at`, `last_run_at NULL`, `created_by NULL`, `updated_at`; index (enabled, next_run_at). At most 25 per org (ADR-074) | tenant; PII: recipient addresses |
| `reports` (F-26) | `org_id`, `schedule_id NULL` (ON DELETE SET NULL), `title text`, `range_start`, `range_end` (CHECK end > start, at most 90 days), `snapshot jsonb` (versioned totals, trend, top lists, posture; computed once, never recomputed), `created_by NULL` (NULL = the schedule); unique(schedule_id, range_start) so a slot is generated at most once | tenant |
| `alert_copilot` (F-27) | `org_id`, `alert_id` (ON DELETE CASCADE), `model text`, `output jsonb` (the validated `CopilotOutput`; CHECK object), `context_sha256 char(64)` (SHA-256 of exactly the context that left), `input_tokens int ≥ 0`, `output_tokens int ≥ 0`, `requested_by NULL`; index (org_id, alert_id, created_at desc). Append-only; the latest row is shown (ADR-075) | tenant |
| `ip_networks` (F-21) | `id bigserial`, `start_ip inet`, `end_ip inet` CHECK same family and start ≤ end, `country char(2) NULL`, `asn int NULL`, `as_org text NULL`, `lat numeric(5,2) NULL`, `lng numeric(6,2) NULL`, `dataset text` (licence tag); index (start_ip). Ships **empty**; loaded only by the owner with `scripts/ipdata-import.ts` from a dataset whose licence they accept (ADR-068). Never seeded with illustrative data. | global reference (no tenant data) |

**State machines** (shared table `packages/shared/src/states.ts`; every transition has a named writer, ADR-021):

| Machine | Transition | Pre | Post |
|---|---|---|---|
| Incident | → `open` | create, ≥ 0 alerts | `sla_due_at = now + SLA(severity)` with critical 1 h, high 4 h, medium 24 h, low 72 h, info 168 h |
| | `open → triaged` | owner set | csf `detect` |
| | `triaged → contained` | ≥ 1 executed or verified proposal **or** a manual containment note | csf `respond` |
| | `contained → eradicated` | note required | csf `respond` |
| | `eradicated → recovered` | no proposal in `executed` awaiting verification | csf `recover` |
| | `recovered → closed` | resolution text required | `closed_at` |
| | any → same, or a skipped state | rejected | 409 `INVALID_TRANSITION` |
| Proposal | → `proposed` | incident not closed; action in catalog; target asset in org | `expires_at = now + 24 h`; validation job enqueued |
| | `proposed → validated / validation_failed` | worker validator only | validation row |
| | `validated → approved` | `approve` count ≥ required, distinct approvers, approver ≠ proposer | — |
| | `validated → rejected` | any approver rejects | terminal |
| | `approved → executed / execution_failed` | responder; token minted and verified; adapter result | `verify_after = now + 2 min` |
| | `executed → verified / rolled_back` | verification job only | rollback executed through the adapter when failing |
| | `validated / approved → expired` | `now > expires_at` (expiry job) | tokens revoked |
| Alert | → `new` | correlation | — |
| | `new → linked` | incident in org | `incident_id` set |
| | `new → suppressed` | reason in enum | counts toward FP rate when `false_positive` |
| Ingest batch | `queued → processed / failed` | worker / watchdog | — |

## 3. API surface

Base path `/api/v1`. All routes take and return JSON. Auth is the session cookie `ss_session` plus `X-CSRF-Token` on every non-GET request. Lists are cursor-paginated (`?cursor=&limit=`, limit 1..100 default 25) and return `{ data, nextCursor }`. Sorting uses allow-listed `sort=field:asc|desc`, and unknown fields return 400. Every non-2xx response is §7. Paths below are written relative to `/api/v1`.

| Endpoint | Auth (permission) | Input | Output | Internal |
|---|---|---|---|---|
| `POST /auth/register` | public, limiter 10/h/IP | `{orgName 2..80, name 1..80, email, password 12..128}` | 201 `{user, org, role:'admin'}` + session cookie | no |
| `POST /auth/login` | public, limiter 5/15 min per (ip,email) + 20/15 min per IP | `{email, password}` | 200 `{user, org, role, mustChangePassword}` + cookie; 401 `INVALID_CREDENTIALS` (uniform) | no |
| `POST /auth/logout` | session | — | 204, cookie cleared | no |
| `GET /auth/session` | session optional | — | 200 `{user, org, role, permissions[], csrfToken}` or 401 | no |
| `PATCH /auth/profile` | session | `{name 1..80}` (strict: email/role rejected) | 200 `{user}` | no |
| `POST /auth/password` | session | `{currentPassword, newPassword 12..128}` | 204; other sessions revoked | no |
| `GET /auth/sessions` | session | — | `{data:[{id, ip, userAgent, lastSeenAt, current}]}` | no |
| `DELETE /auth/sessions` | session | — | 204, revokes all but current | no |
| `GET /overview` | `overview:read` | — | `{posture: snapshot, kpis:{openIncidents, criticalAlerts24h, mttrHours, pendingApprovals, verifiedRate, ledgerStatus}, recommendations:[top 5 proposals], governance:{byStatus}, synthetic: bool}` (Redis 15 s cache, ETag) | no |
| `GET /events` | `events:read` | `severity[]`, `category[]`, `since` (ISO, ≤ 90 d), `synthetic` (true/false/all), `q ≤ 100`, F-24 exact pivots `ip` (src or dst), `srcIp`, `dstIp`, `domain`, `userName`, `fileHash`, `assetId`, `sourceId`, cursor, limit | page of events | no |
| `GET /events/geo` | `events:read`; limiter 60/min/user | `since` (default 24 h), `severity[]` | `{data:[{lat, lng, country, count, maxSeverity, synthetic}]}` (2-dp cells) | no |
| `GET /threat-map` | `events:read`; limiter 60/min/user | `since` (default 24 h, clamped to 30 d), `severity[]` | `{since, links:[{srcIp, srcZone, country, lat, lng, target:{kind: asset\|ip, id, ip, label, zone}, events, maxSeverity, ports (top 5), vectors[]}] (top 300, busiest first), vectors:[{vector, links, sources, events, maxSeverity}], totals:{events, sources, targets, links}, truncated}`; org-scoped, `Cache-Control: no-store` (F-29) | no |
| `GET /events/:id` | `events:read` | — | `{...event, batchId, rawHash, createdAt, indicators[] (intel:read only), alerts:[{id, title, severity, status, rule}]}`; 404 for another org (F-24) | no |
| `GET /search` | `events:read`; limiter 60/min/user | `q 2..200`, `type` (auto, ip, domain, url, hash, email, user, id, text; default auto) | `{q, kind, value, groups:[{type: events/alerts/assets/indicators/rules, count ≤ 100, more, items[] ≤ 5}]}`; groups the role cannot read are omitted; events within 90 d (F-24) | no |
| `GET /threat-profile` | `events:read`; limiter 60/min/user | `kind` (ip, domain), `value` (normalized; 400 when invalid) | `{kind, value, firstSeenAt, lastSeenAt, windowDays, eventCount, network:{asn, asOrg, country, geoSource} or null, indicators[] (intel:read only), timeline:{days[] (last 30 d), series:{severity: counts[]}}, assets[] ≤ 10, detections[], alerts[] ≤ 20}`; counts, assets, detections and alerts over 90 d (F-24) | no |
| `GET /timeline` | `events:read` | `since` default 24 h, `limit ≤ 100` | ledger-derived response timeline `{data:[{at, type, label, subject}]}` | no |
| `GET /alerts` | `alerts:read` | `status[]`, `severity[]`, `q`, `sort` (lastSeenAt, severity), cursor, limit | page | no |
| `GET /alerts/:id` | `alerts:read` | — | alert + events (≤ 50) + asset | no |
| `POST /alerts/:id/link` | `alerts:write` | `{incidentId}` or `{newIncident:{title, severity}}` | 200 alert (`linked`) | no |
| `POST /alerts/:id/suppress` | `alerts:write` | `{reason: false_positive/benign/duplicate, note ≤ 500}` | 200 alert (`suppressed`) | no |
| `GET /incidents` | `incidents:read` | `status[]`, `severity[]`, `q`, `sort` (createdAt, severity, slaDueAt), cursor, limit | page | no |
| `POST /incidents` | `incidents:write` | `{title, severity, alertIds[] ≤ 50}` | 201 incident | no |
| `GET /incidents/export` | `export:run`, limiter 5/h/user | `format=csv/json`, `status[]` | file download (`Content-Disposition: attachment`) | no |
| `GET /incidents/:id` | `incidents:read` | — | incident + alerts + recent events + proposals + allowed next transitions | no |
| `POST /incidents/:id/transition` | `incidents:write` | `{to, note ≤ 1000, ownerId?, resolution?}` | 200 incident; 409 `INVALID_TRANSITION`, 422 `PRECONDITION_FAILED` | no |
| `GET /incidents/:id/attack-path` | `incidents:read` | — | `{nodes[], edges[]}` | no |
| `POST /incidents/:id/candidates` | `proposals:write` | `{objective: contain/investigate/harden}` | QAI ranked candidates `{data:[{action, target, impact, score, breakdown, rationale}]}` (not persisted) | no |
| `GET /proposals` | `proposals:read` | `status[]`, `impact`, `incidentId`, cursor, limit | page | no |
| `POST /proposals` | `proposals:write` | `{incidentId, action, targetAssetId, rationale ≤ 1000}` | 201 proposal (`proposed`); validation enqueued | no |
| `GET /proposals/:id` | `proposals:read` | — | proposal + validation + approvals + execution + allowed actions for caller | no |
| `POST /proposals/:id/approve` | `proposals:approve` | `{note ≤ 500}` | 200 proposal; 403 `SELF_APPROVAL` (proposer), 409 `ALREADY_DECIDED` | no |
| `POST /proposals/:id/reject` | `proposals:approve` | `{reason 3..500}` | 200 proposal (`rejected`) | no |
| `POST /proposals/:id/execute` | `proposals:execute` | `{confirm: true}` | 200 proposal (`executed` or `execution_failed`) | no |
| `GET /ledger` | `ledger:read` | `type`, `subjectId`, cursor, limit | page of entries incl. hashes | no |
| `GET /ledger/verify` | `ledger:read` | — | `{ok, checked, brokenAt: seq or null, headHash}` | no |
| `GET /ledger/export` | `export:run`, limiter 5/h/user | `format=json/csv` | NDJSON or CSV download incl. hashes | no |
| `GET /assets` | `assets:read` | `kind`, `criticality`, `q`, cursor, limit | page | no |
| `POST /assets` | `assets:write` | `{externalId 1..200, kind, name 1..200, criticality 1..5 = 3, businessCritical = false, owner ≤ 80}` | 201 asset; 409 on duplicate (org, externalId) | no |
| `GET /assets/:id` | `assets:read` | — | `{...asset, software[], internetFacing, findings:[open finding]}`; 404 for another org (F-25) | no |
| `PATCH /assets/:id` | `assets:write` | `{criticality 1..5, businessCritical, owner ≤ 80, F-25 software[] ≤ 50 (`vendor:product`, normalized; 400 `invalid_software_key`), internetFacing}` | 200 asset + `findings:{opened, resolved}` when software changed; ledger `asset.exposure_changed` on a software or exposure change | no |
| `GET /vulnerabilities` | `assets:read` | `q ≤ 100`, `ransomware`, `affected` (true/false), cursor, limit | `{data:[{id, cveId, vendor, product, name, dateAdded, dueDate, knownRansomware, requiredAction, affectedAssets}], nextCursor, catalog:{version, total, importedAt}}` (F-25) | no |
| `GET /vulnerabilities/findings` | `assets:read` | `status` (open, resolved), `assetId` (404 for another org), cursor, limit | `{data:[{id, status, firstSeenAt, lastSeenAt, resolvedAt, asset:{id, externalId, name, internetFacing}, vulnerability}], nextCursor, counts:{open, resolved}}` (F-25) | no |
| `POST /vulnerabilities/import` | `vulns:import` (admin); 5/h per org | strict `{source:'bundled'}` or strict `{json ≤ 4 MB}` (CISA KEV feed; 400 `kev_invalid`); the larger body parser (9 MB, room for JSON string escaping) runs only for an authenticated actor holding `vulns:import`, after the 5/h limit | 200 `{catalogVersion, created, updated, removed, total, opened, resolved}`; idempotent; audit + ledger `vuln_catalog.imported` (F-25) | no |
| `GET /posture` | `overview:read` | — | live `{...posture, grade, factors:[{key, label, penalty, max, detail}], trend:[{day, score, grade}] (30 d, one per UTC day)}`; `no-store` (F-25) | no |
| `GET /reports` | `reports:read` | `scheduleId`, cursor, limit 1..100 (25) | `{data:[{id, title, rangeStart, rangeEnd, scheduleId, scheduleName, createdBy, createdAt, totals:{events, alerts, criticalHigh}}], nextCursor}` newest first; 400 `invalid_cursor` (F-26) | no |
| `POST /reports` | `reports:write` (admin, auditor); 30/h per org | strict `{title 3..80, rangeStart, rangeEnd}` (ISO with offset; end > start, at most 90 days; 400 `range_future` if the start is in the future) | 201 the report with its snapshot; audit `report.create`, ledger `report.created` (F-26) | no |
| `GET /reports/:id` | `reports:read` | none | `{id, title, rangeStart, rangeEnd, schedule, createdBy, createdAt, snapshot}`; `no-store`; 404 for another org (F-26) | no |
| `POST /reports/:id/send` | `reports:write`; 20/h per org | strict `{recipients 1..20}` (each an active org member, else 400 `not_org_member`) | 202 `{queued}`: one `report.ready` email per recipient through the outbound validator; audit `report.send`, ledger `report.sent` (F-26) | no |
| `GET /reports/recipients` | `reports:write` | none | `{data:[{email, name, role}]}` active members (F-26) | no |
| `GET /reports/schedules` | `reports:read` | none | `{data:[schedule + description]}` (F-26) | no |
| `POST /reports/schedules` | `reports:write` | strict `{name, cadence, hourUtc, weekday (weekly only), recipients 1..20, enabled = true}` | 201 schedule with `nextRunAt`; 400 field errors or `not_org_member`; 422 `PRECONDITION_FAILED` past 25 per org; ledger `report_schedule.created` (F-26) | no |
| `PATCH /reports/schedules/:id` | `reports:write` | partial of the above (at least 1 field); the merged schedule is re-checked | 200 schedule (next run recomputed when timing changes or it is re-enabled); 409 `CONFLICT` on a concurrent edit; ledger `report_schedule.updated` (F-26) | no |
| `DELETE /reports/schedules/:id` | `reports:write` | none | 204; its reports stay (schedule_id set NULL); ledger `report_schedule.deleted` (F-26) | no |
| `GET /analytics` | `analytics:read` | `range=7d/30d/90d` | `{alertsBySeverity[], incidentsOpened[], mttrHours[], fpRate, suppressedByReason, proposalsByOutcome, detection:{precision, fpRate, ci95}}` | no |
| `GET /policies` | `policies:read` | — | `{data: versions[], current}` | no |
| `POST /policies` | `policies:write` (admin) | `{rules: PolicyRules, note 3..200}` | 201 new version (n+1) | no |
| `GET /sources` | `sources:read` | — | `{data: sources[]}` (no secrets) | no |
| `POST /sources` | `sources:write` | `{name, kind = generic (generic/firewall/edr/identity), silenceAfterMin 5..1440 = 60}` | 201 `{source, secret}`, secret shown once | no |
| `GET /sources/:id` | `sources:read` | — | source + `stats24h {batches, events, accepted, duplicates, rejected, failedBatches}` (test batches excluded) + `recentErrors[] ≤ 50` (F-20) | no |
| `GET /sources/:id/batches` | `sources:read` | cursor, limit | page of batches `{status, count, accepted, duplicates, rejected, errors[], error, test, createdAt, processedAt}` newest first (F-20) | no |
| `POST /sources/:id/test` | `sources:write` | — | 202 `{batchId}`; one `test=true` batch through the normal pipeline; 409 if disabled (F-20) | no |
| `POST /sources/:id/rotate` | `sources:write` | — | 200 `{secret}` | no |
| `PATCH /sources/:id` | `sources:write` | strict `{enabled?, kind?, silenceAfterMin?}`, at least one; a synthetic source cannot change kind (409) | 200 source | no |
| `GET /indicators` | `intel:read` | `type`, `q ≤ 200`, `page`, `pageSize ≤ 100` | `{data, total, page, pageSize}` (F-21) | no |
| `POST /indicators` | `intel:write` | `{type, value, threatType 1..60?, confidence 0..100?, tags[]?, expiresAt?}` | 201 created or 200 updated (upsert on (org,type,normalized value)); 400 `value` invalid_format (F-21) | no |
| `POST /indicators/import` | `intel:write` | `text/csv` ≤ 1 MB, ≤ 5000 rows: `type,value[,threat_type[,confidence]]` | 200 `{created, updated, rejected, errors[] ≤ 50}` (F-21) | no |
| `DELETE /indicators/:id` | `intel:write` | — | 204 (F-21) | no |
| `GET /rules` | `rules:read` | — | `{builtin[], custom[]}`, each with 30-day `stats {alerts, daily[30], falsePositiveRate, meanTimeToTriageSec, mttrSec, lastFiredAt}` (F-22) | no |
| `GET /rules/:id` | `rules:read` | — | rule + current version + `versions[]` newest first + stats; 404 deleted or other org (F-22) | no |
| `POST /rules` | `rules:write` | strict `{name, enabled?=false, definition, severity, confidence?, mitreTechnique?, requiredEvidence?, suppressionMin?}` | 201 rule at v1; 409 `CONFLICT` live name taken (F-22) | no |
| `POST /rules/:id/versions` | `rules:write` | strict version fields + `changeNote?` | 201 rule at the new current version (F-22) | no |
| `PATCH /rules/:id` | `rules:write` | strict `{enabled?, name?}`, at least one | 200 rule (F-22) | no |
| `DELETE /rules/:id` | `rules:write` | — | 204 soft delete; alerts keep their rule version (F-22) | no |
| `GET /notification-policies` | `notify:read` | — | `{data:[policy]}` (F-23) | no |
| `POST /notification-policies` | `notify:write` (admin) | strict `{name 3..80, enabled?, channel, minSeverity, recipients?, roleRecipients?, endpointId?, escalateAfterMin? 5..1440, escalateTo?}`; email needs ≥ 1 recipient or role, webhook needs `endpointId` | 201 policy; addresses lower-cased (F-23) | no |
| `PATCH /notification-policies/:id` | `notify:write` | the same fields, at least one; the result must still satisfy the channel rule | 200 policy (F-23) | no |
| `DELETE /notification-policies/:id` | `notify:write` | — | 204; queued messages keep `policy_id = NULL` and are blocked (F-23) | no |
| `GET /delivery-endpoints` | `notify:read` | — | `{data:[{id, name, url, enabled, secretHint, inUse}]}`; never the secret (F-23) | no |
| `POST /delivery-endpoints` | `notify:write` | strict `{name 3..80, url ≤ 2048}`; https only, no credentials, no private/reserved address | 201 endpoint + `secret` (shown once); 400 unsafe URL (F-23) | no |
| `PATCH /delivery-endpoints/:id` | `notify:write` | strict `{name?, enabled?}`, at least one (the URL is fixed; add a new endpoint to change it) | 200 endpoint (F-23) | no |
| `POST /delivery-endpoints/:id/rotate` | `notify:write` | — | 200 endpoint + new `secret` (shown once) (F-23) | no |
| `DELETE /delivery-endpoints/:id` | `notify:write` | — | 204; 409 `CONFLICT` while a policy uses it (F-23) | no |
| `GET /notifications/log` | `notify:read` | query `channel?`, `status?`, `cursor?` | `{data:[{id, channel, target, kind, alertId, policyName, status, attempts, error, lastStatusCode, checks[], createdAt, nextAttemptAt, completedAt}], nextCursor}`, newest first, 50 per page (F-23) | no |
| `POST /notifications/log/:id/retry` | `notify:write` | — | 200 entry back to `pending`, attempts 0; 409 `INVALID_TRANSITION` unless failed/dlq/blocked (F-23) | no |
| `POST /alerts/:id/escalate` | `alerts:write` | — | 200 `{alertId, level, queued}`; 409 `INVALID_TRANSITION` unless the alert is `new` (F-23) | no |
| `GET /alerts/:id/notifications` | `notify:read` | — | `{data:[entry]}` for this alert (F-23) | no |
| `GET /alerts/:id/copilot` | `alerts:read` | none | `{enabled, configured, suggestion: {id, model, createdAt, requestedBy, output} \| null}` (latest); `no-store`; 404 for another org (F-27) | no |
| `POST /alerts/:id/copilot` | `alerts:write`; 20/h per org | none | 201 the suggestion: the bounded, redacted alert context passes the copilot egress validator, goes once to STARGAZER's paid no-data-collection route, the strict output is re-validated, stored and ledgered `alert.copilot_suggested` (with `contextSha256`), audit `alert.copilot`; 409 `COPILOT_DISABLED` (org off); 422 `PRECONDITION_FAILED` (egress check) or `COPILOT_DECLINED` (nothing stored); 503 `COPILOT_UNAVAILABLE` (not configured or unreachable) (F-27) | no |
| `GET /org/settings` | `org:manage` (admin) | none | `{copilotEnabled, copilotConfigured}` (F-27) | no |
| `PATCH /org/settings` | `org:manage` (admin) | strict `{copilotEnabled: boolean}` | 200 settings; a change is ledgered `org.copilot_toggled` and audited `org.copilot`; a no-op records nothing (F-27) | no |
| `GET /live` (WebSocket upgrade) | any signed-in member | inbound only `{"type":"ping"}`, ≤ 20/10 s, ≤ 1 KB | 101 then change-hint frames `{type, kind, id?, at}` for the actor's org only, no record data; 403 bad Origin (checked first), 401 no session, 428 password change owed; close 4401 when the session ends, 1008 on a bad or excess frame (F-28) | no |
| `POST /rules/dry-run` | `rules:read` | `{definition, requiredEvidence?, hours 1..168 = 24}`; 30/min per user | 200 `{hours, scanned ≤ 5000, matched, truncated, groups[] ≤ 20, sample[] ≤ 20}`; writes nothing (F-22) | no |
| `GET /users` | `users:read` | — | `{data:[{id, name, email, role, active, lastLoginAt}]}` | no |
| `POST /users` | `users:write` | `{name, email, role}` | 201 `{user, temporaryPassword}` (shown once; must change on first login) | no |
| `PATCH /users/:id` | `users:write` | `{role?, active?}`; cannot demote or deactivate self or the last admin | 200 | no |
| `GET /jobs` | `jobs:read` | — | `{data:[{key, schedule, lastRun, nextRunAt, totals24h}]}` | no |
| `GET /jobs/runs` | `jobs:read` | `key`, cursor, limit | page of job runs | no |
| `GET /audit` | `audit:read` | `action`, `actorId`, cursor, limit | page | no |
| `GET /audit/export` | `export:run`, limiter 5/h/user | `format=csv/json` | download | no |
| `GET /notifications` | session | — | `{data, unread}` | no |
| `POST /notifications/read` | session | `{ids[]}` | 204 | no |

Machine and ops endpoints, which no screen binds:

| Endpoint | Auth | Purpose | Internal |
|---|---|---|---|
| `POST /ingest/events` | HMAC source signature (§6 ARCHITECTURE) | machine-to-machine signed ingest; 202 `{batchId, accepted}` | yes — internal machine API |
| `GET /api/health` | none | readiness incl. db/cache/queue/worker | yes — internal probe |
| `GET /api/health/live` | none | liveness | yes — internal probe |
| `GET /metrics` | network-restricted (not routed by Caddy) | Prometheus | yes — internal ops |
| `GET /api/openapi.json` | none | OpenAPI 3.1 | yes — internal contract |
| `POST /admin/jobs/:key/run` | admin session; mounted only when `NODE_ENV!=production && ENABLE_TEST_TRIGGERS=1` | force a scheduled job for smoke/e2e | yes — internal test trigger |

**Performance per endpoint:** see §9.

## 4. Security requirements

**RBAC matrix** (`packages/shared/src/rbac.ts`):

| Permission | admin | responder | approver | analyst | auditor | viewer |
|---|---|---|---|---|---|---|
| overview:read, analytics:read | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| events:read, alerts:read, incidents:read, proposals:read, assets:read, reports:read (F-26) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| alerts:write, incidents:write | ✓ | ✓ | — | ✓ | — | — |
| proposals:write | ✓ | ✓ | — | ✓ | — | — |
| proposals:approve | ✓ | — | ✓ | — | — | — |
| proposals:execute | ✓ | ✓ | — | — | — | — |
| assets:write | ✓ | ✓ | — | — | — | — |
| ledger:read, audit:read, export:run, reports:write (F-26) | ✓ | — | — | — | ✓ | — |
| policies:read, jobs:read, sources:read, intel:read, rules:read, notify:read | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| intel:write (F-21) | ✓ | ✓ | — | ✓ | — | — |
| policies:write, sources:write, users:read, users:write, rules:write (F-22), notify:write (F-23), vulns:import (F-25), org:manage (F-27) | ✓ | — | — | — | — | — |

A denied permission returns 403 `FORBIDDEN` and writes an `authz.denied` audit row. A resource from another tenant returns 404 `NOT_FOUND`.

**Threat-derived controls** (ARCHITECTURE §10):

| Ref | Control | Compliance |
|---|---|---|
| S-1 | Ingest HMAC-SHA256 over `ts.nonce.rawBody`, constant-time compare, skew ≤ 300 s, nonce replay cache (Redis `SET NX EX 600` + unique row), source must be enabled. Verification precedes business reads (INV-8). | COMPLIANCE A07/A08 |
| S-2 | Sessions: 256-bit random token; only its SHA-256 is stored. Cookie `ss_session` is `HttpOnly; Secure; SameSite=Lax; Path=/`. Rotated on login and password change. Idle 30 min, absolute 12 h. | ASVS V3 |
| S-3 | CSRF: per-session token returned by `GET /auth/session`, required as `X-CSRF-Token` on non-GET requests, compared constant-time. | ASVS V4 |
| S-4 | Security headers: CSP nonce + strict-dynamic, HSTS (non-localhost), XCTO, Referrer-Policy, Permissions-Policy, COOP, `frame-ancestors 'none'`. Caddy strips `x-middleware-subrequest` (INV-18). | A05 |
| T-1 | Ledger: SHA-256 chain, `prev_hash` of the first entry = 64×"0". Canonical JSON (sorted keys). Appends are serialized per org with `pg_advisory_xact_lock(hashtext(org_id))` inside the business transaction, unique(org,seq), and an immutability trigger. | A08 |
| E-1 | Approval: approver ≠ proposer (403 `SELF_APPROVAL`). Distinct approvers come from the unique constraint. High impact requires 2. Approve and reject are allowed only in `validated`. | A01 |
| E-2 | Action token: HMAC-SHA256 (`ACTION_TOKEN_SECRET`) compact token `{jti, org, prop, act, tgt, exp}`, exp ≤ 15 min. Verified, then consumed with `UPDATE action_tokens SET used_at=now() WHERE jti=$1 AND used_at IS NULL AND revoked_at IS NULL AND expires_at>now()` (1 row or 409 `TOKEN_INVALID`), then dispatched (INV-9). | A01/A04 |
| E-3 | Guardrail: an action on a `business_critical` asset, or a `disable_identity` on an asset with criticality 5, is always `impact: high`. A policy `deny` rule makes validation fail. | Non-Goal 3 |
| I-1 | Tenant isolation: every service function takes `actor` and scopes by `orgId`. Adversarial tests cover every `/:id` route with a second org. | A01 |
| I-2 | Log redaction (pino `redact`): authorization, cookie, set-cookie, x-csrf-token, x-starshield-signature, password, newPassword, currentPassword, token, secret, email. | A09 |
| D-1 | Rate limits (Redis store, so they hold across replicas): global 600/min/IP; per-route limits as in §3; ingest 120 batches/min per source; mutations 120/min per user. 429 carries `Retry-After`. | A04 |
| A-1 | Audit: auth success/fail, logout, password change, role/active change, source create/rotate/disable, policy publish, exports and authz denials are written to `audit_logs`. | A09 |

**Compliance baseline: memorized-secret verifier (ADR-007, NIST 800-63B §5.1.1.2).**
1. **Breached-password screening** on register, password change and admin-created first-login change. The check calls `GET https://api.pwnedpasswords.com/range/<first 5 SHA-1 hex>` with `Add-Padding: true` and a 2 s timeout. A suffix match is rejected with 422 `PASSWORD_BREACHED` ("This password has appeared in a data breach — choose another."). If the service is unreachable, the check fails open with a `warn` log and a `hibp_unavailable_total` metric. This is a documented availability trade-off (ADR-023). `HIBP_MODE=off` exists only for tests and offline builds and is rejected in production.
2. **Length, not composition.** Minimum 12, maximum 128. There are no composition rules, and passwords are never truncated. Unicode is NFKC-normalized before hashing.
3. **Rate-limit before the hash compare.** `loginLimiter` (ip+email, ip) runs before `authService.login` (INV-11). Unknown emails still run a dummy Argon2 verify so response timing is uniform.
4. **No forced rotation and no security questions.** The recovery path in v1 is an admin reset (the admin issues a temporary password, which the user must change). Email-link reset is deferred because there is no `has_email` in v1.

Argon2id parameters are m=19456 KiB, t=2, p=1 (INV-47).

**Domain checklist: `has_geo`.**
1. Coordinate bounds are enforced twice: by the CHECK constraints `lat BETWEEN -90 AND 90` and `lng BETWEEN -180 AND 180` (INV-20), and by Zod `lat: z.number().min(-90).max(90)`, `lng: z.number().min(-180).max(180)` at ingest. An out-of-range event is rejected per-event (counted in `rejected`), not per-batch.
2. Map provider proxy rate limit: there is no external map provider (ADR-016), so there is no billable proxy. The geo aggregation endpoint `GET /events/geo` is still limited to 60/min/user.
3. PII: coordinates are truncated to 2 decimal places (~1.1 km) **at ingest, before storage**, and aggregated into cells for display. Raw precise coordinates are never stored; only the raw-event hash is kept. Synthetic coordinates are badged.

**Domain checklist: `has_scheduled_work`** is in §5 F-12.

## 5. Features & Key Workflows

Each feature lists acceptance criteria (AC). Each `Wn` maps 1:1 to the RESEARCH.md Key Workflows, with numbered steps and a named terminal step.

### F-1 Platform foundation
Monorepo, env schema, logger, error envelope, health, metrics, graceful shutdown, Prisma schema and migrations, seed.

| AC | Criterion |
|---|---|
| F-1.1 | Boot with a missing required env var exits ≠ 0 naming it. |
| F-1.2 | `GET /api/health` returns 200 within 500 ms when db, cache and queue are ok, and 503 if the db is down. Worker heartbeat age > 90 s makes it `degraded`. |
| F-1.3 | SIGTERM drains in-flight requests (≤ 10 s) and exits 0. |
| F-1.4 | Every response has an `x-request-id`. |

### F-2 Authentication & sessions
Register (creates an org with the creator as admin), login, logout, session, password change, sessions list and revoke, and the 800-63B verifier.

| AC | Criterion |
|---|---|
| F-2.1 | Breached password → 422 `PASSWORD_BREACHED`. |
| F-2.2 | 11-character password → 400 `VALIDATION_FAILED` on field `password`. |
| F-2.3 | The 6th failed login in 15 min for the same ip+email → 429. |
| F-2.4 | Unknown-email and wrong-password responses are identical. |
| F-2.5 | A mutation without CSRF → 403 `CSRF_INVALID`. |
| F-2.6 | `mustChangePassword` users can call only session, password and logout until they change it (403 `PASSWORD_CHANGE_REQUIRED`). |

### F-3 App shell, theming & design system
Ported template shell: header (shield logo, wordmark, taglines, status chips, UTC clock), sidebar with 10+ destinations, mobile drawer nav, footer, theme toggle (auto/dark/light, persisted cookie `ss_theme`), skip link, designed 404 and error boundaries.

AC: all §6 screens use the tokens; theme persists without flash; mobile nav reaches every primary destination at 375 px.

### F-4 Signed ingest & sources
Sources CRUD with the secret shown once, HMAC ingest, replay protection, async normalization, ingest watchdog.

AC:
- Bad signature, stale timestamp or a replayed nonce → 401 `SIGNATURE_INVALID` / `TIMESTAMP_SKEW` / `REPLAY`, with nothing persisted.
- A valid batch → 202, and its events are visible within one worker cycle.
- Out-of-range lat → that event is rejected and the others are accepted.
- Coordinates are stored truncated to 2 dp.

### F-5 Synthetic connector
A `synthetic` source per demo org. The `synthetic-feed` job emits 5–20 realistic events per run across categories, severities and geo cells.

AC: events have `synthetic=true`, every UI surface showing them renders a SYNTHETIC badge, and the job is disabled when `SYNTHETIC_FEED` ≠ 1.

### F-6 Correlation & alerts
Correlation rules over normalized events. Each fires an alert deduped by `dedupe_key`:
- `R-BRUTE`: ≥ 5 auth failures for one identity in 10 min.
- `R-MALWARE`: endpoint critical/high.
- `R-EXFIL`: network high + same asset.
- `R-VULN-CRIT`: vulnerability critical on a business-critical asset.
- `R-INTEL` (F-21): an event matches an unexpired tenant indicator. One alert per indicator (deduped 24 h); severity floor from confidence (≥ 80 critical, ≥ 50 high, else medium), raised by the event's own severity. ATT&CK T1071 in the attack path.
- `R-GENERIC`: any critical.
- `R-CUSTOM` (F-22): an enabled tenant rule matches; the alert names the rule id and the version that fired it.

Every alert carries a risk score 0..100 = severity weight (critical 100, high 75, medium 50, low 25, info 10) × confidence ÷ 100 × asset criticality multiplier (business-critical or criticality 5 → 1.0, 4 → 0.9, 3 or no asset → 0.75, else 0.6), and an ATT&CK technique.

Alert list with filters, detail, link to an incident, suppress with a reason.

AC: the same dedupe key within 24 h increments `event_count` instead of creating a duplicate; suppressing with `false_positive` is reflected in the analytics FP rate.

### F-7 Incidents & attack path
Create from alerts, list and filter, detail, six-state lifecycle with pre/post conditions (§2), SLA, attack-path graph derived from linked alerts, events and assets (nodes for assets and identities, edges labelled with ATT&CK technique ids from the rule mapping).

AC:
- Every allowed transition succeeds.
- Every disallowed one → 409.
- Each transition appends a ledger entry.
- The attack path has ≥ 1 node per linked asset.

### F-8 QAI proposals
The candidate generator over the action catalog:

| Action | Impact | Reversible | Rollback |
|---|---|---|---|
| `enrich_context` | low | yes | — |
| `notify_owner` | low | yes | — |
| `tag_asset` | low | yes | untag |
| `block_ip` | high | yes | unblock |
| `isolate_host` | high | yes | release |
| `disable_identity` | high | yes | re-enable |
| `force_password_reset` | high | no | — |
| `quarantine_file` | high | yes | restore |

The score is `0.35·sev + 0.25·crit + 0.25·conf − 0.10·blast − 0.05·irrev`, each term normalized 0..1. The score is persisted with its breakdown and is reproducible: the same inputs give the same score.

AC: candidates are sorted by score desc; each candidate shows all five components; creating a proposal enqueues validation.

### F-9 Independent validation & policy-as-code
The worker-only validator runs these checks and records each one, with its policy version:

| Check | Requirement |
|---|---|
| `evidence` | ≥ 1 linked alert with events; all in org |
| `policy_allows` | action not denied for the asset kind/criticality |
| `blast_radius` | count of assets affected ≤ `policy.maxBlastRadius` |
| `change_window` | now within `policy.changeWindows` (UTC) or the action is low impact |
| `reversibility` | high impact must be reversible unless `policy.allowIrreversible` |
| `approval_requirement` | sets 1 or 2 |
| `business_critical_guard` | business-critical target → high |
| `confidence_calibration` | mean linked-event confidence ≥ `policy.minConfidence` (0..100, default 30) and within 0.25 of the confidence the proposal was scored with (ADR-060) |

Policies are versioned. Publishing creates version n+1 and a ledger entry, and the current version is used for new validations.

AC: each check failing alone produces `validation_failed` naming that check; validation never runs in the API process.

### F-10 Approvals, JIT execution & verification
Approve/reject (dual control for high impact), execute (mint, verify, consume token, then the simulated adapter), the `post-action-verification` job (independent re-read and expected-signal check → `verified`, or a rollback through the adapter → `rolled_back` plus a `verification_failed` notification and incident escalation), and the `proposal-expiry` job.

AC:
- The proposer approving → 403.
- The same approver twice → 409.
- High impact with 1 approval stays `validated`.
- Token reuse → 409.
- A token past `exp` → 409.
- Verification is independent of the executor's own result: it reads events after `executed_at`.

### F-11 Evidence ledger
Append (within business transactions), list, verify, export, the hourly `ledger-integrity` job and the immutability trigger.

AC:
- An UPDATE or DELETE on `evidence_entries` raises.
- A tampered payload (with the trigger bypassed in tests by superuser `ALTER TABLE … DISABLE TRIGGER`) → verify reports `brokenAt`.
- Concurrent appends produce a gapless seq.

### F-12 Scheduled work & jobs (has_scheduled_work)
Job schedulers registered with BullMQ `upsertJobScheduler` (idempotent on restart), all in UTC. Each job is `run<Name>(deps, now)` with `run<Name>ForOrg(orgId, now)`.

The effect assertions below are the binding test contract (INV-M1). Each test is written failing first and asserts the business effect plus a run-twice-one-effect check.

| Trigger (job key) | Schedule | Condition | Action / effect asserted | Idempotency |
|---|---|---|---|---|
| `sla-escalation` | every 1 min | incident `open/triaged`, `sla_due_at ≤ now`, `escalated_at IS NULL` | sets `escalated_at`, appends `incident.escalated` ledger entry, inserts `sla_escalation` notification | `UPDATE … WHERE escalated_at IS NULL` + unique notification (org,kind,subject) |
| `proposal-expiry` | every 1 min | proposal `validated/approved`, `expires_at ≤ now` | status `expired`, tokens revoked, ledger entry | conditional `UPDATE … WHERE status IN (…)` |
| `post-action-verification` | every 1 min | proposal `executed`, `verify_after ≤ now`, `claimed_at IS NULL` | `verified` (no events from target after execution with severity ≥ medium in category of action) or `rolled_back` (+ rollback adapter call, notification, incident escalation) | claim `UPDATE action_executions SET claimed_at=now() WHERE … claimed_at IS NULL` |
| `retention-sweep` | daily 02:00 UTC | events/alerts (suppressed) older than `org.retention_days`; nonces > 24 h | rows deleted; ledger untouched; counts recorded | window claim |
| `ledger-integrity` | hourly | per org | chain verified; broken → `org.ledger_status='broken'`, `ledger_broken` notification, critical alert | window claim; notification unique |
| `posture-snapshot` | every 15 min | per org | inserts `posture_snapshots` row with KPIs + radar + threat level | window claim + unique(org,taken_at) |
| `ingest-watchdog` | every 1 min | batch `queued` > 10 min; then every non-synthetic source (F-20) | status `failed`, error `"watchdog: no worker receipt in 10 min"`; regrade source health, append `source.health_changed`, `source_health` notification on silent/degraded | conditional update (on the old status / old health) + unique notification (org,kind,subject@time) |
| `synthetic-feed` | every 30 s, if `SYNTHETIC_FEED=1` | demo orgs | inserts synthetic events through the same normalize path | window claim |
| `outbound-drain` (F-23) | every 30 s | `outbound_messages` `pending`/`failed` with `next_attempt_at ≤ now` (≤ 50 per run, claimed with `FOR UPDATE SKIP LOCKED`) | runs the outbound validator, then sends; records status, attempts, checks; terminal outcomes append `outbound.sent`/`delivered`/`blocked`/`dlq` | window claim + claim lease; a row leaves `pending` once |
| `alert-escalation` (F-23) | every 1 min | real `new` alerts at or above a policy's severity whose age passed its `escalate_after_min` and whose level is below the next | `escalation_level + 1`, `escalated_at`, `alert.escalated` messages (policy recipients + `escalate_to`), `alert_escalation` notification, `alert.escalated` ledger entry | `UPDATE … WHERE escalation_level = n` + unique message (…, business_ref `alert:<id>:escalation:<n>`) + unique notification (org,kind,`<id>@<n>`) |

**Window claim.** `INSERT INTO job_runs(job_key, window_start, …) ON CONFLICT (job_key, window_start) DO NOTHING RETURNING id`. If no row is returned, the run is skipped. The run then records `processed`, `skipped`, `failed` and status `completed` or `errored`.

**Observable outcome.** The Admin → Jobs screen shows the last run, its counts and any error per job. `errored` runs and `failed > 0` render in critical color. `/api/health` `checks.worker` reports heartbeat age and queue counts.

**Startup.** Missed windows within 24 h are caught up once, and older ones are skipped and logged. **Shutdown.** The in-flight job completes or is requeued within 10 s.

**Test trigger.** `POST /api/v1/admin/jobs/:key/run` with body `{now?}`, mounted only in non-production with `ENABLE_TEST_TRIGGERS=1` and an admin session. The Functional Smoke Test asserts `escalated_at` and the `sla_escalation` notification on an incident past its SLA. Where the trigger is mounted, it calls `sla-escalation`. On the production stack, where the trigger answers 404, it backdates `sla_due_at` and waits for the scheduled every-minute worker run (scripts/smoke-functional.mjs `sla-escalation` flow).

### F-13 Analytics (scope expansion)
Range selector 7/30/90 d. Alerts by severity per day, incidents opened/closed, MTTR (hours, open→closed), FP rate (suppressed `false_positive` / total dispositioned alerts), suppressions by reason, proposals by outcome.

Detection quality:
- Precision = linked / (linked + false_positive).
- FP rate.
- Wilson 95% CI for each.

Charts have a legend and designed empty/loading/error states. Values derived from synthetic data are badged.

### F-14 Audit log + export (scope expansion)
Filterable audit list (auditor/admin) with CSV/JSON export. The export itself is audited.

### F-15 Admin users & roles (scope expansion)
List members, create a member (temporary password shown once, must change on first login), change role, deactivate (revokes sessions). The last admin and self-demotion are guarded (422 `LAST_ADMIN`).

### F-16 Jobs operations view (scope expansion)
Job definitions, next run, last outcome, 24 h totals, and run history with counts and errors.

### F-17 Assets inventory (scope expansion)
List and filter by kind/criticality, edit criticality, business-critical flag and owner. Changes affect QAI scoring and validation guards and are audited.

### F-18 Data export (scope expansion)
Incidents CSV/JSON, ledger NDJSON/CSV (with hashes so it can be re-verified offline) and audit CSV/JSON. Limited to 5/h/user. Downloads use `Content-Disposition: attachment`.

### F-19 Settings & sessions (scope expansion)
Profile (name), password change (verifier), theme preference, active sessions list, "sign out other sessions".

### F-20 Source integrations & health (merge wave 1, ADR-066/067)
Typed sources (`generic`, `firewall`, `edr`, `identity`; `synthetic` stays system-owned) on the same signed generic ingest API; vendor-specific connectors stay a Non-Goal. Each source has a silence window (5–1440 min, default 60). The `ingest-watchdog` job grades every non-synthetic source each minute:
- `unknown`: disabled or never seen;
- `silent`: no non-test batch within the silence window (outranks degraded);
- `degraded`: a failed batch in 24 h, or ≥ 20 % of the last hour's events rejected;
- `healthy`: otherwise.

A grade change is a conditional update that appends `source.health_changed` to the ledger; a change to `silent` or `degraded` also inserts a `source_health` notification linking to the source. Batches record accepted, duplicates, rejected and per-event errors. The source detail screen shows 24 h stats, recent errors, the batch log, a **test event** (one `test=true` batch through the real pipeline, excluded from health and stats) and curl / Node.js signing snippets that never contain the secret.

AC: a source with no batch past its window turns `silent` with exactly one notification and one ledger entry; the chain still verifies; test events never change health; duplicates are not counted as rejected.

### F-21 Enrichment & threat-intel indicators (merge wave 1, ADR-066/068)
Events accept optional `srcIp`, `dstIp`, `dstPort` (F-29), `domain`, `url`, `fileHash`, `user` (invalid values reject the event). The worker enriches once per batch, set-based:
- **Networks.** Public IPs are looked up in `ip_networks` for country, ASN, AS org and coarse geo (`geo_source = dataset`); sender-supplied geo wins (`geo_source = sender`). The table ships empty, so nothing is enriched until the owner loads a licensed dataset; nothing is invented.
- **Indicators.** Tenant IOCs (ip, domain, url, hash, email) are matched on exact normalized value; expired indicators never match. Matches set `indicator_ids`, bump `hits`/`last_hit_at` and fire `R-INTEL`.

The Threat Intel screen lists, filters, adds, imports (CSV) and deletes indicators; writes need `intel:write` and are audited (`indicator.created/updated/deleted/imported`).

AC: an event whose `dstIp` equals a confidence-90 indicator raises a critical R-INTEL alert in that org only; an expired indicator raises nothing; with an empty `ip_networks` table events carry no ASN.

### F-22 Detection rules (merge wave 2, ADR-066/069)
Tenant custom rules beside the six built-ins, in a declarative DSL (no code, no SQL): `match.all` or `match.any` of 1–20 conditions `{field, op, value}` over normalized event fields (category, severity, title, summary, IPs, domain, url, file hash, user, country, ASN, source kind, asset kind, business-critical, indicator match), with operators typed per field (eq, neq, in, nin, contains, starts_with, cidr, gte, lte, exists); `groupBy` (asset, src_ip, dst_ip, user_name, domain, country) decides what one alert is about; an optional threshold `{count 2..10000, windowSec 60..86400}`. Each version also sets severity, confidence, ATT&CK technique, required evidence (≤ threshold count) and a suppression window (a re-match within it extends the alert).

- Versions are immutable; editing creates the next version and alerts record the version that fired them.
- New rules start disabled. A dry run replays the definition over the newest ≤ 5000 events in a window (1–168 h) and reports matches and would-be alerts without writing anything.
- The Detection Rules screen lists built-ins and custom rules with 30-day alerts, false-positive rate (false_positive suppressions ÷ dispositioned), mean time to triage and MTTR (alert → incident closed); templates (port scan, admin sign-in from an unexpected country, cloud console failures) seed the editor.
- Writes are admin-only (`rules:write`), audited (`rule.created/version_created/updated/deleted`) and appended to the ledger (`rule.created`, `rule.version_created`, `rule.enabled_changed`, `rule.renamed`, `rule.deleted`).

AC: an enabled rule raises one `R-CUSTOM` alert per group with its rule version, ATT&CK id and risk score, and a re-match within the suppression window raises its event count; a disabled rule or another org's rule never fires; a threshold rule fires only once the count is reached; a dry run writes no alert, ledger entry or rule; an UPDATE of a rule version fails at the database.

### F-23 Notifications: email, signed webhooks, escalation (merge wave 3, ADR-066/071)
Port of ThreatWatch F-9. Admins define **notification policies**: a channel (email or webhook), a minimum severity, recipients (addresses and/or roles) or a delivery endpoint, and optional escalation (`escalate_after_min`, `escalate_to`).

- **Enqueue, never send inline.** Correlation inserts one `outbound_messages` row per (policy target, alert) for every new real alert at or above the policy's severity, inside the batch transaction (the outbox). Synthetic alerts never notify. A re-match extends the alert and queues nothing.
- **Outbound validator.** `outbound-drain` claims due rows and runs a validator before any send: `policy_active`, `target_active` (webhook endpoint exists and is enabled), `url_safe` (https, no credentials, every resolved address public; the socket is pinned to the checked address), `transport_ready` (email), `org_rate` (≤ 500 sent per org per hour, else deferred 5 min), `payload_allowlist`. A failed check blocks (or, for a transient DNS failure, retries); every check result is stored and shown.
- **Payload allowlist.** Messages carry only alertId, rule, severity, title, riskScore, mitreTechnique, eventCount, firstSeenAt, escalationLevel and a console URL. Never event bodies, assets, users or secrets.
- **Webhooks** are signed Standard Webhooks style (`webhook-id`, `webhook-timestamp`, `webhook-signature: v1,<base64 HMAC-SHA256>` over `id.timestamp.body`) with a per-endpoint secret shown once; redirects are never followed; 5 s timeout; 2xx is `delivered`; retries back off 1, 5, 30, 120, 360 min, then `dlq` after 6 attempts. Email (SMTP, TLS required by default) retries 1, 10, 60 min, then `dlq` after 3.
- **Escalation.** `alert-escalation` raises a still-`new` alert one level when it outlives a policy's `escalate_after_min`; responders and analysts can also escalate by hand from the alert detail. Each level notifies once.
- **Outbound log.** Every message with its checks, attempts, last HTTP status and error; admins can retry failed, dead-lettered or blocked messages (they re-run the validator).
- Writes are admin-only (`notify:write`), audited (`notify_policy.*`, `delivery_endpoint.*`, `outbound.retried`) and appended to the ledger.

AC: a new high alert queues exactly one message per matching policy target and a second matching event queues none; a webhook message carries a signature that verifies with the endpoint secret; a message whose policy is disabled is blocked with the reason and never sent; an alert past its escalation delay escalates once per level and a re-run adds nothing; an unsafe URL (http, loopback, RFC 1918, link-local, credentials) is rejected at save; another org's policies, endpoints, log rows and alerts answer 404.

### F-24 Search, threat-source profiles and event explorer (merge wave 4, ADR-066/072)
Port of ThreatWatch F-7 (its Intel screen is already F-21).

- **Global search.** One box in the header. The query is classified (ip, hash, url, email, id, domain, else text; or forced) and normalized like intel indicators. Results are grouped (events, alerts, assets, indicators, rules) with counts and up to 5 items each; a group the role cannot read is omitted. Hostnames match assets by name or external id; users match events by user name.
- **Threat-source profile** for an IP or domain over the last 90 days: matching indicators, network (ASN, org, country and its source; "not known" without sender or dataset data, ADR-068), first and last seen, event count, a daily severity timeline, targeted assets, triggered detections and related alerts, with a pivot to the Event Explorer.
- **Event Explorer.** A filterable, cursor-paged table (severity, category, time range, synthetic, text, and exact IP, domain, user, hash, asset and source pivots, all in the URL) and an event drawer: normalized fields, the raw SHA-256 (the raw body is never stored), batch, enrichment and the alerts it fed.
- Read-only: no writes, ledger entries or new permissions. Every query is tenant-scoped.

AC: searching an IP that has events returns events and alerts groups whose items link to the profile, event and alert; a viewer's search has no indicators or rules group; a profile for an IP with no events in 90 days says so; `/events?ip=…` returns only events with that source or destination IP; another org's event id answers 404 and its events never appear in search.

### F-25 KEV exposure and posture score (merge wave 5, ADR-066/073)
Port of ThreatWatch F-25 (KEV exposure) and F-24 (posture score), merged into the existing posture model: there is one score.

- **KEV catalogue.** Each org holds its own copy of the CISA Known Exploited Vulnerabilities catalogue. The release bundles a dated copy (`apps/api/data/kev-<version>.json`, attribution in NOTICE); an admin imports it, or pastes a newer feed (≤ 4 MB). Import is an upsert by CVE: unchanged entries are untouched, withdrawn CVEs are removed, and every asset is re-matched.
- **Exposure.** An asset lists the software it runs as normalized `vendor:product` keys (≤ 50) and whether it is internet-facing. A key matches a KEV entry on normalized vendor and product. Matches open findings; a removed match resolves its finding, and a returning one reopens the same row. Software and exposure changes are ledgered (`asset.exposure_changed`).
- **Posture score.** `100 −` the sum of capped, explained penalties: untriaged non-synthetic alerts (critical 8, high 3, medium 1; ≤ 35), open KEV findings (8 per internet-facing, 3 per other; ≤ 30), enabled source health (silent 5, degraded 3; ≤ 15), 7-day mean time to triage (> 1 h 5, > 4 h 10), incidents past SLA (5 each; ≤ 10) and a broken ledger (10). Grade A ≥ 90, B ≥ 80, C ≥ 70, D ≥ 60, else F. The posture-snapshot job stores score, grade and factors; the trend is the last snapshot of each UTC day over 30 days.
- **Radar.** The six-axis CSF radar stays as the posture profile and feeds only the threat level. Its protect axis also counts assets with open KEV findings.

AC: a second import of the same catalogue creates, updates and removes nothing; adding `microsoft:exchange-server` to an internet-facing asset opens its KEV findings, sets the KEV penalty to 30 and the score to 70 in an otherwise clean org, and removing it resolves them and restores 100; re-adding reopens without duplicates; a malformed software key answers 400; an analyst's import answers 403; another org's asset id answers 404 and its findings never appear.

### F-26 Scheduled daily and weekly reports (merge wave 6, ADR-066/074)
Port of ThreatWatch F-10. STARSHIELD already has the team, audit, outbound and jobs pieces, so this adds reports only.

- **Report.** A report covers a UTC range of at most 90 days and stores an immutable, versioned snapshot computed once at creation: events and alerts with the previous equal period for comparison, critical and high alerts, alerts by severity and status, incidents opened and closed, MTTR, false positives, an hourly trend, the top 10 sources and assets, up to 50 critical and high alerts, and the posture score and grade at the end of the range. Synthetic data is counted separately, never mixed into real totals.
- **On demand.** Admins and auditors create a report for any past range and send it to active org members. Every role can read reports.
- **Schedules.** A schedule is daily or weekly (on a weekday) at a UTC hour, with 1 to 20 recipients who must be active members. The hourly `report-generate` job makes one report per due slot (the 24 h or 7 d ending at the slot), queues one `report.ready` email per recipient who is still an active member and advances `next_run_at` by compare-and-set, so a slot is never generated twice. At most 25 schedules per org.
- **Email.** The email carries the title, period, headline totals and a link; the report itself stays behind sign-in. It goes through the outbound validator, which adds a `recipient_member` check (the address must still be an active member when it sends) and the `report.ready` payload allow-list.

AC: a due daily schedule produces exactly one report for the 24 h ending at its slot, one `report.created` ledger entry and one email per active-member recipient, and a re-run adds nothing; a deactivated recipient gets no email; a weekly schedule without a weekday answers 400; a non-member recipient answers 400 `not_org_member`; a 26th schedule answers 422; an analyst's create answers 403 while a viewer reads; another org's report and schedule answer 404; the validator blocks a queued report email once its recipient is deactivated.

### F-27 AI investigation copilot (merge wave 7, ADR-066/075)
Port of ThreatWatch F-26 (ADR-046), routed through STΛRGΛZΞR instead of a vendor SDK.

- **Opt-in.** Off for every org until an admin turns it on in Settings (`org:manage`); the toggle is ledgered. A server without `STARGAZER_URL` and `STARGAZER_API_KEY` answers 503 `COPILOT_UNAVAILABLE`, so the feature ships dark.
- **Bounded, redacted context.** On request the server builds the alert's normalized fields (rule, severity, status, risk, MITRE, times), its asset (kind, name, criticality, software, open KEV findings), up to 20 matched indicators and the latest 20 linked events (normalized fields only: no user names, owners, raw payloads or ids). Email local parts and credential shapes are redacted, strings are truncated, and the JSON is wrapped in `<alert_context>` as untrusted data.
- **Validate before egress (master rule 3).** The copilot egress validator checks `org_opted_in`, `route_configured`, `context_bounded` (≤ 24 000 chars), `no_email_local_parts`, `no_secret_shapes` and `untrusted_wrapper`. Any failure answers 422 and nothing leaves. Model egress lives only in `lib/copilot` (INV-50).
- **Propose-only.** One structured call to STARGAZER's paid, no-data-collection route returns `{summary, assessment, confidence, recommendedSteps 1..8, falsePositiveIndicators ≤ 5}`; the server re-validates it strictly. The suggestion is advisory text: it never changes the alert, never creates a proposal and never skips the validator or approval. A decline answers 422 `COPILOT_DECLINED` and stores nothing.
- **Replayable.** Each stored suggestion is ledgered `alert.copilot_suggested` with the model, assessment, token counts and the SHA-256 of exactly the context that left.

AC: with the org off, POST answers 409 and nothing is sent; an analyst's org toggle answers 403 and an admin's is ledgered once (a repeat is a no-op); an unconfigured server answers 503; the context contains no email local part, user name, owner, raw hash or id; a success answers 201, stores one row whose `context_sha256` equals the hash of the sent context and appends one ledger entry; a decline answers 422 and a transport failure 503, neither storing anything; a viewer reads the panel but POST answers 403; another org's alert answers 404 on GET and POST.

### F-28 Live updates (merge wave 8, ADR-076)
Port of ThreatWatch's live hub. The console updates without a reload.

- **Hints, not data.** A signed-in tab holds one WebSocket to `/api/v1/live`. The server sends `change` frames naming a kind (`events, alerts, incidents, proposals, notifications, sources`) and optionally an id. The client invalidates the matching cached lists and refetches them through the REST API, so every read is still scoped and role-checked there.
- **Origin first.** The upgrade's Origin must equal `PUBLIC_ORIGIN`, checked before the session (INV-51); then the session cookie must be valid (401) and must not owe a password change (428). A socket only ever receives its own org's hints.
- **Bounded inbound.** A client may only send `{"type":"ping"}`, at most 20 per 10 s and 1 KB; anything else closes the socket (1008). At most 8 sockets per user.
- **Session end.** Logout, revoke or password change closes that session's sockets at once (4401); a 60 s re-check catches expiry, idle timeout, deactivation and removed membership. The client stops on 4401.
- **Header pill.** Live, Reconnecting or Offline. Without a socket the console behaves as before (lists refresh on navigation).

AC: a cross-site Origin answers 403 before any session lookup; no cookie answers 401; a password-change session answers 428; a valid session receives `hello`; a hint for org A reaches org A's socket and never org B's; a frame other than ping, or a 21st frame in 10 s, closes with 1008; logout closes the socket with 4401; a revoked session is closed by the re-check; an ingested batch that opens an alert produces an `alerts` hint after commit; an analyst on /alerts sees the new alert without reloading.

### F-29 THREAT//MAP (merge wave 8 part 2, ADR-077)
Port of ThreatWatch's link map. It shows where attacks come from, what they reach and by which vector.

- **Destination port.** Events accept an optional `dstPort` (integer 0..65535; out of range rejects the event). It is stored as `events.dst_port`.
- **Links.** `GET /threat-map` folds the actor's org's events in the window (with a source IP and a destination IP or asset) into source → target links. The target is the event's asset when it has one, otherwise the destination IP. Each link carries its event count, worst severity, source geo and top 5 destination ports. At most 300 links are returned, busiest first; `truncated` says when more exist.
- **Zones and vectors, derived, never stored.** A public source IP is `external`, a private one `internal`. A target is `dmz` when it is an internet-facing asset, `internal` for any other asset, `external` for a public IP and `internal` for a private one. Vectors: `exposed_admin` (inbound to an admin or database port), `web` (inbound to 80/443/8080/8443), `lateral` (internal to internal on an admin port), `egress` (internal to external), `high_port` (an uncommon port ≥ 1024), `recon` (≥ 8 ports on one link, or one source reaching ≥ 10 targets). A per-vector rollup counts links, sources, events and worst severity.
- **Screen.** `/map` draws up to 80 arcs from geolocated sources to "your network" on the existing SVG map, with a vector filter and a links table that is the map's text alternative. Live hints (F-28) refetch it.

AC: an ingested `dstPort` is stored, and 70000 or −1 rejects only that event; an external source hitting an internet-facing asset on 22 and 443 is one DMZ link with `exposed_admin` and `web`; internal 445 traffic is `lateral`; internal to a public IP on 4444 is `egress` + `high_port`; a source reaching 10 targets marks each of its links `recon`; the severity filter and window apply; a bad `since` or severity answers 400; org B never sees org A's links or assets.

### Key Workflows

### W1 — Observe and normalize
- W1.1 An admin opens Sources and creates source "edr-gateway".
- W1.2 The UI shows the HMAC secret once, with a copy control.
- W1.3 The external system POSTs a signed batch to `/api/v1/ingest/events`, which returns 202.
- W1.4 The worker normalizes the events: Zod, geo truncation, CSF category, asset upsert.
- W1.5 **Terminal:** the analyst sees the new events in the TACTICAL feed (and geo markers), with severity and source.

### W2 — Correlate and investigate
- W2.1 Correlation creates alerts from events.
- W2.2 The analyst filters Alerts by severity/status and opens an alert, which shows its related events and asset.
- W2.3 The analyst links the alert to a new incident (title, severity).
- W2.4 The incident detail shows linked alerts, events and the attack-path graph.
- W2.5 **Terminal:** the analyst assigns themself and transitions the incident to `triaged`.

### W3 — Propose and optimize
- W3.1 On the incident, the analyst chooses an objective and requests QAI candidates.
- W3.2 The ranked candidates show score breakdown components, impact and reversibility.
- W3.3 **Terminal:** the analyst submits one candidate as a proposal (status `proposed`).

### W4 — Independently validate
- W4.1 The worker validator evaluates all checks against the current policy version.
- W4.2 **Terminal:** the proposal detail shows the validation verdict and every check with its detail (`validated` or `validation_failed`).

### W5 — Authorize and respond
- W5.1 Approver A opens the Governance queue (filter `validated`) and approves with a note.
- W5.2 For high impact, the proposal stays `validated` with "1 of 2 approvals". Approver B approves and the status becomes `approved`. The proposer cannot approve (control disabled with an explanation; the API returns 403).
- W5.3 A responder executes with explicit confirmation: a JIT token is minted and consumed, and the simulated adapter runs.
- W5.4 **Terminal:** the proposal shows `executed` with the execution record (adapter SIMULATED, token jti, expiry).

### W6 — Verify and recover
- W6.1 After `verify_after`, the verification job independently re-reads target telemetry.
- W6.2 The proposal shows `verified`, or `rolled_back` with the rollback record and a notification.
- W6.3 The responder moves the incident `triaged → contained → eradicated → recovered`.
- W6.4 **Terminal:** the responder closes the incident with a resolution. The ledger shows the full chain for the incident.

### W7 — Learn and govern
- W7.1 The analyst suppresses a noisy alert as `false_positive`.
- W7.2 Analytics shows the FP rate, precision with 95% CI, MTTR and proposal outcomes for the chosen range.
- W7.3 The admin publishes policy version n+1 (e.g. lowers `maxBlastRadius`) with a note.
- W7.4 **Terminal:** the new version is current, a ledger entry records it, and subsequent validations cite that version.

### W8 — Operate the portal
- W8.1 A viewer signs in and lands on COMMAND: KPIs, posture radar, QAI recommendations, governance status, ledger status.
- W8.2 They switch to TACTICAL: globe markers, threat feed tabs, attack path, response timeline.
- W8.3 RBAC hides or disables controls the role lacks, and the API enforces them (403).
- W8.4 An auditor verifies the ledger chain and exports it.
- W8.5 **Terminal:** the auditor downloads the ledger export and the audit log export.

### W9 — Integrate and enrich (merge wave 1)
- W9.1 An admin creates a typed source (kind `firewall`) and opens its detail page: health, 24 h stats, batch log and signing snippets.
- W9.2 The admin sends a test event; a `test` batch appears in the batch log and health is unchanged.
- W9.3 The admin changes the silence window; a source past it turns `silent` and a `source_health` notification links back to it.
- W9.4 An analyst adds an indicator (or imports a CSV) on Threat Intel.
- W9.5 **Terminal:** a matching event raises an R-INTEL alert, and the indicator's hit count rises.

### W10 — Tune detections (merge wave 2)
- W10.1 An admin opens Detection Rules and reviews built-in rule metrics (alerts, FP rate, triage time, MTTR).
- W10.2 The admin starts a rule from a template, edits its conditions and dry-runs it against recent events.
- W10.3 The admin creates the rule (disabled) and enables it.
- W10.4 The admin saves a new version on the rule detail page; the version history shows both.
- W10.5 **Terminal:** a matching event raises an R-CUSTOM alert whose detail shows the risk score, ATT&CK technique and a link to the rule version.

### W11 — Notify and escalate (merge wave 3)
- W11.1 An admin opens Notifications and adds a webhook endpoint; the signing secret is shown once.
- W11.2 The admin creates an email policy (high and above, with an escalation contact) and a webhook policy on that endpoint.
- W11.3 A matching event raises an alert; its detail lists the queued email and webhook messages.
- W11.4 The admin escalates the alert by hand; the detail shows the alert.escalated messages.
- W11.5 **Terminal:** the Outbound Log shows the messages with their validator checks, filters by outcome, and retries a blocked message.

### W12 — Investigate a threat source (merge wave 4)
- W12.1 An analyst searches an IP from the header search box; results come back grouped by type with counts.
- W12.2 The analyst opens the threat-source profile: matching indicators, network (ASN, org, country when known), first and last seen, and the daily timeline.
- W12.3 The profile lists the targeted assets, triggered detections and related alerts; each links to its record.
- W12.4 **Terminal:** the analyst pivots to the Event Explorer filtered on that IP and opens an event's drawer (normalized fields, raw hash, enrichment, linked alerts).

### W13 — Manage known-exploited exposure (merge wave 5)
- W13.1 An admin imports the bundled CISA KEV catalogue from the Vulnerabilities screen and searches it.
- W13.2 The admin opens an asset, lists the software it runs and marks it internet-facing; a malformed key is rejected inline.
- W13.3 The asset shows its open KEV findings with CISA's required action and due date (overdue spelled out); the Vulnerabilities screen lists them org-wide.
- W13.4 Command shows the posture grade and the KEV factor pulling the score down, with the 30-day trend.
- W13.5 **Terminal:** the admin removes the software and the findings resolve; a viewer sees everything read-only.

### W14 — Report to leadership (merge wave 6)
- W14.1 An admin opens Reports and creates a report for the last 7 days.
- W14.2 The report shows totals against the previous period, the trend, the top sources and assets, critical alerts (each linking to its alert) and the posture grade.
- W14.3 The admin sends it to two members; a non-member address is not offered.
- W14.4 The admin creates a weekly Monday 07:00 UTC schedule and sees its next run.
- W14.5 **Terminal:** a viewer opens the report read-only, with no create, send or schedule controls.

### W15 — Ask the copilot (merge wave 7)
- W15.1 An analyst opens an alert and sees that the copilot is off for the org.
- W15.2 An admin turns the copilot on in Settings.
- W15.3 The analyst asks the copilot and sees an advisory assessment, summary and next steps, marked advisory.
- W15.4 **Terminal:** a viewer opens the same alert, reads the stored suggestion and has no Ask control.

### W16 — Watch it arrive (merge wave 8)
- W16.1 An analyst opens Alerts and the header shows Live.
- W16.2 A source sends a batch that opens a new alert.
- W16.3 **Terminal:** the new alert appears in the analyst's list without a reload.

### W17 — See where it comes from (merge wave 8 part 2)
- W17.1 An analyst opens THREAT//MAP from the nav.
- W17.2 The map shows source arcs, the attack-vector list and the links table.
- W17.3 The analyst picks a vector and the table narrows to its links.
- W17.4 **Terminal:** the analyst opens a target asset from a link row.

## 6. UI Surface

| Screen | Route | Example | Roles | Binds | Workflow steps | User controls | States |
|---|---|---|---|---|---|---|---|
| Landing | `/` | `/` | public | `GET /auth/session` | W8.1 | sign in link, register link, theme toggle | success |
| Sign in | `/login` | `/login` | public | `POST /auth/login`, `GET /auth/session` | W8.1 | email, password, submit | idle, submitting, error (field + form), success redirect |
| Register | `/register` | `/register` | public | `POST /auth/register` | W8.1 | org name, name, email, password, submit | idle, submitting, field errors (breached, too short), success |
| COMMAND | `/command` | `/command` | all | `GET /overview`, `GET /posture`, `GET /notifications`, `POST /notifications/read`, `POST /auth/logout` | W8.1, W8.3, W13.4 | notification mark-read, sign out, link to Vulnerabilities | empty (no data yet), loading skeleton, error retry, success, SYNTHETIC badge, posture factors and trend (trend needs two days) |
| TACTICAL | `/tactical` | `/tactical` | all | `GET /events`, `GET /events/geo`, `GET /timeline`, `GET /incidents/:id/attack-path`, `GET /incidents` | W1.5, W8.2 | severity tabs, category filter, time range (since), synthetic filter, incident picker for attack path | empty, loading, error, success |
| THREAT//MAP | `/map` | `/map` | all | `GET /threat-map` | W17 | severity tabs, time window, attack-vector filter (toggle), asset links | empty, loading, error, success, truncated |
| Alerts | `/alerts` | `/alerts` | all (write: admin, responder, analyst) | `GET /alerts` | W2.2, W7.1 | status filter, severity filter, search q, sort, pagination | empty, loading, error, success |
| Alert detail | `/alerts/[id]` | `/alerts/00000000-0000-4000-8000-000000000001` | all | `GET /alerts/:id`, `POST /alerts/:id/link`, `POST /alerts/:id/suppress`, `GET /incidents`, `GET /alerts/:id/notifications`, `POST /alerts/:id/escalate`, `GET /alerts/:id/copilot`, `POST /alerts/:id/copilot` | W2.2, W2.3, W7.1, W11.3, W11.4, W15.1, W15.3, W15.4 | link to existing incident select, new incident title + severity, suppress reason select + note, escalate now, retry message, ask copilot | loading, error, not-found, success |
| Incidents | `/incidents` | `/incidents` | all | `GET /incidents`, `POST /incidents`, `GET /incidents/export` | W2.3, W8.5 | status filter, severity filter, search q, sort, pagination, new incident form, export format | empty, loading, error, success |
| Incident detail | `/incidents/[id]` | `/incidents/00000000-0000-4000-8000-000000000002` | all (write per RBAC) | `GET /incidents/:id`, `POST /incidents/:id/transition`, `GET /incidents/:id/attack-path`, `POST /incidents/:id/candidates`, `POST /proposals`, `GET /users` | W2.4, W2.5, W3.1, W3.2, W3.3, W6.3, W6.4 | transition target, note, owner select, resolution, objective select, candidate submit + rationale | loading, error, not-found, success, transition error |
| Governance queue | `/governance` | `/governance` | all | `GET /proposals` | W5.1 | status filter, impact filter, pagination | empty, loading, error, success |
| Proposal detail | `/governance/[id]` | `/governance/00000000-0000-4000-8000-000000000003` | all (approve: approver/admin; execute: responder/admin) | `GET /proposals/:id`, `POST /proposals/:id/approve`, `POST /proposals/:id/reject`, `POST /proposals/:id/execute` | W4.2, W5.1, W5.2, W5.3, W5.4, W6.2 | approve note, reject reason, execute confirm checkbox | loading, error, not-found, success, pending validation (polling) |
| Evidence ledger | `/ledger` | `/ledger` | admin, auditor | `GET /ledger`, `GET /ledger/verify`, `GET /ledger/export` | W6.4, W8.4, W8.5 | type filter, subject filter, pagination, verify button, export format | empty, loading, error, success, broken-chain alert |
| Assets | `/assets` | `/assets` | all (write: admin, responder) | `GET /assets`, `PATCH /assets/:id` | W1.4 | kind filter, criticality filter, search q, pagination, criticality edit, business-critical toggle, owner, asset name links to the detail page | empty, loading, error, success |
| Analytics | `/analytics` | `/analytics` | all | `GET /analytics` | W7.2 | range select (7d/30d/90d) | empty, loading, error, success |
| Policies | `/policies` | `/policies` | admin (read: all but viewer) | `GET /policies`, `POST /policies` | W7.3, W7.4 | maxBlastRadius, change windows, denied actions per asset kind, allowIrreversible, minConfidence (0–100), note | loading, error, validation errors, success |
| Sources | `/sources` | `/sources` | admin (read: all but viewer) | `GET /sources`, `POST /sources`, `POST /sources/:id/rotate`, `PATCH /sources/:id` | W1.1, W1.2, W9.1 | source name, kind select, enable toggle, rotate, name links to detail; health badge | empty, loading, error, success, secret-once dialog |
| Source detail | `/sources/[id]` | `/sources/00000000-0000-4000-8000-000000000004` | admin (read: all but viewer) | `GET /sources/:id`, `GET /sources/:id/batches`, `POST /sources/:id/test`, `PATCH /sources/:id` | W9.1, W9.2, W9.3 | send test event, silence window (minutes), batch log pagination, snippet language toggle | loading, error, not-found, success, empty batch log, no errors |
| Threat Intel | `/intel` | `/intel` | all but viewer (write: admin, responder, analyst) | `GET /indicators`, `POST /indicators`, `POST /indicators/import`, `DELETE /indicators/:id` | W9.4, W9.5 | type filter, search, pagination, add form (type, value, threat type, confidence), CSV file import, delete with confirm | empty, loading, error, success, import result with per-line errors |
| Detection Rules | `/rules` | `/rules` | all but viewer (write: admin) | `GET /rules`, `POST /rules`, `PATCH /rules/:id`, `POST /rules/dry-run` | W10.1, W10.2, W10.3 | new rule toggle, template select, name, condition field/operator/value, add/remove condition, match all/any, group by, threshold, severity, confidence, ATT&CK id, required evidence, suppression, dry-run window, dry run, enable toggle | empty, loading, error, success, dry-run result, field errors |
| Rule detail | `/rules/[id]` | `/rules/00000000-0000-4000-8000-000000000005` | all but viewer (write: admin) | `GET /rules/:id`, `POST /rules/:id/versions`, `PATCH /rules/:id`, `DELETE /rules/:id`, `POST /rules/dry-run` | W10.4, W10.5 | enable toggle, delete with confirm, new-version editor (conditions, severity, confidence, ATT&CK id, change note), dry run | loading, error, not-found, success |
| Notifications | `/notifications` | `/notifications` | all but viewer (write: admin) | `GET /notification-policies`, `POST /notification-policies`, `PATCH /notification-policies/:id`, `DELETE /notification-policies/:id`, `GET /delivery-endpoints`, `POST /delivery-endpoints`, `PATCH /delivery-endpoints/:id`, `POST /delivery-endpoints/:id/rotate`, `DELETE /delivery-endpoints/:id` | W11.1, W11.2 | policy name, channel, minimum severity, recipients, roles, endpoint select, escalate after, escalate to, enable toggle, delete; endpoint name, URL, add, pause/resume, rotate secret with confirm, delete | empty, loading, error, success, secret-shown-once dialog, field errors |
| Outbound log | `/notifications/log` | `/notifications/log` | all but viewer (retry: admin) | `GET /notifications/log`, `POST /notifications/log/:id/retry` | W11.5 | channel filter, outcome filter, pagination, retry | empty, loading, error, success, filtered-empty |
| Search | `/search` | `/search?q=203.0.113.10` | all | `GET /search` | W12.1 | query box, type select, submit; group links and "show all" pivots | idle (no query), loading, error, no-results, success (grouped) |
| Threat profile | `/threats/[kind]/[value]` | `/threats/ip/203.0.113.10` | all | `GET /threat-profile` | W12.2, W12.3 | pivot to events, indicator, asset and alert links | loading, error, not-seen (no events in 90 d), success |
| Vulnerabilities | `/vulnerabilities` | `/vulnerabilities` | all (import: admin) | `GET /vulnerabilities/findings`, `GET /vulnerabilities`, `POST /vulnerabilities/import` | W13.1, W13.3 | findings/catalogue view, open/resolved filter, catalogue search, ransomware and affected filters, pagination, import dialog (bundled or pasted JSON) | loading, error, forbidden, empty (no findings / no catalogue), filtered-empty, success |
| Asset detail | `/assets/[id]` | `/assets/<uuid>` | all (write: admin, responder) | `GET /assets/:id`, `PATCH /assets/:id` | W13.2, W13.3, W13.5 | software list, internet-facing toggle, save | loading, error, not-found, forbidden, no findings, success, read-only |
| Reports | `/reports` | `/reports` | all (write: admin, auditor) | `GET /reports`, `POST /reports`, `GET /reports/schedules`, `POST /reports/schedules`, `PATCH /reports/schedules/:id`, `DELETE /reports/schedules/:id`, `GET /reports/recipients` | W14.1, W14.4 | new-report dialog (title, from, to), pagination, schedule dialog (name, cadence, hour, weekday, recipients), enable toggle, edit, delete (confirmed) | loading, error, forbidden, empty (no reports / no schedules), success, read-only |
| Report detail | `/reports/[id]` | `/reports/<uuid>` | all (send: admin, auditor) | `GET /reports/:id`, `POST /reports/:id/send`, `GET /reports/recipients` | W14.2, W14.3, W14.5 | alert links, send dialog (recipient checkboxes) | loading, error, not-found, success, read-only |
| Event Explorer | `/events` | `/events` | all | `GET /events`, `GET /events/:id` | W12.4 | severity, category, time range, synthetic, text search, IP/domain/user/hash pivots, clear filters, pagination, open event drawer (deep link `?event=`), close | empty, filtered-empty, loading, error, success, drawer loading/not-found |
| Users | `/admin/users` | `/admin/users` | admin | `GET /users`, `POST /users`, `PATCH /users/:id` | W8.3 | name, email, role select, active toggle | empty, loading, error, success, temp-password dialog |
| Jobs | `/admin/jobs` | `/admin/jobs` | all but viewer | `GET /jobs`, `GET /jobs/runs` | W6.1 | job key filter, pagination | empty, loading, error, success, errored-run highlight |
| Audit log | `/audit` | `/audit` | admin, auditor | `GET /audit`, `GET /audit/export` | W8.5 | action filter, actor filter, pagination, export format | empty, loading, error, success |
| Settings | `/settings` | `/settings` | all | `GET /auth/session`, `POST /auth/password`, `GET /auth/sessions`, `DELETE /auth/sessions`, `GET /org/settings`, `PATCH /org/settings` | W8.3, W15.2 | name, current/new password, theme select, sign out others, org copilot toggle (admin) | loading, error, field errors, success |

The designed 404 (`not-found.tsx`) and the global error boundary are also delivered.

### Design System

The binding source is the Design Template. The tokens are in DECISIONS.md (ADR-001..003), and every screen uses them; no screen invents its own.

- **Archetype.** `saas`: a persistent app shell, a composed dashboard hierarchy, and designed tables and forms.
- **Fonts.**
  - Rajdhani 600/700 for display labels and panel titles (uppercase, letter-spacing .08em).
  - Orbitron 700 for the wordmark only.
  - Inter 400/500/600 for body.
  - JetBrains Mono 400/500 for ids, hashes, timestamps and numerics (`font-variant-numeric: tabular-nums`).
  - All are self-hosted (`next/font/local` from `artifacts/template-assets`; Orbitron via `next/font/google`, downloaded at build time).
- **Type scale (px).** 11 (min label; the template uses 10, raised for legibility, ADR-024) / 12 / 13 (body) / 14 / 16 / 18 / 20 / 24 / 30 / 40.
- **Color.** Dark is the default and light is derived. Use the full token set in DECISIONS.md. Severity colors are semantic: critical, high, medium, low, info. QAI is violet. A single accent (cyan) marks primary actions and the active state.
- **Spacing.** A 14 px rhythm for panel padding and gaps; steps 4/6/8/10/12/14/16/20/24/32.
- **Radius.** Panel 10, card 8, tab 6, badge 4, pill 999.
- **Shadow.** The only one is the active-nav inset bar `inset 3px 0 0 var(--color-accent)`. Depth otherwise comes from surface steps and borders.
- **Motion.**
  - Durations: 120 ms (hover/focus), 200 ms (panel/drawer), `ssPulse` 2 s (live indicators only).
  - Every animation is disabled under `prefers-reduced-motion: reduce`.
  - No decorative scroll motion.
- **Layout doctrine.**
  - Header 56 px. The sidebar is 224 px at ≥ 1024 px and an off-canvas drawer below that. Content max-width is 1440 px.
  - The COMMAND hierarchy is a lead metric (threat level gauge + posture), then supporting KPIs, then detail panels. It is never an equal-weight tile grid.
- **Signature components.** App shell + nav, data table (sort, filter, pagination, row actions, designed states), forms with inline validation, metric card with trend, threat gauge, posture radar (SVG), attack-path graph (SVG), response timeline, severity badge, SYNTHETIC/SIMULATED badges, toasts, dialog.
- **Breakpoints.** 640 / 768 / 1024 / 1280. Audited at 375 / 768 / 1280.
- **Accessibility.**
  - WCAG 2.2 AA.
  - 44 px primary targets on mobile.
  - Visible focus ring (2 px accent, offset 2).
  - Skip link.
  - Charts carry a text alternative (a table or `aria-label` summary).

## 7. Error contract

Every non-2xx response is `application/problem+json`:

```json
{ "type": "https://starshield.local/errors/validation", "title": "Validation failed", "status": 400,
  "detail": "1 field failed validation", "instance": "/api/v1/incidents",
  "code": "VALIDATION_FAILED", "requestId": "4b1e…",
  "errors": [ { "field": "title", "message": "Title must be at least 3 characters", "code": "TOO_SMALL" } ] }
```

| code | status | When |
|---|---|---|
| `VALIDATION_FAILED` | 400 | Zod failure; `errors[]` per field |
| `UNAUTHENTICATED` | 401 | no or expired session |
| `INVALID_CREDENTIALS` | 401 | login failure (uniform) |
| `SIGNATURE_INVALID`, `TIMESTAMP_SKEW`, `REPLAY` | 401 | ingest |
| `CSRF_INVALID` | 403 | missing or wrong CSRF token |
| `FORBIDDEN` | 403 | missing permission |
| `SELF_APPROVAL` | 403 | proposer approving |
| `PASSWORD_CHANGE_REQUIRED` | 403 | must change password first |
| `NOT_FOUND` | 404 | missing **or foreign-tenant** resource |
| `INVALID_TRANSITION`, `ALREADY_DECIDED`, `TOKEN_INVALID`, `CONFLICT`, `COPILOT_DISABLED` (F-27) | 409 | state conflicts |
| `PRECONDITION_FAILED`, `PASSWORD_BREACHED`, `LAST_ADMIN`, `COPILOT_DECLINED` (F-27) | 422 | business rule |
| `RATE_LIMITED` | 429 | with `Retry-After` |
| `INTERNAL` | 500 | no internal detail |
| `UNAVAILABLE`, `COPILOT_UNAVAILABLE` (F-27) | 503 | dependency down |

The client maps `errors[].field` onto form controls (`aria-invalid`, `aria-describedby`) and shows `requestId` in error panels ("Reference: 4b1e…").

## 8. Configuration & environment

All configuration comes from environment variables, validated at boot by Zod (`apps/api/src/config/env.ts`, `apps/web/src/env.ts`). The process exits ≠ 0 naming the bad key. An optional variable that is present but empty is treated as absent. `.env.example` mirrors this table.

| Variable | Purpose | Required | Service |
|---|---|---|---|
| `NODE_ENV` | `development`/`test`/`production` | optional (default development) | api, worker, web |
| `APP_VERSION` | release version reported by health | optional (default package version) | api, worker |
| `GIT_COMMIT` | commit sha reported by health | optional | api |
| `API_PORT` | api listen port | optional (4000) | api |
| `WORKER_METRICS_PORT` | worker metrics/heartbeat port | optional (4001) | worker |
| `DATABASE_URL` | Postgres via pgbouncer (`?pgbouncer=true`) | **required** | api, worker |
| `DIRECT_DATABASE_URL` | direct Postgres for migrations | **required** for migrate | migrate |
| `DATABASE_READ_URL` | read replica | optional (falls back to primary) | api |
| `REDIS_URL` | Redis (sessions cache, limits, queues, replay) | **required** | api, worker |
| `SESSION_SECRET` | HMAC for CSRF/session derivation, ≥ 32 chars | **required** in prod (SECRET) | api |
| `ACTION_TOKEN_SECRET` | HMAC for JIT action tokens, ≥ 32 chars | **required** in prod (SECRET) | api, worker |
| `SOURCE_SECRET_KEY` | 32-byte base64 AES-256-GCM key for source secrets | **required** in prod (SECRET) | api, worker |
| `COOKIE_SECURE` | `true` (HTTPS edge) | optional (true) | api |
| `PUBLIC_ORIGIN` | external https origin for CSRF/Origin check, e.g. `https://localhost:8443` | **required** | api, web |
| `TRUST_PROXY` | hops to trust for client IP (Caddy) | optional (1) | api |
| `LOG_LEVEL` | pino level | optional (info) | api, worker |
| `HIBP_MODE` | `on`/`off` (off rejected in production) | optional (on) | api |
| `HIBP_TIMEOUT_MS` | HIBP timeout | optional (2000) | api |
| `INGEST_MAX_SKEW_SEC` | HMAC clock skew | optional (300) | api |
| `RATE_LIMIT_GLOBAL_PER_MIN` | per-IP global limit | optional (600) | api |
| `SYNTHETIC_FEED` | `1` enables the synthetic connector job | optional (0) | worker |
| `ENABLE_TEST_TRIGGERS` | `1` mounts the test trigger route (ignored in production) | optional (0) | api |
| `SMTP_HOST` | SMTP server for F-23 email; unset outside production captures mail in memory, unset in production blocks email (`transport_ready`) | optional | worker |
| `SMTP_PORT` | SMTP port | optional (587) | worker |
| `SMTP_USER` / `SMTP_PASS` | SMTP credentials | optional (SECRET) | worker |
| `SMTP_FROM` | From header | optional (`STARSHIELD <alerts@starshield.local>`) | worker |
| `SMTP_REQUIRE_TLS` | `1` requires STARTTLS/TLS | optional (1) | worker |
| `ALLOW_HTTP_DELIVERY` | `1` permits http:// webhook URLs for local receivers; ignored in production; never permits a private address | optional (0) | api, worker |
| `STARGAZER_URL` | F-27 copilot: STΛRGΛZΞR model gateway base URL (https in production); unset means the copilot answers 503 | optional | api |
| `STARGAZER_API_KEY` | F-27 bearer key for STARGAZER, ≥ 16 chars | optional (SECRET) | api |
| `STARGAZER_ROUTE` | F-27 route name; must be the paid, no-data-collection route | optional (`paid-no-data-collection`) | api |
| `STARGAZER_TIMEOUT_MS` | F-27 request timeout | optional (60000) | api |
| `COPILOT_MODE` | `stargazer`, or `fake` (deterministic offline port for tests and e2e; rejected in production) | optional (stargazer) | api |
| `JOB_CONCURRENCY` | BullMQ worker concurrency | optional (8) | worker |
| `SEED_DEMO` | `1` seeds a demo org + users on migrate | optional (0) | migrate |
| `SEED_PASSWORD` | demo users' password (≥ 12, not breached) | required when `SEED_DEMO=1` (SECRET) | migrate |
| `API_INTERNAL_URL` | api URL for server-side fetches from web (`http://api:4000`) | **required** | web |
| `HTTP_PORT` / `HTTPS_PORT` | Caddy host ports | optional (8080 / 8443, probed free) | caddy |
| `GRAFANA_PORT` / `PROMETHEUS_PORT` | host ports | optional (3301 / 9391) | grafana, prometheus |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin | **required** (SECRET) | grafana |
| `POSTGRES_PASSWORD` | DB superuser password | **required** (SECRET) | postgres |
| `CADDY_SITE` | site address (`localhost` → tls internal; a domain → ACME) | optional (localhost) | caddy |

Header buffers:
- Node services start with `--max-http-header-size=32768`.
- Caddy's defaults are generous; this is recorded in DECISIONS.

## 9. Performance targets

These are derived from the Success Metrics and scale tier **large** (50k+ concurrent). The client model is 50k concurrent users polling COMMAND/TACTICAL every 10 s, which is about 5,000 req/s of read traffic at the edge and ~3% writes.

| Operation | Target (p95 at sustained load) | Notes |
|---|---|---|
| `GET /overview` | < 150 ms | Redis cache 15 s + ETag 304 |
| `GET /events`, `/alerts`, `/incidents` (page 25) | < 250 ms | composite indexes (org_id, time) |
| `GET /events/geo` | < 250 ms | aggregated cells, 24 h window |
| `GET /search`, `GET /threat-profile` | < 400 ms | exact matches on (org_id, field) indexes, 90-day window |
| `GET /proposals/:id`, `/incidents/:id` | < 200 ms | |
| `POST /ingest/events` (500 events) | < 300 ms to 202 | async normalize |
| ingest → visible in feed | < 15 s p95 | worker + 10 s polling |
| `POST /proposals/:id/execute` | < 400 ms | token mint + consume + adapter |
| `GET /ledger/verify` (100k entries) | < 5 s | streaming hash |
| `POST /auth/login` | < 400 ms | Argon2id dominates |
| Error rate under load | < 0.5% non-429 errors | |
| Availability | 99.9% monthly (production target) | |

**Load test.** k6 `constant-arrival-rate`. The scenario mix is 60% overview, 20% events, 10% alerts, 5% incidents, 4% geo and 1% proposals detail, using pre-authenticated sessions. Stages: ramp, sustained 10 min at the single-host target, then step-up to saturation. The results report achieved RPS, p95/p99, error rate and the headroom to 5,000 RPS, extrapolated to replicas. The single-host ceiling is residual R1.

The web front end meets WEB.md budgets: LCP ≤ 2.5 s, CLS ≤ 0.1, and Lighthouse ≥ 95 a11y/best-practices with performance ≥ 85 on the audited screens.

## 10. Test plan

| Layer | Tooling | Scope |
|---|---|---|
| Unit | Vitest | shared: QAI scoring determinism, state tables, RBAC, geo truncation, canonical JSON; api lib: HMAC, tokens, password verifier (HIBP mocked), problem+json |
| Integration | Vitest + real Postgres + Redis (compose `test` profile / CI services) + supertest | every §3 endpoint happy + error paths; tenant isolation (second org on every `/:id`); CSRF; rate limits; ledger trigger + concurrency; job effect tests |
| E2E | Playwright against the production build behind Caddy | one `test('Wn …')` per workflow W1–W8 driving the UI to its terminal step |
| Web baseline | `scripts/web-baseline.mjs` | every §6 screen × 375/768/1280 × light/dark |
| Smoke | `smoke-test.sh` over https:// | health, fail-fast env, shutdown, auth, a workflow, scheduled job effect, error envelope, headers, TLS redirect |
| Load | k6 | §9 |

**Mapping:**

| Item | Unit | Integration | E2E |
|---|---|---|---|
| F-1 | env schema, problem | health, request id, shutdown | — |
| F-2 | password policy, HIBP | auth routes, limiter, CSRF, enumeration | W8 |
| F-3 | — | — | W8 (theme, nav) |
| F-4 | HMAC, skew, replay | ingest route + worker normalize, geo bounds | W1 |
| F-5 | generator | synthetic-feed effect | W1 (badge) |
| F-6 | rules | correlation, dedupe, suppress | W2, W7 |
| F-7 | transitions | transitions, preconditions, ledger | W2, W6 |
| F-8 | scoring | candidates, create | W3 |
| F-9 | checks | validator per-check failure | W4 |
| F-10 | token | approve/reject/execute, dual control, reuse, expiry, verification | W5, W6 |
| F-11 | chain | trigger, concurrency, verify, tamper | W6, W8 |
| F-12 | — | 8 effect tests (failing-first, run-twice) | W6 |
| F-13 | Wilson CI | analytics | W7 |
| F-14 / F-18 | — | audit + exports | W8 |
| F-23 | signing, SSRF guard, templates, payload allowlist, backoff | policies, endpoints, enqueue, escalation, log/retry, isolation; outbound-drain and alert-escalation effect tests | W11 |
| F-26 | slot maths, schemas, report payload allow-list, report email | reports, send, schedules, RBAC, isolation, validator recipient check; report-generate effect test | W14 |
| F-27 | output schema, redaction, egress validator, STARGAZER adapter contract, fake, env rules | off/on toggle (ledgered), 503/409/422/201, context contents, hash, ledger, RBAC, isolation | W15 |
| F-28 | origin check, hint parse, kind→tag map, frame policy, URL | origin/401/428/hello, org fan-out isolation, ping/1008/rate, logout 4401, recheck, ingest hint after commit | W16 |
| F-29 | vector classifier, severity rank, arc path and arc cap | dst_port ingest + rejection, links/zones/vectors/fan-out recon/rollup, severity + window filters, 400s, org isolation, 401 | W17 |
| F-15 | — | users, last-admin guard | W8 |
| F-16 | — | jobs endpoints | W6 |
| F-17 | — | assets patch affects validation | W1 |
| F-19 | — | password change, sessions revoke | W8 |

The Test & Doc Budget follows BUILD.md. Every §5 AC has at least one named test.
