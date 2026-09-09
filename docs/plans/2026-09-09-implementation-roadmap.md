# Pikol — Implementation Roadmap

**Date:** 2026-09-09
**Spec:** [`2026-09-09-court-booking-system-design.md`](./2026-09-09-court-booking-system-design.md)

The spec covers nine independently shippable subsystems. Each gets its own
plan, because each must produce working, testable software on its own — a
single plan at TDD granularity would be several thousand lines and unusable.

## Phase order and dependencies

```
1. Backend foundation ──┬─→ 2. Identity & auth ──┬─→ 3. Venues, courts, pricing
                        │                        │
                        │                        └─→ 6. Frontend foundation & auth
                        │
                        └─────────────────────────→ (blocks everything)

3. ──→ 4. Booking core ──→ 5. Payments ──→ 7. Frontend player flow
                       └─────────────────→ 8. Frontend staff console

5 + 7 + 8 ──→ 9. Deployment
```

| # | Phase | Deliverable — what works when it is done | Spec |
|---|---|---|---|
| 1 | **Backend foundation** | API boots, `/health` responds, Postgres connected, Alembic migrating, `parameters` seeded and typed, CI green | §3.3–3.5, §4.5 |
| 2 | **Identity & auth** | Register → verify email → log in → refresh → reset password. Staff invitations. `VenueScope` enforced. | §6 |
| 3 | **Venues, courts, pricing** | Owner creates venues and courts, sets wildcard price rules; resolution returns the right price for any hour | §4.4, §7.4 |
| 4 | **Booking core** | Availability grid query, holds with lazy expiry, walk-ins, closures, the full state machine — with the mock provider | §4.1–4.3, §5.1–5.3, §7.2 |
| 5 | **Payments** | Provider port, `MockProvider`, PayMongo adapter, idempotent webhooks, resolution queue, receipt email | §5.4–5.6, §7.5–7.6 |
| 6 | **Frontend foundation & auth** | Next.js scaffold, API client with refresh rotation, all `(auth)` screens | §8 |
| 7 | **Frontend player flow** | Availability grid, consecutive-hour selection, checkout, hold countdown, polling return page, My Bookings | §8.1–8.2 |
| 8 | **Frontend staff console** | Today's schedule, walk-in booking, mark-paid, outcomes, courts/pricing/closures, staff invites, parameters | §8 |
| 9 | **Deployment** | Supabase + Render + Vercel live, keep-alive, CI/CD, PayMongo test → live | §9 |

## Why this order

**Phase 4 depends on 3, not the reverse.** Bookings need courts and prices to
exist before availability means anything.

**Phase 4 ships before payments** and is fully testable, because the provider
port (§5.6) lets `MockProvider` stand in. This is the payoff of the
mock-first decision: the entire booking state machine — including hold expiry
and the webhook confirmation path — is exercised with no PayMongo account and
no internet.

**Phase 9 is last but starts early.** PayMongo live mode needs a registered PH
business (DTI/SEC + BIR), which is the long pole and is not shortened by any
code. Begin that registration during Phase 1.

## Repository state

| Repo | Status |
|---|---|
| `pikol-parent` | Exists. Docs and plans only; no code. |
| `pikol-backend` | Created in Phase 1, Task 1. |
| `pikol-frontend` | Created in Phase 6, Task 1. |

Plans live here in the parent, for every repo's features — the asima
convention. Working files (`tasks/plan.md`, `tasks/todo.md`) stay gitignored
in whichever repo is being worked on.
