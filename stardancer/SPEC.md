# SPEC.md — STΛRDANCΞR

Behavioural contract · build_depth **comprehensive** · scale tier **large** · version 0.1.0

All HTTP paths in this document are relative to the `/api` mount (the browser calls `https://<host>/api/v1/...`). Path parameters are written `:name` in §3 and `[name]` in §6.

## §1 Overview & scope

### Problem
Security and operations teams run dozens of tools that each understand one domain. Alerts arrive without system context, inventories go stale, relationships between assets, identities and data are invisible, and AI recommendations arrive without governance. STARDANCER keeps one living **System Graph** of an organisation's environment. Its deterministic reasoning layer, the **Mirror Brain Ψ**, explains what is connected, what changed, what could happen next, and what action is safe. Every conclusion and every action is recorded in a tamper-evident **evidence ledger**.

### Personas
| Persona | Role in app | Goals |
|---|---|---|
| Security analyst | `analyst` | investigate events, find attack paths, prioritise vulnerabilities, reconstruct incidents |
| SRE / operations engineer | `analyst` | correlate health degradation with changes, find root cause, assess change impact |
| Security / platform lead | `admin` | connect sources, register AI agents, set policies, approve high-risk actions |
| CISO / executive / auditor | `viewer` | understand posture, trust the evidence, check compliance |
| Tenant owner | `owner` | billing, organisation settings, membership |
| AI agent (machine) | agent API key | request context, propose actions |
| Telemetry source (machine) | ingest API key | push observations |

### Non-Goals (restated from RESEARCH.md; never specced)
- Replace every monitoring or security platform, or replace an organisation's SIEM.
- Automatically remediate every finding, or execute unapproved infrastructure changes. Auto-approval is limited to low-risk, reversible actions that an explicit policy allows.
- Treat AI-generated hypotheses as facts. Predictions are always labelled and kept separate from observed facts.
- Claim perfect prediction accuracy, or make cryptographic claims such as "mathematically unbreakable".
- Require quantum-computing hardware, or depend on a simulated AI entity to deliver intelligence.
- Display fabricated telemetry as live. Simulated data is created only on request and is always badged SIMULATED.
- Introduce autonomous self-modification without validation. Rule-weight changes are `model_update` proposals that require human approval.
- Allow AI to modify governance boundaries. Agents can't create or edit policies and can't approve anything.
- Build every possible integration before the core System Graph works. v1 ships one generic ingest API.
- Become a generic chatbot, or treat LLM output as authoritative. There is no LLM in v1.
- Expose sensitive telemetry unnecessarily. Everything is org-scoped, logs are redacted, and the viewer role is limited.

### Scope expansion (comprehensive)
These capabilities are implied by the product and promoted to features (ADR-028…ADR-033): team and invitations admin surface (F-04), ingest log and source health (F-05), evidence export (F-15), billing with refunds and disputes (F-17), operations health correlation (F-18), and profile and password management (F-03).

## §2 Data model

PostgreSQL via Prisma. Tables are snake_case (`@@map`) and ids are UUID v4. Every tenant table has `org_id NOT NULL` with an index led by `org_id`. Timestamps are `timestamptz`. There are no floating-point columns anywhere:
- money is `BIGINT` minor units
- coordinates are `DECIMAL(9,6)`
- scores are `INT`

Ownership: **T** = tenant-owned (`org_id`), **U** = user-owned, **G** = global.

| Table | Own | Key fields | Constraints |
|---|---|---|---|
| `orgs` | G | `name` (2–80), `slug` (unique, `^[a-z0-9-]{3,40}$`), `plan` (`explorer`/`constellation`/`sovereign`), `stale_after_minutes` INT (5–10080, default 1440) | `slug` UNIQUE |
| `users` | U | `email` (lower-cased, ≤254), `name` (1–80), `password_hash` (argon2id), `disabled_at` | `email` UNIQUE |
| `memberships` | T | `org_id`, `user_id`, `role` (`owner`/`admin`/`analyst`/`viewer`) | UNIQUE(`org_id`,`user_id`); ≥1 owner per org (service rule) |
| `sessions` | U | `token_hash` (sha256 hex), `user_id`, `org_id`, `csrf_token`, `expires_at`, `idle_expires_at`, `ip`, `user_agent` | `token_hash` UNIQUE |
| `invites` | T | `email`, `role` (not owner), `token_hash`, `expires_at` (+72 h), `accepted_at`, `revoked_at`, `created_by` | `token_hash` UNIQUE; one open invite per (org,email) — service rule |
| `password_resets` | U | `user_id`, `token_hash`, `expires_at` (+1 h), `used_at`, `created_by` | `token_hash` UNIQUE |
| `sources` | T | `name` (1–80), `kind` (`threat_watch`/`cyberops`/`synapse_pulse`/`alpha_zero`/`edr`/`siem`/`cloud`/`identity`/`cmdb`/`manual`/`simulated`), `status` (`active`/`paused`), `last_seen_at` | UNIQUE(`org_id`,`name`) |
| `agents` | T | `name`, `description`, `capabilities` TEXT[] (action types), `max_risk` (`low`/`medium`/`high`), `status` (`active`/`suspended`) | UNIQUE(`org_id`,`name`) |
| `api_keys` | T | `kind` (`ingest`/`agent`), `source_id`?, `agent_id`?, `name`, `prefix` (8 chars), `key_hash` (sha256), `last_used_at`, `revoked_at`, `created_by` | `prefix` UNIQUE, `key_hash` UNIQUE; CHECK exactly one of source_id/agent_id matching kind |
| `entities` | T | `external_id` (1–200), `type` (`service`/`host`/`container`/`database`/`identity`/`network`/`application`/`data_store`/`endpoint`/`cloud_resource`/`ai_agent`/`site`), `name`, `environment` (`production`/`staging`/`development`/`unknown`), `criticality` (`low`/`medium`/`high`/`critical`), `crown_jewel` BOOL, `exposure` (`internal`/`external`), `owner`, `tags` TEXT[] (≤20), `attributes` JSONB (≤16 KB), `lat`/`lng` DECIMAL(9,6) (both or neither, ranges CHECKed), `location_label`, `health_score` INT 0–100 NULL, `state` (`active`/`stale`/`retired`), `containment` (`none`/`isolated`/`disabled`), `layout_x`/`layout_y` INT, `first_seen_at`, `last_seen_at`, `simulated` BOOL, `source_id` | UNIQUE(`org_id`,`external_id`) |
| `relationships` | T | `from_id`, `to_id` (entities, same org), `type` (`depends_on`/`connects_to`/`runs_on`/`authenticates_to`/`stores_data_in`/`exposes`/`manages`/`member_of`), `confidence` INT 0–100, `state` (`active`/`stale`/`retired`), `first_seen_at`, `last_seen_at`, `simulated` | UNIQUE(`org_id`,`from_id`,`to_id`,`type`); CHECK `from_id <> to_id` |
| `observations` | T | `source_id`, `idempotency_key` (1–200), `kind` (`entity`/`relationship`/`event`/`vulnerability`/`change`/`health`), `payload` JSONB, `status` (`received`/`processed`/`rejected`), `error`, `received_at`, `processed_at` | UNIQUE(`org_id`,`source_id`,`idempotency_key`) |
| `events` | T | `entity_id`?, `source_id`, `kind` (`alert`/`auth`/`network`/`process`/`deployment`/`outage`/`health`/`integrity`), `severity` (`info`/`low`/`medium`/`high`/`critical`), `title`, `details` JSONB, `occurred_at`, `simulated` | index (`org_id`,`occurred_at` desc) |
| `vulnerabilities` | T | `entity_id`, `cve` (`^CVE-\d{4}-\d{4,7}$`), `title`, `cvss10` INT 0–100 (CVSS×10), `epss_bp` INT 0–10000, `kev` BOOL, `exploit_available` BOOL, `status` (`open`/`mitigated`/`accepted`), `first_seen_at` | UNIQUE(`org_id`,`entity_id`,`cve`) |
| `changes` | T | `entity_id`, `kind` (`config`/`deploy`/`permission`/`network`/`created`/`removed`/`attribute`), `summary`, `before`/`after` JSONB, `actor`, `occurred_at`, `drift` BOOL, `expected` BOOL, `simulated` | index (`org_id`,`occurred_at` desc) |
| `health_samples` | T | `entity_id`, `score` INT 0–100, `latency_ms` INT, `error_rate_bp` INT, `at` | index (`org_id`,`entity_id`,`at`) |
| `feed_items` | T | `kind` (`event`/`change`/`drift`/`prediction`/`proposal`/`investigation`/`integrity`/`ingest`), `severity`, `title`, `ref_type`, `ref_id`, `created_at`, `simulated` | index (`org_id`,`created_at` desc) |
| `investigations` | T | `title` (3–160), `summary` (≤4000), `status` (`open`/`in_progress`/`resolved`/`closed`), `severity`, `created_by`, `assignee_id`?, `conclusion` (≤8000), `root_cause_entity_id`?, `resolved_at` | — |
| `investigation_items` | T | `investigation_id`, `kind` (`event`/`entity`/`change`/`prediction`/`vulnerability`), `ref_id`, `note`, `added_by` | UNIQUE(`investigation_id`,`kind`,`ref_id`) |
| `investigation_notes` | T | `investigation_id`, `author_id`, `body` (1–8000) | — |
| `predictions` | T | `rule` (`impact_propagation`/`exploit_path`/`staleness_trend`/`change_velocity`/`health_degradation`), `entity_id`, `fingerprint`, `title`, `statement`, `horizon_at`, `confidence` INT 0–100, `uncertainty`, `assumptions` JSONB[], `evidence` JSONB[], `impact` JSONB, `status` (`proposed`/`reviewed`/`confirmed`/`refuted`/`expired`), `reviewed_by`, `reviewed_at`, `review_note` | UNIQUE(`org_id`,`fingerprint`) |
| `rule_weights` | T | `rule`, `weight` INT 10–200 (percent), `confirmed`, `refuted` INT | UNIQUE(`org_id`,`rule`) |
| `policies` | T | `action_type`, `name`, `auto_approve_max_risk` (`none`/`low`/`medium`), `approver_min_role` (`analyst`/`admin`/`owner`), `enabled`, `version` INT, `hash` | UNIQUE(`org_id`,`action_type`) |
| `proposals` | T | `idempotency_key`, `proposer_type` (`user`/`agent`/`system`), `proposer_user_id`?, `proposer_agent_id`?, `action_type` (`isolate_endpoint`/`disable_credential`/`revoke_relationship`/`mitigate_vulnerability`/`restore_endpoint`/`model_update`), `target_entity_id`?, `params` JSONB, `rationale` (10–4000), `risk` (`low`/`medium`/`high`/`critical`), `reversible`, `status` (see §5 state machine), `policy_id`, `policy_hash`, `validation` JSONB, `decision_reason`, `result` JSONB, `decided_at`, `executed_at` | UNIQUE(`org_id`,`idempotency_key`) |
| `approvals` | T | `proposal_id`, `approver_id`, `decision` (`approve`/`reject`), `reason` | UNIQUE(`proposal_id`,`approver_id`) |
| `evidence_records` | T | `seq` BIGINT, `kind`, `actor_type` (`user`/`agent`/`source`/`system`), `actor_id`, `subject_type`, `subject_id`, `payload` JSONB, `prev_hash`, `hash` (sha256 hex) | UNIQUE(`org_id`,`seq`); no UPDATE/DELETE path in code |
| `evidence_checkpoints` | T | `seq`, `hash`, `hmac`, `status` (`verified`/`broken`), `detail` | UNIQUE(`org_id`,`seq`) |
| `controls` | G | `code` (e.g. `PR.AA-01`), `function` (`GV`/`ID`/`PR`/`DE`/`RS`/`RC`), `title`, `description`, `rule` | `code` UNIQUE |
| `control_evaluations` | T | `control_id`, `status` (`pass`/`partial`/`fail`/`not_applicable`), `score` INT 0–100, `findings` JSONB, `evaluated_at` | index (`org_id`,`control_id`,`evaluated_at` desc) |
| `health_snapshots` | T | `at`, `overall`/`infra`/`apps`/`security`/`network` INT 0–100, `counters` JSONB, `top_risks` JSONB | index (`org_id`,`at` desc) |
| `billing_subscriptions` | T | `plan`, `status` (`incomplete`/`active`/`past_due`/`canceled`), `stripe_customer_id`, `stripe_subscription_id`, `current_period_end`, `cancel_at_period_end` | UNIQUE(`org_id`) |
| `checkout_requests` | T | `idempotency_key`, `plan`, `amount` BIGINT, `currency` CHAR(3), `status` (`open`/`completed`/`expired`), `stripe_session_id`, `created_by` | UNIQUE(`org_id`,`idempotency_key`), `stripe_session_id` UNIQUE |
| `payments` | T | `stripe_payment_intent_id`, `amount`, `amount_refunded` BIGINT, `currency`, `status` (`succeeded`/`failed`/`refunded`/`partially_refunded`/`disputed`/`dispute_won`/`dispute_lost`), `description` | `stripe_payment_intent_id` UNIQUE |
| `refunds` | T | `payment_id`, `idempotency_key`, `stripe_refund_id`, `amount` BIGINT (>0, ≤ remaining), `reason`, `status` (`pending`/`succeeded`/`failed`), `created_by` | UNIQUE(`org_id`,`idempotency_key`), `stripe_refund_id` UNIQUE |
| `stripe_events` | G | `event_id`, `type`, `org_id`?, `received_at`, `processed_at` | `event_id` UNIQUE |
| `job_claims` | G | `key` (`<job>:<window-iso>`), `claimed_at`, `worker` | `key` UNIQUE |

Migrations: `server/prisma/migrations/<ts>_<name>/migration.sql`, each with a hand-written `down.sql` beside it.

## §3 API surface

Auth column:
- **S** = session cookie, plus `X-CSRF-Token` on unsafe methods; the minimum role follows.
- **K:ingest** / **K:agent** = `Authorization: Bearer sd_…` API key.
- **P** = public.
- **W** = Stripe signature.

All list endpoints take `?cursor=&limit=` (1–100, default 25) and return `{ items, nextCursor }`. Unsafe methods accept `Idempotency-Key` (1–200 chars). A replay with the same key returns the original status and body for 24 h; the same key with a different body returns 422 `IDEMPOTENCY_MISMATCH`. Rows whose Internal column says yes (internal) are machine-facing and bound by no screen.

| Endpoint | Auth | Input | Output | Internal |
|---|---|---|---|---|
| `POST /v1/auth/signup` | P | `{email, password (12–128), name, orgName}` | 201 session cookie + `{user, org, role}` | no |
| `POST /v1/auth/login` | P | `{email, password}` | 200 cookie + `{user, org, role}`; 401 `INVALID_CREDENTIALS`; 429 | no |
| `POST /v1/auth/logout` | S viewer | — | 204, cookie cleared | no |
| `GET /v1/auth/session` | S viewer | — | `{user, org, role, csrfToken}`; 401 when absent | no |
| `POST /v1/auth/password` | S viewer | `{currentPassword, newPassword}` | 204; other sessions revoked | no |
| `PATCH /v1/auth/profile` | S viewer | `{name}` | `{user}` | no |
| `GET /v1/auth/reset/:token` | P | — | `{email}` or 404 | no |
| `POST /v1/auth/reset/:token` | P | `{password}` | 204 | no |
| `GET /v1/invites/:token` | P | — | `{orgName, email, role}` or 404 | no |
| `POST /v1/invites/:token/accept` | P | `{name, password}` (new user) or `{password}` (existing) | 200 cookie + `{user, org, role}` | no |
| `GET /v1/org` | S viewer | — | `{org, plan, staleAfterMinutes}` | no |
| `PATCH /v1/org` | S owner | `{name?, staleAfterMinutes?}` | `{org}` | no |
| `GET /v1/members` | S viewer | — | `{items:[{id,name,email,role,joinedAt}]}` (viewer sees no emails) | no |
| `PATCH /v1/members/:id` | S admin | `{role}` | `{member}`; 409 `LAST_OWNER`; admin cannot grant owner | no |
| `DELETE /v1/members/:id` | S admin | — | 204; 409 `LAST_OWNER` | no |
| `POST /v1/members/:id/reset-link` | S admin | — | `{url, expiresAt}` shown once | no |
| `GET /v1/invites` | S admin | — | `{items}` | no |
| `POST /v1/invites` | S admin | `{email, role}` | 201 `{invite, url}` (url shown once) | no |
| `DELETE /v1/invites/:id` | S admin | — | 204 | no |
| `GET /v1/sources` | S viewer | — | `{items:[{…, keyCount, lastSeenAt, observations24h}]}` | no |
| `POST /v1/sources` | S admin | `{name, kind, description?}` | 201 `{source}` | no |
| `PATCH /v1/sources/:id` | S admin | `{name?, status?}` | `{source}` | no |
| `GET /v1/api-keys` | S admin | `?kind=` | `{items:[{id,kind,name,prefix,lastUsedAt,revokedAt}]}` | no |
| `POST /v1/sources/:id/keys` | S admin | `{name}` | 201 `{key: "sd_ingest_…"}` shown once | no |
| `POST /v1/agents/:id/keys` | S admin | `{name}` | 201 `{key: "sd_agent_…"}` shown once | no |
| `DELETE /v1/api-keys/:id` | S admin | — | 204 (revoke) | no |
| `POST /v1/ingest` | K:ingest | `{observations:[{idempotencyKey, kind, data}] (1–500)}` ≤1 MB | 202 `{accepted, duplicates, rejected:[{index,errors}]}`; 429 backpressure — yes (internal, machine) | yes (internal) |
| `GET /v1/observations` | S admin | `?sourceId=&status=` | paged observations (payload summary) | no |
| `POST /v1/demo/environment` | S admin | `{size: small\|medium\|large}` | 202 `{entities, relationships}` queued, all `simulated` | no |
| `DELETE /v1/demo/environment` | S admin | — | 202 removes simulated rows | no |
| `GET /v1/overview` | S viewer | — | `{health:{overall,infra,apps,security,network}, counters:{assets,relationships,criticalRisks,activeIncidents}, topRisks[], investigations[], prediction, flow:{observe,understand,predict,validate,act}, simulated, at}` | no |
| `GET /v1/feed` | S viewer | `?kind=&severity=` | paged feed items | no |
| `GET /v1/stream` | S viewer | — | `text/event-stream` (`feed`,`graph`,`health`,`proposals`,`predictions`,`ping`) | no |
| `GET /v1/graph` | S viewer | `?view=topology\|geo&type=&criticality=&environment=&q=&focus=&depth=1–3&includeStale=` | `{nodes:[{id,name,type,criticality,crownJewel,state,x,y,lat,lng,risk}], edges:[{id,from,to,type,confidence}], truncated, total}` (≤2 000 nodes) | no |
| `GET /v1/entities` | S viewer | `?type=&criticality=&state=&q=&sort=risk\|name\|lastSeen` | paged entities | no |
| `GET /v1/entities/:id` | S viewer | — | `{entity, risk, vulnerabilities, openEvents, predictions}` | no |
| `PATCH /v1/entities/:id` | S analyst | `{criticality?, crownJewel?, owner?, tags?}` | `{entity}` (evidence recorded) | no |
| `GET /v1/entities/:id/neighbors` | S viewer | `?depth=1–3&direction=in\|out\|both` | `{nodes, edges}` | no |
| `GET /v1/entities/:id/impact` | S viewer | `?depth=1–6` | `{affected:[{id,name,hops,criticality,path}], crownJewelsAffected, score}` | no |
| `GET /v1/entities/:id/timeline` | S viewer | `?from=&to=` | `{items:[{at,kind,title,ref}]}` events+changes+health | no |
| `GET /v1/events` | S viewer | `?severity=&kind=&entityId=&from=&to=` | paged events | no |
| `GET /v1/events/:id` | S viewer | — | `{event, entity, related:{changes, events}, investigations}` | no |
| `GET /v1/changes` | S viewer | `?drift=&kind=&entityId=` | paged changes | no |
| `GET /v1/changes/:id` | S viewer | — | `{change, entity, impact:{affected, crownJewelsAffected, score}, relatedEvents}` | no |
| `GET /v1/drift` | S viewer | — | `{stale:[entities], unexpected:[changes], summary}` | no |
| `GET /v1/investigations` | S viewer | `?status=&severity=` | paged | no |
| `POST /v1/investigations` | S analyst | `{title, summary?, severity, seed?:{kind, refId}}` | 201 `{investigation}` | no |
| `GET /v1/investigations/:id` | S viewer | — | `{investigation, items, notes}` | no |
| `PATCH /v1/investigations/:id` | S analyst | `{status?, assigneeId?, conclusion?, rootCauseEntityId?}` | `{investigation}`; 409 `INVALID_TRANSITION` | no |
| `POST /v1/investigations/:id/items` | S analyst | `{kind, refId, note?}` | 201 `{item}`; 409 duplicate | no |
| `POST /v1/investigations/:id/notes` | S analyst | `{body}` | 201 `{note}` | no |
| `GET /v1/investigations/:id/reconstruction` | S viewer | `?windowHours=1–168` | `{timeline:[{at,kind,title,entity,ref}], entities, firstSignal, lastSignal}` | no |
| `GET /v1/investigations/:id/root-cause` | S viewer | — | `{candidates:[{entity, score, reasons[], evidence[]}]}` | no |
| `GET /v1/vulnerabilities` | S viewer | `?status=&kev=&minScore=` | paged, sorted by contextual priority with `{priority, reasons[]}` | no |
| `PATCH /v1/vulnerabilities/:id` | S analyst | `{status}` | `{vulnerability}` | no |
| `GET /v1/attack-paths` | S viewer | `?targetId=&maxDepth=2–6&limit=1–20` | `{paths:[{nodes, edges, score, entry, reasons[]}], searched, truncated}` | no |
| `GET /v1/exposure` | S viewer | — | `{external:[{entity, reachableCrownJewels, vulnerabilities, score}]}` | no |
| `GET /v1/operations/health` | S viewer | `?hours=1–168` | `{snapshots:[…], degraded:[{entity, score, trend}]}` | no |
| `GET /v1/operations/correlations` | S viewer | `?entityId=&hours=1–72` | `{entity, degradation, changes:[{change, lagMinutes, score}], dependencies:[{entity, health}]}` | no |
| `GET /v1/predictions` | S viewer | `?status=&rule=` | paged | no |
| `GET /v1/predictions/:id` | S viewer | — | `{prediction, entity, evidence (resolved), assumptions}` | no |
| `POST /v1/predictions/:id/review` | S analyst | `{verdict: reviewed\|confirmed\|refuted, note?}` | `{prediction}`; 409 terminal | no |
| `POST /v1/predictions/run` | S analyst | — | 202 `{queued}` | no |
| `GET /v1/predictions/accuracy` | S viewer | — | `{rules:[{rule, weight, confirmed, refuted, precision}], suggestions:[{rule, proposedWeight, reason}]}` | no |
| `GET /v1/agents` | S viewer | — | `{items}` | no |
| `POST /v1/agents` | S admin | `{name, description?, capabilities[], maxRisk}` | 201 `{agent}` | no |
| `GET /v1/agents/:id` | S viewer | — | `{agent, keys, recentProposals, contextReads24h}` | no |
| `PATCH /v1/agents/:id` | S admin | `{capabilities?, maxRisk?, status?}` | `{agent}` | no |
| `GET /v1/agent/context` | K:agent | `?entityId=` | `{entity, neighbors, risks, policies}` — yes (internal, machine) | yes (internal) |
| `POST /v1/agent/proposals` | K:agent | `{actionType, targetEntityId, params, rationale}` + Idempotency-Key (required) | 201 `{proposal}` — yes (internal, machine) | yes (internal) |
| `GET /v1/proposals` | S viewer | `?status=&risk=` | paged | no |
| `POST /v1/proposals` | S analyst | `{actionType, targetEntityId?, params, rationale}` + Idempotency-Key (required) | 201 `{proposal}` | no |
| `GET /v1/proposals/:id` | S viewer | — | `{proposal, approvals, policy, validation, evidence}` | no |
| `POST /v1/proposals/:id/approve` | S analyst (policy role) | `{reason?}` | `{proposal}`; 403 `SELF_APPROVAL` / `ROLE_REQUIRED`; 409 state | no |
| `POST /v1/proposals/:id/reject` | S analyst | `{reason}` | `{proposal}` | no |
| `GET /v1/policies` | S viewer | — | `{items}` | no |
| `PUT /v1/policies/:actionType` | S admin | `{name, autoApproveMaxRisk, approverMinRole, enabled}` | `{policy}` (version+1, new hash) | no |
| `GET /v1/evidence` | S viewer | `?kind=&subjectType=&subjectId=` | paged, newest first | no |
| `GET /v1/evidence/:seq` | S viewer | — | `{record, prev, next, verified}` | no |
| `POST /v1/evidence/verify` | S analyst | `{fromSeq?}` | `{status: verified\|broken, checked, brokenAt?}` | no |
| `GET /v1/evidence/checkpoints` | S viewer | — | paged checkpoints | no |
| `GET /v1/evidence/export` | S admin | `?fromSeq=&toSeq=` | `application/x-ndjson` download (≤50 000 rows) | no |
| `GET /v1/compliance/controls` | S viewer | `?function=` | `{items:[{control, latest}] , coverage}` | no |
| `GET /v1/compliance/controls/:id` | S viewer | — | `{control, history[], findings}` | no |
| `POST /v1/compliance/evaluate` | S analyst | — | 202 `{queued}` | no |
| `GET /v1/billing` | S owner | — | `{subscription, plans[], payments(latest 10), mode}` | no |
| `POST /v1/billing/checkout` | S owner | `{plan}` + Idempotency-Key (required) | 201 `{checkoutId, url}` | no |
| `GET /v1/billing/checkout/:id` | S owner | — | `{checkout}` (simulated mode only, else 404) | no |
| `POST /v1/billing/checkout/:id/complete` | S owner | `{outcome: success\|decline}` | 202 (simulated mode only; emits signed webhook) | no |
| `GET /v1/billing/payments` | S owner | — | paged payments with refunds | no |
| `POST /v1/billing/refunds` | S owner | `{paymentId, amount, reason}` + Idempotency-Key (required) | 201 `{refund}`; 422 `REFUND_EXCEEDS_REMAINING` | no |
| `POST /v1/billing/cancel` | S owner | — | `{subscription}` (cancel at period end) | no |
| `POST /v1/billing/webhook` | W | raw Stripe event | 200 `{received}` — yes (internal, Stripe) | yes (internal) |
| `GET /health` | P | — | `{status, version, commit, checks:{db, redis}}` — yes (internal, probes) | yes (internal) |
| `GET /metrics` | P (edge-blocked) | — | Prometheus text — yes (internal) | yes (internal) |
| `GET /v1/openapi.json` | P | — | OpenAPI 3.1 document — yes (internal, tooling) | yes (internal) |

`/api/health` is served at `/api/health` (the health route lives outside `/v1`). The Caddy edge answers 404 on `/api/metrics`; Prometheus scrapes `/metrics` on the container network.

## §4 Security requirements

### Threat-derived controls (RESEARCH §5 STRIDE B1–B7)
| ID | Control | Boundary |
|---|---|---|
| SC-1 | TLS everywhere at the edge (Caddy), HTTP→HTTPS 308 redirect, HSTS when `HSTS_ENABLED` (1 day on loopback, 1 year on a public host; ADR-065) | B1 |
| SC-2 | CSP with a per-request nonce (`script-src 'self' 'nonce-…' 'strict-dynamic'`, `object-src 'none'`, `base-uri 'self'`, `frame-ancestors 'none'`, `connect-src 'self'`, `img-src 'self' data:`, `style-src 'self' 'unsafe-inline'` is allowed only for styles), plus `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`, and COOP `same-origin` | B1 |
| SC-3 | Session cookie `__Host-sd_session`: HttpOnly, Secure, SameSite=Lax, Path=/. CSRF double-submit (`X-CSRF-Token` must equal the session's `csrf_token`) on every unsafe cookie-auth request; Origin check | B1 |
| SC-4 | `trust proxy` = 1 (Caddy only). Body limits: 100 KB JSON default, 1 MB ingest, raw body for the webhook only | B2 |
| SC-5 | Rate limits (Redis):<br>• global 600 req/min/IP<br>• login 5/15 min per account plus 20/15 min per IP, checked **before** argon2 verify<br>• signup 5/h/IP<br>• ingest 120 req/min/key<br>• agent 60 req/min/key<br>• traversal endpoints (`graph`, `impact`, `attack-paths`) cost-weighted 30/min/user<br>• exports 5/h/org<br>Limited responses get 429 with `Retry-After` | B2/B4 |
| SC-6 | Tenant isolation: `orgId` comes from the session or key only, and every service query filters by `org_id`. A foreign or missing id returns 404 `NOT_FOUND` | B3 |
| SC-7 | Prisma only, parameterised. No `$queryRawUnsafe`; raw SQL uses tagged `$queryRaw` | B3 |
| SC-8 | API keys: 32 random bytes (base64url), SHA-256 stored, prefix lookup, constant-time compare, shown once, revocable. Scoped by kind (ingest keys cannot call agent routes, and the reverse) | B4 |
| SC-9 | Ingest validation (zod) per observation kind. Strings capped, JSON depth ≤ 8, ≤ 500 items per batch, queue-depth backpressure | B4 |
| SC-10 | Stripe webhook: `verifyWebhook` (constructEvent, 300 s tolerance) before `claimStripeEvent` (UNIQUE event_id); handlers idempotent; unknown event types acknowledged and ignored | B5 |
| SC-11 | Governance: agents propose only within `capabilities` and `max_risk`, cannot approve, and cannot touch policies. No self-approval. Policy hash pinned at decision time and re-checked by `assertExecutable` | B6 |
| SC-12 | Evidence ledger is append-only. `hash = sha256(prev_hash ‖ canonical(record))`, seq assigned under a per-org advisory lock in the same transaction. Hourly verify writes an HMAC-SHA256 checkpoint (`EVIDENCE_HMAC_KEY`). A break raises a critical `integrity` event | B7 |
| SC-13 | Passwords: argon2id (m=19456 KiB, t=2, p=1). NIST 800-63B verifier: 12–128 chars, all characters allowed, no composition rules, blocklist of common/breached passwords plus context words (email local-part, org name). Anti-enumeration: login performs a dummy verify when the user is missing; signup returns the same 201/409 timing class | B1 |
| SC-14 | Sessions: 256-bit token, SHA-256 stored, 12 h idle, 7 d absolute, rotated at login, all others revoked on password change, revoked on member removal | B1 |
| SC-15 | Logging: pino JSON with `requestId`. Redaction paths: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.newPassword`, `*.currentPassword`, `*.token`, `*.key`, `*.apiKey`, `*.secret`, `*.email`, `*.lat`, `*.lng` | all |
| SC-16 | Errors: RFC 9457 envelope (§7). 5xx carries no stack or internal message | all |
| SC-17 | Containers: non-root, read-only root filesystem where possible, tini as PID 1, no secrets baked into images; `.env` gitignored | ops |
| SC-18 | Dependency audit: `npm audit --omit=dev` must show no high/critical findings at the Security Audit gate | supply chain |

### RBAC matrix
| Capability | viewer | analyst | admin | owner |
|---|---|---|---|---|
| Read graph, events, changes, predictions, evidence, controls, proposals | ✓ | ✓ | ✓ | ✓ |
| Edit entity criticality / tags, triage vulnerabilities | — | ✓ | ✓ | ✓ |
| Investigations (create, notes, items, status) | — | ✓ | ✓ | ✓ |
| Review predictions, run Mirror Brain, run compliance eval, verify ledger | — | ✓ | ✓ | ✓ |
| Create proposals | — | ✓ | ✓ | ✓ |
| Approve/reject proposals (low/medium per policy `approver_min_role`) | — | ✓ | ✓ | ✓ |
| Approve high/critical proposals, approve `model_update` | — | — | ✓ | ✓ |
| Sources, ingest keys, agents, agent keys, policies, observations log | — | — | ✓ | ✓ |
| Members, invites, reset links (cannot grant/remove owner) | — | — | ✓ | ✓ |
| Demo environment generate/remove, evidence export | — | — | ✓ | ✓ |
| Org settings, billing, refunds, cancel | — | — | — | ✓ |

### Domain-signal checklists (traceable to COMPLIANCE.md)
| Item | Requirement | Where |
|---|---|---|
| geo-1 | Coordinates validated (lat −90..90, lng −180..180, both or neither) in zod and by a DB CHECK | ingest schema, migration |
| geo-2 | No third-party map/tile provider, so no billable proxy and no API key exposure | ADR-013 |
| geo-3 | Coordinates are sensitive topology: org-scoped and redacted from logs | SC-6, SC-15 |
| sched-1 | Every recurring trigger claims a `job_claims` key per window (at-most-once per window) | §5 F-08/F-12/F-16, INV-08 |
| sched-2 | Each trigger has an effect test | §10, INV-M1 |
| sched-3 | Jobs are bounded (batch sizes, timeouts) and failures go to retry/DLQ without crashing the worker | worker |
| pay-1 | Amounts are integer minor units plus ISO currency; no float | INV-04 |
| pay-2 | Checkout is idempotent per org (Idempotency-Key, UNIQUE) | INV-07 |
| pay-3 | The webhook is the source of truth; signature verified before the claim | INV-05/06 |
| pay-4 | Refunds are owner-only, ≤ remaining, idempotent; disputes tracked through to their outcome | F-17 |
| pay-5 | Hosted checkout only; no PAN touches the app | ADR-011 |
| pay-6 | Plan price is resolved server-side from the catalogue; the client never sends an amount | F-17 |
| pwd-1..5 | 800-63B verifier floor (length, blocklist, no composition, no hints, rate limit) | SC-13, SC-5 |

## §5 Features & Key Workflows

### Features

**F-01 Platform foundation.** Covers:
- the monorepo, env schema, logger, error envelope, request id, health, metrics and OpenAPI
- Prisma schema plus migrations (up and down), Redis, the BullMQ worker shell, rate limiter, idempotency middleware, SSE hub
- compose with Caddy TLS, Dockerfiles and CI

*Acceptance:*
- `GET /api/health` returns `{status:"ok", version, commit, checks:{db:"ok", redis:"ok"}}` in under 200 ms.
- A missing env var exits non-zero at boot, naming the variable.
- Every non-2xx response matches §7.
- SIGTERM drains within 10 s.

**F-02 Landing & marketing surface.** Covers:
- the template port (topbar, hero with the `#field` constellation canvas and sigil)
- a gradient `STARDANCER` name, "INTELLIGENCE IN MOTION." eyebrow and the "Every enterprise is a constellation." line
- EXPLORE STARDANCER (→ `/signup`) and VIEW ARCHITECTURE (→ `#architecture`) CTAs
- the OBSERVE→UNDERSTAND→PREDICT→VALIDATE→ACT loop section, a capabilities band, an architecture section, a plans section, footer and the privacy/terms pages

*Acceptance:*
- The purpose is legible in under 30 s: the hero states the problem and the product.
- The canvas respects `prefers-reduced-motion` (static frame).
- LCP ≤ 2.5 s locally.

**F-03 Auth, sessions & profile.** Covers signup (which creates an org with the creator as owner), login, logout, session, CSRF, the password verifier, password change, profile name and admin-issued reset links.

*Acceptance:* SC-3, SC-5, SC-13 and SC-14 hold; a wrong password and an unknown email produce identical 401 bodies.

**F-04 Team & invitations.** Covers the members list, role change, removal (with a last-owner guard), invites with a copyable one-time link, invite acceptance for both new and existing users, and the reset-link issue.

*Acceptance:*
- An invite token works once.
- An expired or revoked token returns 404.
- An admin cannot create or demote an owner.

**F-05 Sources, ingest keys & ingest pipeline.** Covers source CRUD, ingest key issue and revoke, `POST /v1/ingest`, worker processing (entity/relationship upserts, events, vulnerabilities, changes, health samples), the observation log and backpressure.

*Acceptance:*
- A replayed idempotency key is counted as a duplicate.
- An invalid item is rejected per index without failing the batch.
- Graph Freshness p95 ≤ 60 s from ingest to visibility.
- An entity whose attributes change produces a `changes` row with before/after.

**F-06 Simulated environment.** `POST /v1/demo/environment` generates a realistic multi-tier environment through the real ingest pipeline under a `simulated` source:
- sizes small/medium/large ≈ 120/600/2 000 entities
- sites with coordinates, vulnerabilities, events, changes and health samples

`DELETE` removes it.

*Acceptance:* every resulting surface shows a SIMULATED badge, and nothing is generated without an explicit request.

**F-07 Constellation & entity detail.** Covers:
- an SVG graph with server-computed layout positions, a Topology / Geography toggle and filters (type, criticality, environment, stale, search)
- focus and depth, keyboard navigation of nodes (roving tabindex, list fallback) and an accessible table alternative
- entity detail with dependencies (in/out), blast radius, timeline, vulnerabilities, predictions, and criticality/crown-jewel editing

*Acceptance:*
- Selecting an entity shows its relationships.
- The graph stays interactive (pan/zoom ≥ 30 fps) at 2 000 nodes via level of detail.
- Current state is visually distinct from stale/retired entities.

**F-08 Overview & live feed.** Covers:
- the overview dashboard from the reference image: system constellation mini-map with the Ψ core, counters, intelligence feed, system health ring plus four sub-scores, top risks, active investigations, prediction & impact, and the loop flow
- `health-rollup` every 1 min, SSE live updates and the feed screen

*Acceptance:* the overview reads one snapshot (≤ 1 query plus cache) and updates within 5 s of a feed event.

**F-09 Changes & drift.** Covers change detail with impact (blast radius), the drift list (stale entities, unexpected changes) and `drift-sweep` every 5 min.

*Acceptance:*
- An entity not seen for `stale_after_minutes` becomes `stale` within one sweep.
- A change flagged `expected:false` appears as drift.

**F-10 Investigations.** Covers create (optionally seeded from an event, change or prediction), items, notes, the status machine, reconstruction timeline (W7), root-cause ranking (W4) and conclusion.

*Acceptance:* every conclusion links its evidence items, and a resolved investigation records an evidence entry.

**F-11 Attack paths, vulnerability prioritisation & exposure.** Covers:
- bounded path search from external-exposed entities to crown jewels
- contextual vulnerability priority (CVSS × exploitability × reachability × criticality, with reasons)
- the external exposure list

*Acceptance:*
- The search is bounded (depth ≤ 6, frontier ≤ 5 000) and reports `truncated`.
- Each priority shows its reasons.

**F-12 Predictions — Mirror Brain Ψ.** Covers the five deterministic rules, the `mirror-brain` job every 15 min (create/refresh/expire), prediction detail (assumptions, evidence, confidence, uncertainty), review verdicts and accuracy with weight suggestions.

*Acceptance:*
- Every prediction lists ≥ 1 assumption and ≥ 1 evidence ref.
- Predictions are labelled PREDICTION in violet and never shown as fact.

**F-13 Agents & policies.** Covers agent registry, agent keys, capabilities, `max_risk`, suspend; the agent context API; and per-action-type policies (version plus hash).

*Acceptance:* a suspended agent's key returns 401 `KEY_INVALID`, and an agent proposing outside its capabilities gets 403 `CAPABILITY_DENIED`.

**F-14 Governance — proposals, approvals, execution.** Covers proposals from users, agents and the system; the policy check; validation by blast-radius simulation; auto-approval within policy; human approval with separation of duties; the executor; results; and `model_update` for rule weights (W15).

*Acceptance:*
- Unauthorized actions = 0 and governance bypasses = 0: `assertExecutable` re-checks state, approval and policy hash.
- An approved high-risk action needs an admin or owner who is not the proposer.

**F-15 Evidence ledger.** Covers the explorer, record detail with chain links, verify, the `ledger-verify` hourly job, checkpoints and NDJSON export.

*Acceptance:*
- Editing a row out-of-band makes verify return `broken` with the seq.
- A checkpoint HMAC is verifiable with `EVIDENCE_HMAC_KEY`.

**F-16 Trust & continuous compliance.** Covers the NIST CSF 2.0 control catalogue (26 controls that the platform can evaluate from its own data), `compliance-eval` every 15 min, the Trust page (coverage, per-function scores, ledger integrity) and control detail history.

*Acceptance:*
- Each evaluation stores findings.
- Coverage % = evaluated / applicable.

**F-17 Billing.** Covers plans, checkout (Stripe or simulated), the signed webhook, subscription state, payments, refunds (partial and full), disputes, and cancel at period end.

*Acceptance:* pay-1 to pay-6 hold, and a replayed webhook event is a no-op.

**F-18 Operations health correlation.** Covers health history, degraded entities, and correlation of degradation with preceding changes and with upstream dependency health (W12).

*Acceptance:* the correlation ranks changes within the window by lag and dependency distance, with reasons.

### State machines (pre/post conditions)

**Proposal**
```
pending_policy ──(no enabled policy, or the agent lacks the capability, or risk > agent.max_risk)──▶ policy_rejected   [terminal]
pending_policy ──(policy ok)──▶ pending_validation
pending_validation ──(target missing/retired, or blast radius includes ≥1 crown jewel while risk=low, or params invalid)──▶ validation_failed [terminal]
pending_validation ──(ok, risk ≤ policy.auto_approve_max_risk AND reversible)──▶ approved (decision_reason="auto: policy v<n>")
pending_validation ──(ok otherwise)──▶ pending_approval
pending_approval ──(approve by role ≥ policy.approver_min_role (≥admin when risk ∈ {high,critical} or action=model_update), approver ≠ proposer)──▶ approved
pending_approval ──(reject)──▶ rejected [terminal]
approved ──(executor claims, assertExecutable ok)──▶ executing ──▶ succeeded | failed [terminal]
```
- Pre-conditions are re-checked in a transaction with `SELECT … FOR UPDATE` on the proposal row.
- A concurrent second approval sees a non-`pending_approval` state and returns 409 `INVALID_STATE`.
- Each transition appends an evidence record and a feed item.

**Prediction:** `proposed → reviewed | confirmed | refuted`; `reviewed → confirmed | refuted`; `proposed|reviewed → expired` (when `horizon_at` passes, set by `mirror-brain`). `confirmed`, `refuted` and `expired` are terminal. A review of a terminal prediction returns 409.

**Investigation:** `open → in_progress → resolved → closed`. Also `open → resolved` and `resolved → in_progress` (reopen). `closed` is terminal. Resolving requires a non-empty `conclusion`.

**Payment:** `succeeded → partially_refunded → refunded`; `succeeded|partially_refunded → disputed → dispute_won | dispute_lost`; `failed` is terminal. Status is driven only by webhooks.

**Subscription:** `incomplete → active` (checkout.session.completed / invoice.paid); `active → past_due` (invoice.payment_failed); `past_due → active`; `* → canceled` (customer.subscription.deleted).

**Observation:** `received → processed | rejected`. **Entity:** `active ⇄ stale` (drift-sweep, re-observation); `* → retired` (a `removed` change).

### Races, concurrency & idempotency
- Ingest duplicates: UNIQUE(org, source, idempotency_key) with `createMany skipDuplicates`; the duplicate count is returned.
- Concurrent entity upserts on the same `external_id`: `upsert` on UNIQUE(org, external_id); `last_seen_at` uses `GREATEST`.
- Evidence seq: `pg_advisory_xact_lock(hashtext(org_id))`, then `max(seq)+1` inside the same transaction, so there are no gaps or forks.
- Scheduled jobs: `job_claims` INSERT … ON CONFLICT DO NOTHING; a second worker in the same window does nothing.
- Webhook replay: `stripe_events` claim; replays return 200 without reprocessing.
- Proposal approval: row lock plus state check; double-click approve is idempotent (same approver UNIQUE).
- Checkout and refund double-submit: an Idempotency-Key is required, and the UNIQUE constraint returns the original row.

### Key Workflows (1:1 with RESEARCH.md; W16–W17 are the Users & Outcomes workflows A and C)

- **W1 Continuous environment discovery.**
  - W1.1 An admin creates a source on Settings → Sources.
  - W1.2 The admin issues an ingest key (shown once).
  - W1.3 The source posts an observation batch to `/v1/ingest`.
  - W1.4 The worker builds entities and relationships.
  - W1.5 The observation log shows the batch as processed.
  - W1.6 **Terminal:** new entities appear in the Constellation.
- **W2 Security event investigation.**
  - W2.1 The analyst opens a high-severity event from the Intelligence feed.
  - W2.2 Event detail shows the entity, related changes and events.
  - W2.3 The analyst opens an investigation seeded with the event.
  - W2.4 The analyst attaches the entity and adds a note.
  - W2.5 **Terminal:** the investigation is resolved with a conclusion (evidence recorded).
- **W3 Attack-path analysis.**
  - W3.1 The analyst opens Security → Attack paths.
  - W3.2 The analyst selects a crown-jewel target and the maximum depth.
  - W3.3 Paths are ranked with reasons.
  - W3.4 **Terminal:** the analyst opens a path node's entity detail.
- **W4 Root-cause analysis.**
  - W4.1 On an investigation, the analyst opens Root cause.
  - W4.2 Candidates are ranked with reasons and evidence.
  - W4.3 **Terminal:** the analyst sets the root-cause entity on the investigation.
- **W5 Change-impact analysis.**
  - W5.1 The analyst opens a change from the feed.
  - W5.2 The blast radius (affected entities, hops, crown jewels) is shown.
  - W5.3 **Terminal:** the analyst opens an investigation from the change.
- **W6 Drift detection.**
  - W6.1 `drift-sweep` marks entities stale.
  - W6.2 Operations → Drift lists stale entities and unexpected changes.
  - W6.3 **Terminal:** the analyst opens a drifted entity and sees its stale state and last-seen time.
- **W7 Incident reconstruction.**
  - W7.1 On an investigation, the analyst opens Reconstruction.
  - W7.2 The analyst chooses the window.
  - W7.3 A chronological timeline across entities is shown.
  - W7.4 **Terminal:** the analyst adds a timeline item to the investigation.
- **W8 Vulnerability prioritisation.**
  - W8.1 Security → Vulnerabilities is sorted by contextual priority.
  - W8.2 The analyst expands the reasons.
  - W8.3 The analyst filters by KEV.
  - W8.4 **Terminal:** the analyst marks a vulnerability mitigated or accepted.
- **W9 AI-agent context & authorization.**
  - W9.1 An admin registers an agent with capabilities and a maximum risk.
  - W9.2 The admin issues an agent key.
  - W9.3 The agent requests context.
  - W9.4 The agent proposes an action.
  - W9.5 **Terminal:** the proposal appears on Governance with the policy result, and out-of-capability proposals are denied.
- **W10 Automated response.**
  - W10.1 An admin sets a policy allowing auto-approval of low-risk `isolate_endpoint`.
  - W10.2 A low-risk reversible proposal is auto-approved.
  - W10.3 The executor applies it.
  - W10.4 **Terminal:** the entity shows containment `isolated` and the proposal shows `succeeded` with its evidence.
- **W11 Cryptographic trust.**
  - W11.1 Each action appends a chained evidence record.
  - W11.2 The analyst opens Evidence.
  - W11.3 The analyst runs Verify.
  - W11.4 **Terminal:** the result is "verified" with the checked count and latest checkpoint, and the analyst can export NDJSON (admin).
- **W12 Operational health correlation.**
  - W12.1 Operations shows degraded entities.
  - W12.2 The analyst selects one.
  - W12.3 Correlated preceding changes and upstream dependency health are ranked.
  - W12.4 **Terminal:** the analyst opens the top-ranked change.
- **W13 External exposure analysis.**
  - W13.1 Security → Exposure lists externally exposed entities with reachable crown jewels.
  - W13.2 **Terminal:** the analyst opens attack paths for one of them.
- **W14 Continuous compliance.**
  - W14.1 `compliance-eval` evaluates the controls.
  - W14.2 Trust shows coverage and scores.
  - W14.3 The analyst opens a control and sees its findings and history.
  - W14.4 **Terminal:** the analyst re-runs the evaluation and sees a new timestamp.
- **W15 Self-improvement.**
  - W15.1 The analyst confirms or refutes predictions.
  - W15.2 Predictions → Accuracy shows precision per rule with a suggested weight.
  - W15.3 The analyst proposes a `model_update`.
  - W15.4 An admin approves it (not the proposer).
  - W15.5 **Terminal:** the rule weight is updated and the change is recorded in evidence.
- **W16 Understand the environment (Workflow A).**
  - W16.1 A new user signs up.
  - W16.2 They generate the simulated environment.
  - W16.3 The overview shows the graph summary.
  - W16.4 They select an entity in the Constellation.
  - W16.5 Its dependencies are shown.
  - W16.6 **Terminal:** they review its supporting evidence (timeline and evidence records).
- **W17 Evaluate a prediction (Workflow C).**
  - W17.1 Mirror Brain runs.
  - W17.2 The analyst opens a prediction.
  - W17.3 Its assumptions, evidence, confidence and uncertainty are shown.
  - W17.4 **Terminal:** the analyst records a verdict.

Workflow B (Investigate a change) is covered by W5 and W2. Workflow D (Propose an improvement) is covered by W9, W10 and W15.

## §6 UI Surface

| Screen | Route | Example | Roles | Binds | Workflow steps | User controls | States |
|---|---|---|---|---|---|---|---|
| Landing | `/` | `/` | public | — | — | nav links, CTAs, plan cards | success |
| Sign in | `/login` | `/login` | public | `POST /v1/auth/login` | W16.1 | email, password, show-password | idle, submitting, error, success |
| Create account | `/signup` | `/signup` | public | `POST /v1/auth/signup` | W16.1 | name, email, password, org name | idle, submitting, field errors, success |
| Accept invitation | `/invite/[token]` | `/invite/demo-invite-token` | public | `GET /v1/invites/:token`, `POST /v1/invites/:token/accept` | — | name, password | loading, invalid/expired, submitting, success |
| Reset password | `/reset/[token]` | `/reset/demo-reset-token` | public | `GET /v1/auth/reset/:token`, `POST /v1/auth/reset/:token` | — | new password | loading, invalid, submitting, success |
| Privacy | `/privacy` | `/privacy` | public | — | — | — | success |
| Terms | `/terms` | `/terms` | public | — | — | — | success |
| Overview | `/overview` | `/overview` | all | `GET /v1/overview`, `GET /v1/feed`, `GET /v1/stream`, `GET /v1/auth/session`, `POST /v1/auth/logout`, `POST /v1/demo/environment` | W16.2, W16.3 | demo size select, generate demo, feed live toggle | empty (no telemetry → onboarding), loading, error, success, simulated |
| Constellation | `/constellation` | `/constellation` | all | `GET /v1/graph`, `GET /v1/entities` | W1.6, W16.4 | view toggle topology/geo, type, criticality, environment, include stale, search, focus depth, table view | empty, loading, error, success, truncated |
| Entity detail | `/constellation/[entityId]` | `/constellation/00000000-0000-4000-8000-000000000101` | all | `GET /v1/entities/:id`, `PATCH /v1/entities/:id`, `GET /v1/entities/:id/neighbors`, `GET /v1/entities/:id/impact`, `GET /v1/entities/:id/timeline` | W3.4, W6.3, W16.5, W16.6 | neighbor depth, direction, impact depth, timeline range, criticality, crown jewel, owner, tags | loading, not found, error, success |
| Intelligence | `/intelligence` | `/intelligence` | all | `GET /v1/feed`, `GET /v1/events`, `GET /v1/changes` | W2.1, W5.1 | tab feed/events/changes, kind, severity, drift only, from/to | empty, loading, error, success |
| Event detail | `/intelligence/events/[eventId]` | `/intelligence/events/00000000-0000-4000-8000-000000000301` | all | `GET /v1/events/:id`, `POST /v1/investigations` | W2.2, W2.3 | open investigation (title, severity) | loading, not found, success |
| Change detail | `/intelligence/changes/[changeId]` | `/intelligence/changes/00000000-0000-4000-8000-000000000401` | all | `GET /v1/changes/:id`, `POST /v1/investigations` | W5.2, W5.3, W12.4 | open investigation | loading, not found, success |
| Investigations | `/investigations` | `/investigations` | all | `GET /v1/investigations`, `POST /v1/investigations` | W2.3 | status, severity filters, new investigation form | empty, loading, error, success |
| Investigation detail | `/investigations/[investigationId]` | `/investigations/00000000-0000-4000-8000-000000000501` | all | `GET /v1/investigations/:id`, `PATCH /v1/investigations/:id`, `POST /v1/investigations/:id/items`, `POST /v1/investigations/:id/notes`, `GET /v1/investigations/:id/reconstruction`, `GET /v1/investigations/:id/root-cause`, `GET /v1/members` | W2.4, W2.5, W4.1–W4.3, W7.1–W7.4 | status, assignee, conclusion, root cause pick, note body, window hours, add item | loading, not found, error, success |
| Security | `/security` | `/security` | all | `GET /v1/vulnerabilities`, `PATCH /v1/vulnerabilities/:id`, `GET /v1/attack-paths`, `GET /v1/exposure` | W3.1–W3.3, W8.1–W8.4, W13.1, W13.2 | tab vulns/paths/exposure, status, KEV, min score, target select, max depth, path limit | empty, loading, error, success, truncated |
| Operations | `/operations` | `/operations` | all | `GET /v1/operations/health`, `GET /v1/operations/correlations`, `GET /v1/drift` | W6.2, W12.1–W12.3 | hours range, entity select, correlation window, tab health/drift | empty, loading, error, success |
| Trust | `/trust` | `/trust` | all | `GET /v1/compliance/controls`, `POST /v1/compliance/evaluate`, `GET /v1/evidence/checkpoints` | W14.2, W14.4 | function filter, re-evaluate | empty, loading, error, success |
| Control detail | `/trust/controls/[controlId]` | `/trust/controls/00000000-0000-4000-8000-000000000601` | all | `GET /v1/compliance/controls/:id` | W14.3 | — | loading, not found, success |
| Predictions | `/predictions` | `/predictions` | all | `GET /v1/predictions`, `POST /v1/predictions/run`, `GET /v1/predictions/accuracy`, `POST /v1/proposals` | W15.2, W15.3, W17.1 | status, rule filters, run now, tab list/accuracy, propose weight | empty, loading, error, success |
| Prediction detail | `/predictions/[predictionId]` | `/predictions/00000000-0000-4000-8000-000000000701` | all | `GET /v1/predictions/:id`, `POST /v1/predictions/:id/review` | W15.1, W17.2–W17.4 | verdict, note | loading, not found, success, terminal |
| Agents | `/agents` | `/agents` | all | `GET /v1/agents`, `POST /v1/agents` | W9.1 | register form (name, description, capabilities, max risk) | empty, loading, error, success |
| Agent detail | `/agents/[agentId]` | `/agents/00000000-0000-4000-8000-000000000801` | all | `GET /v1/agents/:id`, `PATCH /v1/agents/:id`, `POST /v1/agents/:id/keys`, `DELETE /v1/api-keys/:id` | W9.2 | capabilities, max risk, suspend/activate, issue key name, revoke | loading, not found, success, key-shown-once |
| Evidence | `/evidence` | `/evidence` | all | `GET /v1/evidence`, `POST /v1/evidence/verify`, `GET /v1/evidence/export` | W11.2–W11.4 | kind, subject filters, verify, export range | empty, loading, error, success, broken |
| Evidence record | `/evidence/[seq]` | `/evidence/1` | all | `GET /v1/evidence/:seq` | W11.1, W16.6 | prev/next | loading, not found, success |
| Governance | `/governance` | `/governance` | all | `GET /v1/proposals`, `POST /v1/proposals`, `GET /v1/policies`, `PUT /v1/policies/:actionType` | W9.5, W10.1 | status, risk filters, new proposal form (action, target, params, rationale), policy edit (auto max risk, approver role, enabled) | empty, loading, error, success |
| Proposal detail | `/governance/proposals/[proposalId]` | `/governance/proposals/00000000-0000-4000-8000-000000000901` | all | `GET /v1/proposals/:id`, `POST /v1/proposals/:id/approve`, `POST /v1/proposals/:id/reject` | W10.2–W10.4, W15.4, W15.5 | approve, reject reason | loading, not found, success, terminal |
| Settings — organization | `/settings` | `/settings` | all (edit owner) | `GET /v1/org`, `PATCH /v1/org`, `DELETE /v1/demo/environment` | — | org name, stale-after minutes, remove simulated data | loading, error, success |
| Settings — members | `/settings/members` | `/settings/members` | all (edit admin) | `GET /v1/members`, `PATCH /v1/members/:id`, `DELETE /v1/members/:id`, `POST /v1/members/:id/reset-link`, `GET /v1/invites`, `POST /v1/invites`, `DELETE /v1/invites/:id` | — | invite email + role, role select, remove, reset link, revoke invite | empty, loading, error, success, link-shown-once |
| Settings — sources | `/settings/sources` | `/settings/sources` | all (edit admin) | `GET /v1/sources`, `POST /v1/sources`, `PATCH /v1/sources/:id`, `GET /v1/api-keys`, `POST /v1/sources/:id/keys`, `GET /v1/observations` | W1.1, W1.2, W1.5 | source name, kind, pause, key name, revoke, observation status filter | empty, loading, error, success, key-shown-once |
| Settings — billing | `/settings/billing` | `/settings/billing` | owner | `GET /v1/billing`, `POST /v1/billing/checkout`, `GET /v1/billing/payments`, `POST /v1/billing/refunds`, `POST /v1/billing/cancel` | — | plan select, refund amount + reason, cancel | loading, error, success, simulated mode |
| Simulated checkout | `/settings/billing/checkout/[checkoutId]` | `/settings/billing/checkout/00000000-0000-4000-8000-000000000a01` | owner | `GET /v1/billing/checkout/:id`, `POST /v1/billing/checkout/:id/complete` | — | pay (success), decline | loading, not found, processing, success, declined |
| Settings — profile | `/settings/profile` | `/settings/profile` | all | `GET /v1/auth/session`, `PATCH /v1/auth/profile`, `POST /v1/auth/password` | — | name, current password, new password | idle, submitting, error, success |

App routes (everything except the public ones) require a session. With none, they redirect to `/login?next=`. App screens send `noindex`.

### Design System

- **Tokens:** the DESIGN-TEMPLATE.html `:root` is copied verbatim into `web/src/styles/tokens.css` (ADR-003), plus derived tokens for spacing (`--s1..--s8`, 4 px unit), type steps (`--fs-xs…--fs-3xl` = 9.7/11.7/14/16.8/20.2/24.2/29 px plus display 44/64 px), radius (`--r` 14 px for panels, `--r-sm` 6 px, `--r-xs` 2 px for dense controls), motion (`--t-fast/med/slow`, `cubic-bezier(.2,.7,.2,1)`) and focus (2 px `--amber` ring, 2 px offset).
- **Type:** Manrope 600/700 for display, headings, metrics and the wordmark. Inter 400/500 for UI. JetBrains Mono for ids, hashes, counters, timestamps and eyebrows (uppercase, letter-spacing .14em, `--muted`).
- **Colour semantics:**
  - amber→ember gradient: primary CTA (one per view)
  - cyan: relationships, telemetry, links
  - violet: Mirror Brain and predictions (always with the PREDICTION label)
  - green: healthy and pass
  - amber: warning
  - red: critical, fail and integrity break
  - `--muted` / `--dim`: secondary text
  - SIMULATED badge: dashed `--line2` border plus amber text
- **Components** (`web/src/components/ui`): Button (primary via `data-primary`, secondary, ghost, danger; ≥44 px touch target), IconButton, Input, Select, Textarea, Field (label plus hint plus error, `aria-describedby`), Badge (severity, status, SIMULATED, PREDICTION), Panel (template `.panel`: `--panel` fill, `--line` border, `--r` radius, backdrop blur), Stat, Table (sticky header, `role=region` scroll wrapper with `tabindex=0` and `aria-label`), Tabs (roving focus), Drawer (focus trap, Esc), Dialog, Toast (`aria-live=polite`), Skeleton, EmptyState (headline plus why plus primary action), ErrorState (problem title plus requestId plus retry), HealthRing (SVG), Sparkline, ConstellationCanvas (SVG plus keyboard list), KeyReveal (copy-once secret).
- **Layout:** the app shell is a 224 px sidebar (11 destinations with mono icons, active state in amber), a topbar (org name, search, live indicator, user menu) and content up to 1440 px. Below `md` (768 px) the sidebar becomes a drawer: toggle with `aria-expanded` and `aria-controls`, focus trap. Grid: 12 columns at ≥1024, 6 at ≥640, 1 below.
- **Motion:** 140–320 ms, transform and opacity only. The constellation drift and pulses are disabled under `prefers-reduced-motion`, and the landing canvas renders one static frame.
- **States:** every data view has designed empty, loading (skeleton matching the layout), error (§7 title, requestId and retry) and success states. Simulated data shows the badge.
- **Accessibility:** WCAG 2.2 AA. Skip link to `#main`, visible focus, contrast ≥ 4.5:1 for text (`--muted` on `--bg` = 6.0:1), graph data available as a table, forms with programmatic labels and inline errors.

## §7 Error contract

Every non-2xx response is `application/problem+json`:

```json
{ "type": "https://stardancer.local/errors/validation", "title": "Validation failed", "status": 400,
  "detail": "2 fields failed validation", "instance": "/api/v1/investigations",
  "code": "VALIDATION_FAILED", "requestId": "0f6c…",
  "errors": [ { "field": "title", "message": "Title must be at least 3 characters", "code": "TOO_SHORT" } ] }
```

| code | status | When | Client recovery |
|---|---|---|---|
| `VALIDATION_FAILED` | 400 | zod failure; `errors[]` per field | attach each message to its control |
| `MALFORMED_JSON` | 400 | unparsable body | fix the client |
| `UNAUTHENTICATED` | 401 | no or expired session | redirect to `/login?next=` |
| `INVALID_CREDENTIALS` | 401 | bad email/password (identical for unknown email) | show the form error |
| `KEY_INVALID` | 401 | missing, revoked or suspended API key | rotate the key |
| `CSRF_FAILED` | 403 | CSRF token or Origin mismatch | refetch the session and retry once |
| `FORBIDDEN` / `ROLE_REQUIRED` | 403 | role below the requirement on an own-tenant resource | show "ask an admin" |
| `SELF_APPROVAL` | 403 | an approver who is also the proposer | explain separation of duties |
| `CAPABILITY_DENIED` | 403 | agent action outside capabilities | — |
| `NOT_FOUND` | 404 | missing **or other tenant's** resource | not-found state |
| `CONFLICT` / `INVALID_STATE` / `INVALID_TRANSITION` / `LAST_OWNER` / `DUPLICATE` | 409 | state machine or uniqueness | refetch and show the reason |
| `IDEMPOTENCY_MISMATCH` | 422 | same key, different body | generate a new key |
| `REFUND_EXCEEDS_REMAINING` | 422 | refund > remaining | show the maximum |
| `PAYLOAD_TOO_LARGE` | 413 | body over the limit | split the batch |
| `RATE_LIMITED` | 429 | limiter; `Retry-After` header | back off |
| `BACKPRESSURE` | 429 | ingest queue too deep; `Retry-After` | back off |
| `SERVER_ERROR` | 500 | unexpected (no detail beyond requestId) | retry and report the requestId |
| `UNAVAILABLE` | 503 | dependency down or shutting down | retry |

## §8 Configuration & environment

| Variable | Purpose | Req | Service |
|---|---|---|---|
| `NODE_ENV` | `production`/`development`/`test` | req | all |
| `APP_VERSION` / `GIT_COMMIT` | reported by `/api/health` | opt | api |
| `PUBLIC_ORIGIN` | e.g. `https://localhost:8643`; used for the Origin check, links and CSP | req | api, web |
| `PORT` | api 4000, web 3000 | opt | api, web |
| `DATABASE_URL` | Postgres/Neon pooled URL | req | api, worker |
| `DIRECT_URL` | non-pooled URL for migrations | opt | migrate |
| `DATABASE_REPLICA_URL` | read replica (falls back to primary) | opt | api |
| `REDIS_URL` | queues, pub/sub, limits, cache | req | api, worker |
| `SESSION_SECRET` | ≥32 bytes; pepper for token hashing | req | api |
| `EVIDENCE_HMAC_KEY` | ≥32 bytes; checkpoint HMAC | req | api, worker |
| `BILLING_MODE` | `simulated` or `stripe` | req | api |
| `STRIPE_SECRET_KEY` | required when `BILLING_MODE=stripe` | cond | api |
| `STRIPE_WEBHOOK_SECRET` | webhook signing secret (`whsec_…`); simulated mode signs with it | req | api |
| `STRIPE_PRICE_CONSTELLATION` / `STRIPE_PRICE_SOVEREIGN` | Stripe price ids in stripe mode | cond | api |
| `HSTS_ENABLED` | `true` (local CA trusted per QUICKSTART; 1-day max-age on loopback, ADR-065) | opt | edge, web |
| `LOG_LEVEL` | pino level (default `info`) | opt | all |
| `INGEST_MAX_DEPTH` | backpressure threshold (default 10 000) | opt | api |
| `API_INTERNAL_URL` | web→api URL used by server components (`http://api:4000`) | req | web |
| `HTTP_PORT` / `HTTPS_PORT` | host ports for Caddy (8280/8643) | opt | compose |
| `PROMETHEUS_PORT` / `GRAFANA_PORT` | 9390/3390 on 127.0.0.1 | opt | compose |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin | req | grafana |
| `POSTGRES_PASSWORD` | local Postgres | req | postgres |

## §9 Performance targets

These derive from RESEARCH Success Metrics, PBR-3/4 and the large tier. They are measured by the load test and gates, and never claimed until measured.

| Operation | Target (p95 unless noted) |
|---|---|
| `GET /api/health` | ≤ 50 ms |
| `GET /v1/overview` (snapshot + cache) | ≤ 150 ms |
| List endpoints (`events`, `feed`, `entities`, `proposals`, `evidence`) | ≤ 200 ms at limit 25 |
| `GET /v1/entities/:id`, `/neighbors` depth ≤ 2 | ≤ 200 ms |
| `GET /v1/graph` (≤ 2 000 nodes, cached) | ≤ 400 ms; payload ≤ 400 KB gzip |
| `GET /v1/entities/:id/impact` depth 6, `GET /v1/attack-paths` | ≤ 800 ms, bounded frontier |
| `POST /v1/ingest` (500 items) | ≤ 300 ms to 202 |
| Graph freshness (ingest → visible) | ≤ 60 s p95 (PBR-3) |
| SSE fan-out (event → client) | ≤ 5 s |
| Proposal policy + validation | ≤ 2 s from submit to decision state |
| Login | ≤ 500 ms (argon2 cost dominates) |
| Web: LCP ≤ 2.5 s, CLS ≤ 0.1, INP ≤ 200 ms; JS per route ≤ 400 KB | |
| Availability | 99.9 % SLO; RTO ≤ 1 h, RPO ≤ 5 min (Neon PITR) |
| Error rate under load | < 1 % non-429 errors |
| Concurrency | design target 50k+ concurrent (horizontal); single-host load test reports measured ceiling (ADR-016) |

## §10 Test plan

| Scope | Unit (Vitest) | Integration (Vitest + real Postgres/Redis via compose) | E2E (Playwright, prod build) |
|---|---|---|---|
| F-01 | env schema, problem envelope, cursor codec | health, 404 envelope, idempotency replay, rate limit | smoke |
| F-02 | — | — | landing renders, CTAs route, reduced motion |
| F-03 | password verifier, session token | signup/login/logout, CSRF, enumeration parity, lockout | W16 |
| F-04 | — | invite accept once, last-owner guard | members flow |
| F-05 | ingest schemas | ingest dedupe, rejection per index, worker processing, backpressure | W1 |
| F-06 | generator determinism | demo generate/remove | W16 |
| F-07 | layout, graph filter | graph, neighbors, impact bounds, tenant 404 | W16 |
| F-08 | health scoring | health-rollup effect, overview | W16 |
| F-09 | drift rules | drift-sweep effect, change impact | W5, W6 |
| F-10 | root-cause scoring, transitions | investigation lifecycle, reconstruction | W2, W4, W7 |
| F-11 | path search, priority | attack paths truncation, exposure | W3, W8, W13 |
| F-12 | each rule | mirror-brain effect + expiry, review transitions | W17, W15 |
| F-13 | capability check | agent key auth, context, suspended agent | W9 |
| F-14 | state machine | auto-approve, SoD, concurrent approve, executor gate | W10, W15 |
| F-15 | hash chain | append, verify, tamper detection, ledger-verify effect | W11 |
| F-16 | control rules | compliance-eval effect | W14 |
| F-17 | price catalogue | checkout idempotency, webhook verify→claim, replay no-op, refund bounds, dispute | billing flow |
| F-18 | correlation scoring | correlations | W12 |

Each `W1`–`W17` has a Playwright `test('W<n> …')` in `e2e/` running on Desktop Chrome and Pixel 7. The functional smoke test (`smoke-test.sh`) runs against `https://localhost:8643`. The load test (`load/`, autocannon) follows BUILD Load Test Methodology.
