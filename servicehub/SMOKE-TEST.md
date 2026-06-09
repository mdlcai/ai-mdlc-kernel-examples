# SMOKE-TEST.md — ServiceHub

Functional smoke — every SPEC §5 Key Workflow exercised end-to-end against the
running stack at the API contract level. Script: `scripts/smoke-test.mjs` (run via
`npm run smoke`); full request/response evidence is written to `smoke-test.log`.
Target defaults to the deployed stack (`http://localhost:18080`).

## Checklist
- [x] Health: `/api/health` 200 — `db: up`, `redis: up`
- [x] Auth: requester login issues token + cookie; staff login resolves agent/manager; `/me` role correct
- [x] Anti-enumeration: unknown-user and bad-password both return an identical `401 auth.invalid_credentials`
- [x] W1 Requester self-service: create ticket (SLA due set) → list **own** → detail → comment → upload + download own attachment
- [x] **Object-level authz (W1):** a non-owner downloading another user's attachment → **404**
- [x] W2 Agent: work queue → assign → add **internal note** → resolve
- [x] **Internal-note confidentiality (W2):** the internal staff note is **hidden from the requester**
- [x] **Authorization isolation (W3):** same-tenant cross-user ticket read → **404**; its sub-resource (comments) → **404**; cross-tenant read → **404**
- [x] W4 Manager: SLA dashboard (open / breached / MTTR); report export start → ready → download
- [x] **RBAC (W4):** a requester hitting the manager dashboard → **403**
- [x] Webhook: invalid signature → **401** (verify-before-process)
- [x] Large-header resilience: 16 KB cookie on health + a deep route → 200 / 200 (not 431)

## Key Workflows (SPEC §5 — all passing)
- [x] W1 Submit & track an IT request — requester, **own tickets only**
- [x] W2 Triage → assign → internal note → resolve — agent
- [x] W3 Authorization isolation — same-tenant cross-user **and** cross-tenant
- [x] W4 Manager dashboard, SLA tracking, and report export

Last run: **27 / 27 functional smoke checks passed** against the deployed stack (`http://localhost:18080`), 0 failed. Full request/response evidence in `smoke-test.log`.
