# Phase 4 — Booking Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Players see a day's availability grid and place 15-minute holds;
staff book walk-ins, record cash, set outcomes, cancel, and close courts —
all through one table pair protected by one partial unique index.

**Architecture:** `bookings` is a rich module (spec §3.4). `domain/` is pure
Python: enums, a `BookingSnapshot`, every transition guard, window checks,
reference codes, and the derived display status — unit-tested with no
database. `persistence/` owns `booking_groups` + `booking_slots` and **one
method that writes every status** (`BookingRepository.set_status`), so the
§4.2 mirror invariant has exactly one implementation. Services orchestrate
one transaction per request; routers are thin.

**Tech Stack:** as Phase 3. No new dependencies.

**Spec:** [`2026-09-09-court-booking-system-design.md`](./2026-09-09-court-booking-system-design.md)
§4.1–4.3, §5.1–5.3, §5.7, §6.5, §7.2–7.4. Schema:
[`../adr/pikol.dbml`](../adr/pikol.dbml) `booking_groups`, `booking_slots`,
Appendix A §2, §3, §5–8. [ADR 0001](../adr/0001-booking-slot-uniqueness.md).

## Global Constraints

Everything in Phases 2–3, plus:

- `uq_court_slot_active ON booking_slots (court_id, starts_at) WHERE booking_status NOT IN ('CANCELLED','EXPIRED')` — Appendix A §2 verbatim. It is the only double-booking guard; there is no `SELECT FOR UPDATE` anywhere.
- **Every read of `PENDING` adds `expires_at > now()`.** Availability, the hold cap, and the derived display status all obey it.
- `booking_slots.booking_status` and `court_id` mirror the group, written only by `BookingRepository.set_status` / `insert`, in the same transaction.
- Prices are frozen at creation: `booking_slots.price_centavos` per hour, `booking_groups.price/fee/total_centavos`. Nothing recomputes them.
- Day boundaries via `manila_day_bounds`. Every slot is on the hour (Appendix A §3 CHECK).
- `IN_PROGRESS` is derived, never stored. Cancellation always records `cancelled_at`, `cancelled_by_user_id`, and `cancellation_reason` (Appendix A §7).

## Decisions this plan makes

| Decision | Why |
|---|---|
| A request names `{court_id, starts_at, hours}`, not a list of slots | Consecutive by construction — a whole class of validation disappears |
| `hours` is 1–12 per booking; a closure spans ≤ 31 days per request | Bounds; the spec is silent and an unbounded range is a 10,000-row insert |
| **Checkout, the provider port, and the webhook are Phase 5.** Phase 4 ships the domain transition `confirm_online_payment` with unit tests | The roadmap lists the port under both phases; drawing the line here keeps Phase 4 free of payment plumbing. A hold in Phase 4 simply expires |
| Grid status `PAST` = starts before `now + min_lead_minutes` | A slot a player can no longer book must not render as bookable; staff still walk-in into it |
| Mark-paid is allowed from `CONFIRMED` **or** `COMPLETED` (walk-in, `UNPAID`) | People pay after they play; staff mark the outcome first |
| A closure is `CONFIRMED/UNPAID`, zero money, one group per court; its reason goes in `notes` | ADR 0001; cancelled through its own route with `CLOSURE:Manage`, never through booking routes |
| Player cancellation reason is optional (defaults to "Cancelled by player") | The CHECK needs one; players should not have to type it |
| Display status: `PENDING` with a lapsed `expires_at` reads `EXPIRED`; `CONFIRMED` during its hours reads `IN_PROGRESS` | The PENDING trap applied to every response, not just queries |
| `GET /admin/bookings/unfulfilled` is Phase 5 | `needs_resolution` is only ever set by a late payment |
| Domain errors are `BookingError(code, detail, kind)`; `kind` ∈ `invalid` (→ 422) / `conflict` (→ 409) | The domain stays HTTP-free; one mapping in the service layer |

## Review Focus

1. **Lapsed hold, uncontested** — an abandoned hold must not block the grid, must not count toward the cap, and must be expired the moment someone books its slot. Tests in Task 3.
2. **Mirror drift** — after every transition (expire, cancel, outcome) every slot row carries the group's status. Test in Task 2 (`set_status`) and asserted again in Tasks 3–5.
3. **Partial index predicate** — a `COMPLETED` or `NO_SHOW` slot still blocks a new booking; a `CANCELLED` one frees it. Test in Task 2.
4. **12am Manila** — `GET /availability?date=` returns the 00:00–01:00 Manila slot as the first row of that date. Test in Task 3.
5. **Cross-venue staff** — staff at Venue A reading, cancelling, or walking-in at Venue B get 404. Test in Task 5.

---

### Task 1: The booking domain (pure)

**Files:** Create `src/bookings/domain/{enums,errors,rules}.py`. Test `tests/bookings/test_booking_domain.py`.

**Produces:**
- `BookingType`, `BookingStatus`, `PaymentStatus`, `PaymentMethod (ONLINE, CASH, MANUAL_GCASH)` — `StrEnum`s; `DisplayStatus` = `BookingStatus` values + `IN_PROGRESS`. `RELEASING = {CANCELLED, EXPIRED}`.
- `BookingError(code: str, detail: str, kind: Literal["invalid", "conflict"])`.
- `slot_starts(starts_at, hours) -> list[datetime]`; `check_on_the_hour(starts_at)` (`invalid`, `NOT_ON_THE_HOUR`; also rejects naive).
- `check_player_window(starts_at, hours, now, *, min_lead_minutes, horizon_days)` — first slot ≥ `now + min_lead` (`TOO_SOON`), last slot ≤ `now + horizon` (`BEYOND_HORIZON`).
- `check_walk_in_window(starts_at, now, *, backdate_days)` — first slot ≥ `now − backdate_days` (`TOO_FAR_BACK`).
- `@dataclass(frozen=True) BookingSnapshot(type, booking_status, payment_status, starts_at, hours, expires_at)`.
- Guards, each returning nothing or raising `conflict`: `can_cancel_by_player(b, now, *, cutoff_minutes)` (PENDING unexpired, or CONFIRMED before `starts_at − cutoff`; not MAINTENANCE); `can_cancel_by_staff(b)` (CONFIRMED, not MAINTENANCE); `can_cancel_closure(b)` (MAINTENANCE, CONFIRMED); `can_mark_paid(b)` (WALK_IN, UNPAID, CONFIRMED or COMPLETED); `can_set_outcome(b, now)` (CONFIRMED, not MAINTENANCE, `starts_at ≤ now`); `can_confirm_online_payment(b, now)` (PLAYER_BOOKING, PENDING, unexpired).
- `display_status(b, now) -> DisplayStatus`.
- `new_reference_code() -> str` — `PKL-` + 5 chars from `ABCDEFGHJKMNPQRSTVWXYZ23456789`.

- [ ] Tests for every guard's allow and deny cases with an explicit `now`; `display_status` for lapsed PENDING → `EXPIRED`, CONFIRMED mid-slot → `IN_PROGRESS`, CONFIRMED after the last hour → `CONFIRMED`; reference code shape. Fail → implement → gate → commit.

### Task 2: Schema and the single status writer

**Files:** Create `src/bookings/persistence/{models,repository}.py`, migration `0004_bookings.py`; modify `env.py`. Test `tests/bookings/test_booking_schema.py`.

**Produces:** `BookingGroupModel`, `BookingSlotModel` per dbml (enums `booking_type`, `booking_status`, `payment_status`, `payment_method`; FKs per `Ref:`; indexes per dbml). Appendix A §2, §3, §5 (slot + group), §6, §7, §8 verbatim, also declared on the models (`alembic check` clean). `BookingRepository`:
- `insert(session, group, slot_prices: list[tuple[datetime, int]]) -> BookingGroupModel` — inside a savepoint; `uq_court_slot_active` → `AppError(409, "SLOT_UNAVAILABLE")`; a reference-code collision retries with a fresh code (≤ 5 tries).
- `set_status(session, group, status, **group_fields)` — updates the group and **all** its slots in one statement pair. The only writer of `booking_status`.
- `expire_lapsed(session, court_id, starts_ats, now)` — the §4.3 CTE, both tables.

- [ ] **Tests:** `test_slot_blocked_by_active_booking` (raw second insert → `IntegrityError` on `uq_court_slot_active`); `test_cancelled_slot_frees_it`; `test_completed_and_no_show_still_block`; `test_slot_must_be_on_the_hour`; `test_pending_requires_expiry_and_only_pending_has_one`; `test_cancellation_must_be_documented`; `test_only_player_booking_has_player`; `test_set_status_mirrors_every_slot`; `test_expire_lapsed_releases_all_slots_of_the_group`. Up/down/up; `alembic check`; commit.

### Task 3: Holds and the availability grid

**Files:** Create `src/bookings/{dto,service}.py`, `src/bookings/controllers/{player,availability}.py`. Test `tests/bookings/test_holds.py`, `tests/bookings/test_availability.py`.

**Interfaces:**
- `POST /bookings {court_id, starts_at, hours}` (any active user with role PLAYER — staff use walk-in) → 201 `BookingOut`. One transaction, in §5.3 order: court active at an active venue (else 404) → window checks → unexpired-hold count vs `max_active_holds_per_player` (`409 HOLD_LIMIT`) → `PriceResolver` per slot, fee via `convenience_fee_centavos` → `expire_lapsed` → `insert` (`PENDING/UNPAID`, `expires_at = now + hold_duration_minutes`).
- `BookingOut {id, reference_code, court_id, venue_id, type, booking_status, payment_status, display_status, starts_at, ends_at, hours, slots: [{starts_at, price_centavos}], price_centavos, fee_centavos, total_centavos, payment_method, expires_at, cancelled_at, cancellation_reason, notes, created_at}`.
- `GET /availability?venue_id=&date=` (any authenticated user; venue active else 404) → `{venue_id, date, courts: [{court_id, name, slots: [{starts_at, hour, status, price_centavos}]}]}`, active courts in grid order. One query for occupancy (with the hold-expiry filter), one for the venue's rules, prices in memory. Status: occupied by MAINTENANCE → `CLOSED`; occupied otherwise → `BOOKED`; else starts before `now + min_lead` → `PAST`; else `AVAILABLE`. Never reveals who booked.

- [ ] **Tests:** `test_hold_freezes_prices_and_fee` (rules in place → slot prices, sum, 3% fee, `expires_at ≈ now + 15 min`); `test_second_hold_on_same_slot_is_409`; `test_overlapping_hold_is_409` (2h hold vs 1h on its second hour); `test_lapsed_hold_is_expired_when_its_slot_is_taken` (the old group and both its slots become `EXPIRED`); `test_lapsed_holds_do_not_count_toward_cap` (3 lapsed + 1 new succeeds; 3 live → 409 `HOLD_LIMIT`); `test_too_soon_and_beyond_horizon_are_422`; `test_not_on_the_hour_is_422`; `test_inactive_court_is_404`; `test_staff_cannot_place_player_hold` (403); grid: `test_grid_first_row_is_manila_midnight`; `test_grid_marks_booked_closed_and_hides_lapsed_holds`; `test_grid_prices_resolve_per_cell`; `test_grid_marks_too_soon_as_past`; `test_grid_does_not_reveal_who_booked`.

### Task 4: Player reads and cancellation

**Files:** Modify `src/bookings/{service,dto}.py`, `src/bookings/controllers/player.py`. Test `tests/bookings/test_my_bookings.py`.

- `GET /bookings/me?page&limit` → `Page[BookingOut]`, newest first. `GET /bookings/me/{id}` → another player's id is 404. `POST /bookings/me/{id}/cancel {reason?}` → `BookingOut`; guard `can_cancel_by_player`; slots released.
- [ ] **Tests:** `test_my_bookings_lists_only_mine`; `test_other_players_booking_is_404`; `test_lapsed_hold_displays_expired`; `test_cancel_hold_releases_slot` (same slot bookable again); `test_cancel_after_cutoff_is_409` (set `cancellation_cutoff_minutes` high via the parameters table and reset `parameter_service`'s cache); `test_cancel_records_audit_fields`.

### Task 5: Staff operations

**Files:** Create `src/bookings/controllers/admin.py`; modify service/dto/repository. Test `tests/bookings/test_staff_ops.py`.

- `GET /admin/bookings?venue_id&court_id&date&booking_status&type&page&limit` (`BOOKING:ReadAny` + scope) → `Page[AdminBookingOut]` (`BookingOut` + `player: {id, first_name, last_name, email} | null`, `created_by_user_id`), ordered by first slot. `GET /admin/bookings/{id}`.
- `POST /admin/bookings/walk-in {court_id, starts_at, hours, notes?}` (`BOOKING:CreateWalkIn`) → `CONFIRMED/UNPAID`, fee 0, prices resolved; `check_walk_in_window`.
- `POST /admin/bookings/{id}/cancel {reason}` (`BOOKING:CancelAny`), `/mark-paid {payment_method: CASH|MANUAL_GCASH}` (`BOOKING:MarkPaid`), `/outcome {outcome: COMPLETED|NO_SHOW}` (`BOOKING:SetOutcome`). Out of scope → 404.
- [ ] **Tests:** `test_walk_in_is_confirmed_unpaid_with_no_fee`; `test_walk_in_may_backdate_within_window` and `test_walk_in_too_far_back_is_422`; `test_walk_in_collides_with_hold_is_409`; `test_mark_paid_records_method`; `test_mark_paid_online_method_is_422`; `test_mark_paid_player_booking_is_409`; `test_outcome_before_start_is_409`; `test_no_show_still_blocks_the_slot`; `test_staff_cancel_requires_reason` (422 without); `test_staff_cannot_cancel_closure_via_booking_route`; `test_cross_venue_staff_gets_404` (read, cancel, mark-paid, walk-in at B); `test_player_cannot_reach_admin_bookings` (403); `test_admin_list_filters_by_manila_date`.

### Task 6: Closures

**Files:** Create `src/bookings/controllers/closures.py`; modify service/dto. Test `tests/bookings/test_closures.py`.

- `POST /admin/closures {court_ids: [int], starts_at, ends_at, reason}` (`CLOSURE:Manage`) → 201 `list[BookingOut]`, one MAINTENANCE group per court, N slot rows each; all-or-nothing; any collision → 409 `SLOT_UNAVAILABLE`. `starts_at` ≥ the start of the current hour, both on the hour, `ends_at > starts_at`, ≤ 31 days.
- `POST /admin/closures/{id}/cancel {reason}` → reopens the slots.
- [ ] **Tests:** `test_closure_blocks_bookings_and_shows_closed`; `test_closure_cannot_evict_a_booking` (409, nothing written for any court); `test_cancel_closure_reopens`; `test_staff_can_close` (staff hold `CLOSURE:Manage`); `test_closure_other_venue_court_is_404`; `test_closure_range_limits` (31 days + 1 hour → 422; past → 422).
- [ ] Update both briefs; push both repos; CI green.

## Out of scope (Phase 5)

Checkout session creation, `checkout_url`, the provider port and `MockProvider`, webhooks, `payments`/`webhook_events` tables, late-payment re-acquisition, `needs_resolution` and the unfulfilled queue, receipt email.
