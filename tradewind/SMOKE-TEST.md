# SMOKE-TEST.md — Tradewind

Manual verification checklist. The automated equivalent is `smoke-test.sh` (run `bash smoke-test.sh`
against a running stack; results append to `smoke-test.log`).

## Prerequisites
- Stack running (`docker compose up` or local processes); API on http://localhost:4000, web on :3000.
- DB migrated + seeded (demo users present).

## Checklist

| # | Check | Expected |
|---|-------|----------|
| 1 | `GET /v1/health` | 200, `services.database = "up"` |
| 2 | Landing page loads in a browser | hero + Sign in / Get started |
| 3 | Register a client + a freelancer | "Account created"; can sign in |
| 4 | Anti-enumeration: register an existing email | identical response shape (no "email taken" leak) |
| 5 | Visit `/ledger` while signed out | redirected to `/login` |
| 6 | Client: create project → add milestone ($100) | milestone shows `PENDING_ACCEPTANCE` |
| 7 | Invalid milestone amount ($0.05) | field-level error on the amount input (not a generic message) |
| 8 | Freelancer: complete Connect onboarding → accept milestone | milestone `ACCEPTED` |
| 9 | Client: Fund via Stripe Checkout | redirect → funding/success → milestone `FUNDED` (confirmed by webhook, not the redirect) |
| 10 | Webhook with a bad signature (`curl` with junk `stripe-signature`) | 400 (verified before any processing) |
| 11 | Freelancer: mark delivered; Client: Approve & release | milestone `RELEASED` |
| 12 | Freelancer balance | `$90.00` (net of 10% fee) |
| 13 | Freelancer: request a $90 payout; then request another | first `REQUESTED`→`PAID`; second `422 INSUFFICIENT_FUNDS` |
| 14 | A different client opens the milestone URL + its `/ledger` | `404` on both (same-tenant IDOR closed) |
| 15 | Admin: Ledger → reconcile | **Balanced ✓**, system total `0` |
| 16 | Ledger → Export CSV / PDF | downloads scoped to the signed-in user |
| 17 | Dispute: client opens a dispute on a funded/delivered milestone; admin refunds/releases/splits | milestone moves to `REFUNDED`/`RELEASED`/`SPLIT`; ledger still balances |

## Capability generalization
The ledger/reconciliation engine operates on arbitrary amounts and account combinations (not curated
fixtures): create milestones of varying amounts and resolve disputes with arbitrary split percentages —
every transaction's legs sum to zero and the closed-system total stays `0` (verify via `/v1/ledger/reconcile`).

## Error-scenario checks (comprehensive depth)
- Invalid input → field-level message on the offending control (T7).
- Expired/absent session → 401 on protected routes (T5).
- Missing required env var → API fails fast at boot with which var (see config/env.ts).
- DB unreachable → `/v1/health` returns 503 (contract-correct).
