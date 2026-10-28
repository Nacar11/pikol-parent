# Booking Action Events Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Record every booking and payment change in an append-only `action_events` table and show it as a coloured, parcel-tracker timeline to staff (`/staff/bookings/{id}`) and to customers (`/book/result/{checkoutId}`).

**Architecture:** A new backend module `src/action_events/` (enums → model → repository → service → DTO). Existing services call `event_service.record(...)` before their own commit, so an event and its change share one transaction. Two read endpoints (staff-scoped, customer-owned) merge stored events with two derived ones. The frontend adds one shared `ActionTimeline` in the `bookings` slice.

**Tech Stack:** FastAPI, SQLAlchemy 2 async, Alembic, Postgres (backend); Next.js 15, TanStack Query, zod, Tailwind, vitest, Playwright (frontend).

**Spec:** [`2026-10-28-booking-action-events-design.md`](2026-10-28-booking-action-events-design.md). Read it; the event table there (types, actors, colours, customer visibility, `data` keys) is normative and is not repeated here.

## Global Constraints

- Events are written **inside the transaction of the change**, before its `commit()`; never after commit, never best-effort.
- `occurred_at` is when it happened: `EXPIRED` = the group's `expires_at`; everything else = the `now` the service already computed.
- Append-only: the repository exposes `insert` and reads, nothing else.
- Customers never see `FLAGGED`, `RESOLVED`, or staff names (staff → "Venue staff"; their own actions → "You").
- Payment events (`PAYMENT_*`) carry `checkout_id` only (`booking_group_id` NULL): one per checkout.
- Schema: change `docs/adr/pikol.dbml` first, then migration `0011_action_events`.
- Every chip carries its word; colour is never the only signal.
- Backend gates: `uv run ruff format . && uv run ruff check . && uv run mypy src && uv run pytest && uv run vulture`. Frontend gates: `npm run lint:ci && npm run format:check && npm run typecheck && npm test && npm run deadcode`, plus `npm run e2e`.
- Commit straight to each repo's `main`, push after its gates pass; trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

1. **A payment covering two courts** — one `PAYMENT_SUCCEEDED`, two `CONFIRMED` (one per court), and the customer timeline names the court on each `CONFIRMED`. (Task 3 test.)
2. **A hold cleaned up lazily hours after it lapsed** — `EXPIRED.occurred_at` equals the old `expires_at`, not the cleanup time. (Task 3 test.)
3. **A collision that rolls back a savepoint** (walk-in, closure, checkout create hitting `SLOT_UNAVAILABLE`) — no orphan events survive. (Tasks 2 and 3 tests.)
4. **The same Maya event delivered twice** (webhook + result-page refresh) — events are not duplicated; the existing `record_event` guard returns early. (Task 3 test.)
5. **Someone else's booking** — staff at another venue and another customer both get 404 from the events endpoints, never an empty list. (Task 4 test.)

---

### Task 1: Schema, model and recorder (backend)

**Files:**
- Modify: `../docs/adr/pikol.dbml` (enums `action_event_type`, `actor_kind`; table `action_events`; Appendix A: both CHECKs and both indexes)
- Create: `src/database/migrations/versions/0011_action_events.py` (revises `0010_one_open_attempt`)
- Create: `src/action_events/__init__.py`, `domain/__init__.py`, `domain/enums.py`, `persistence/__init__.py`, `persistence/models.py`, `persistence/repository.py`, `service.py`
- Modify: `src/database/migrations/env.py` (add `import src.action_events.persistence.models  # noqa: F401`)
- Test: `tests/action_events/__init__.py`, `tests/action_events/test_schema.py`

**Interfaces:**
- Produces:
  - `ActionEventType(StrEnum)`: `HOLD_PLACED WALK_IN_BOOKED CLOSURE_CREATED PAYMENT_STARTED PAYMENT_SUCCEEDED PAYMENT_FAILED PAYMENT_CANCELLED CONFIRMED MARKED_PAID COMPLETED NO_SHOW CANCELLED EXPIRED REFUNDED VOIDED FLAGGED RESOLVED REOPENED`
  - `ActorKind(StrEnum)`: `STAFF CUSTOMER SYSTEM`
  - `ActionEventModel` (`__tablename__ = "action_events"`), columns per the spec
  - `action_event_repository.insert(session, *, type, actor_kind, actor_user_id, occurred_at, booking_group_id=None, checkout_id=None, data=None) -> ActionEventModel`
  - `action_event_repository.for_booking(session, group: BookingGroupModel) -> list[ActionEventModel]` — the group's own events plus its checkout's booking-less ones, newest first
  - `action_event_repository.for_checkout(session, checkout_id: int) -> list[ActionEventModel]` — newest first
  - `event_service.record(session, type, *, actor: UserModel | None, group: BookingGroupModel | None = None, checkout_id: int | None = None, data: dict[str, Any] | None = None, occurred_at: datetime | None = None) -> None` — `actor_kind` is derived: `None` → SYSTEM; `actor.id == group/checkout customer` → CUSTOMER; else STAFF. A given `group` also fills `checkout_id` from `group.checkout_id`. Datetimes inside `data` are stored as ISO strings.

- [x] **Step 1: Failing tests** in `test_schema.py`: `test_event_needs_a_booking_or_checkout` (insert with both NULL → IntegrityError on `ck_action_event_has_subject`); `test_system_events_have_no_user` (SYSTEM with a user id → IntegrityError on `ck_action_event_actor`); `test_record_fills_checkout_from_group` (record on a player group → row has both ids, `actor_kind == CUSTOMER`); `test_backfill_creates_creation_and_cancellation_events` (run the migration's backfill SQL over one walk-in and one cancelled player booking → `WALK_IN_BOOKED` at `created_at`, `HOLD_PLACED` + `CANCELLED` at `cancelled_at` with the reason).
- [x] **Step 2:** `uv run pytest tests/action_events -v` → FAIL (module missing).
- [x] **Step 3:** Implement dbml, migration (enums, table, CHECKs, indexes `ix_action_events_group_time`, `ix_action_events_checkout_time`, backfill INSERT…SELECT keyed by `booking_groups.type`), model, repository, service. Expose the backfill SQL as a module constant so the test can run it.
- [x] **Step 4:** `uv run alembic upgrade head && uv run pytest tests/action_events -v` → PASS.
- [x] **Step 5:** Commit `pikol-backend` ("Add the action_events table and recorder"); commit the dbml in `pikol-parent`.

### Task 2: Record staff-side events (backend)

**Files:**
- Modify: `src/bookings/service.py` (create_walk_in, close_courts, mark_paid, set_outcome, resolve, `_cancel` → CANCELLED for bookings, REOPENED for closures)
- Modify: `src/bookings/controllers/admin.py` (pass the guard's user into `mark_paid` and `set_outcome`)
- Test: `tests/action_events/test_staff_events.py`

**Interfaces:**
- Consumes: `event_service.record`, `action_event_repository.for_booking` (Task 1).
- Produces: `booking_service.mark_paid(session, staff: UserModel, booking_id, method, *, scope)` and `booking_service.set_outcome(session, staff: UserModel, booking_id, outcome, *, scope)` — `staff` is the new second positional parameter.

- [x] **Step 1: Failing tests** (reuse `tests/bookings/conftest.py` `World` and the `walk_in` helper pattern from `test_staff_ops.py`): `test_walk_in_records_walk_in_booked` (one event, STAFF, actor = desk user, `data.notes`); `test_mark_paid_records_method_and_amount` (`MARKED_PAID`, `data == {"method": "CASH", "amount_centavos": total}`); `test_outcome_records_completed_and_no_show` (parametrised); `test_staff_cancel_records_reason`; `test_closure_records_one_event_per_court` and `test_reopen_records_reopened`; `test_resolve_records_resolved` (`data.resolution`); `test_failed_walk_in_records_nothing` (409 collision → zero events for that court).
- [x] **Step 2:** `uv run pytest tests/action_events/test_staff_events.py -v` → FAIL.
- [x] **Step 3:** Add one `event_service.record(...)` before each commit; pass `staff` through.
- [x] **Step 4:** Run the new file plus `tests/bookings` → PASS (existing tests updated only where the `mark_paid`/`set_outcome` signature changed).
- [x] **Step 5:** Commit ("Record staff booking actions as events").

### Task 3: Record customer and payment events (backend)

**Files:**
- Modify: `src/checkouts/service.py` (create → `HOLD_PLACED` per group with `expires_at`, `total_centavos`; cancel → `CANCELLED` per group, actor customer, reason `CUSTOMER_CANCEL_REASON`)
- Modify: `src/payments/service.py` (start_checkout → `PAYMENT_STARTED`, recorded only when this call's row wins; `_release` → `PAYMENT_FAILED`/`PAYMENT_CANCELLED` plus per-group `CANCELLED` or `EXPIRED`; `_paid` → `PAYMENT_SUCCEEDED` (+`late: true` on the late path) and per-group `CONFIRMED`; refund via webhook / `_undo` → `REFUNDED` or `VOIDED` with `amount_centavos`, `reference`; `_flag` becomes `async` and takes `session`, recording `FLAGGED` per group with `reason`)
- Modify: `src/bookings/persistence/repository.py` (`expire_lapsed` reads `(id, expires_at)` of the candidates before the UPDATE and records `EXPIRED` per expired id at its own `expires_at`)
- Test: `tests/action_events/test_payment_events.py`

**Interfaces:**
- Consumes: Task 1. Uses the `tests/payments/helpers.py` `pay`, `open_maya`, `groups` and the `tests/checkouts/helpers.py` `lapse`, `item`.
- Produces: no new public names.

- [x] **Step 1: Failing tests:** `test_hold_records_hold_placed_per_group`; `test_two_court_payment_records_one_payment_event_and_two_confirmed` (Review Focus 1); `test_failed_payment_records_failed_then_cancelled`; `test_backed_out_records_payment_cancelled`; `test_lazy_expiry_is_dated_at_the_deadline` (lapse a hold by 120 min, then walk-in on the same hour → `EXPIRED.occurred_at == old expires_at`; Review Focus 2); `test_collision_on_create_records_nothing` (Review Focus 3); `test_replayed_webhook_records_once` (Review Focus 4); `test_late_payment_taken_hour_records_flagged_then_refund` (`FLAGGED`, then `VOIDED` same Manila day); `test_customer_cancel_records_cancelled_by_customer`.
- [x] **Step 2:** Run → FAIL.
- [x] **Step 3:** Implement. `expire_lapsed` imports `event_service` lazily or through the repository-level helper to avoid an import cycle; pick one and keep it.
- [x] **Step 4:** `uv run pytest tests/action_events tests/checkouts tests/payments tests/bookings -v` → PASS.
- [x] **Step 5:** Commit ("Record customer and payment lifecycle events").

### Task 4: Read endpoints, derived events, redaction (backend)

**Files:**
- Create: `src/action_events/dto.py`
- Modify: `src/action_events/service.py` (timelines)
- Modify: `src/bookings/controllers/admin.py` (`GET /admin/bookings/{booking_id}/events`, guard `read_any`, scoped via `booking_service.get_scoped`)
- Modify: `src/checkouts/controllers/customer.py` (`GET /checkouts/me/{checkout_id}/events`, via `checkout_service.get_mine`)
- Test: `tests/action_events/test_event_reads.py`

**Interfaces:**
- Consumes: Tasks 1–3.
- Produces (JSON contract the frontend parses):
  - `EventActor`: `{kind: "STAFF"|"CUSTOMER"|"SYSTEM", name: str | None}`
  - `ActionEventOut`: `{id: int | None, type: ActionEventType | "PLAY_STARTED" | "HOLD_ENDED", occurred_at, actor: EventActor, booking_id: int | None, reference_code: str | None, court_name: str | None, data: dict, derived: bool}`
  - Both endpoints return the list envelope `{data, total, page, limit, has_more}` (one page, `limit` 200), newest first.
  - `event_service.staff_timeline(session, group, now) -> list[ActionEventOut]`; `event_service.customer_timeline(session, checkout, now) -> list[ActionEventOut]`.

- [x] **Step 1: Failing tests:** `test_staff_timeline_newest_first_with_names` (actor names are "First Last"); `test_derived_play_started_after_start` (confirmed group whose first slot started → `PLAY_STARTED` dated at its start, `derived: true`, `id: null`); `test_derived_hold_ended_for_untouched_lapsed_hold`; `test_customer_view_hides_flag_and_resolve_and_staff_names` (staff → `name == "Venue staff"`, own → `"You"`); `test_other_venue_staff_gets_404` and `test_other_customer_gets_404` (Review Focus 5).
- [x] **Step 2:** Run → FAIL.
- [x] **Step 3:** Implement; derived types exist only in the DTO, never in the DB enum.
- [x] **Step 4:** Run all backend gates → PASS.
- [x] **Step 5:** Commit and push `pikol-backend` ("Serve booking timelines to staff and customers"). Update `pikol-backend/CLAUDE.md` module table in the same commit.

### Checkpoint A: backend green and pushed.

### Task 5: Timeline component and data (frontend)

**Files:**
- Create: `src/features/bookings/action-events.ts` (zod schema, look-up, detail line)
- Create: `src/features/bookings/components/action-timeline.tsx`
- Modify: `src/features/bookings/index.ts` (export `ActionTimeline`, `ActionEventPageSchema`, type `ActionEvent`)
- Modify: `src/app/globals.css`, `tailwind.config.ts` (tokens below)
- Test: `tests/unit/bookings/action-timeline.test.tsx`

**Interfaces:**
- Consumes: Task 4's JSON contract.
- Produces:
  - `ActionEventSchema`, `ActionEventPageSchema`, `type ActionEvent`
  - `type EventTone = 'wait' | 'confirmed' | 'done' | 'bad' | 'neutral'`
  - `EVENT_LOOK: Record<ActionEvent['type'], { title: string; chip: string; tone: EventTone; icon: LucideIcon }>` — tones per the spec's colour column; `PLAY_STARTED` → confirmed, `HOLD_ENDED` → bad
  - `eventDetail(e: ActionEvent): string | null` (amount + method, reason, "Held until 3:05 PM", refund reference…, all money via `formatPeso`, times via `lib/manila`)
  - `ActionTimeline({ events, showCourt }: { events: ActionEvent[]; showCourt: boolean })` — an `<ol aria-label="Action events">`, each item: dot (tone bg + icon), title, chip (`bg-event-<tone>-soft text-event-<tone>-fg`, its word), detail, actor line, `<time dateTime>` "3:02 PM · Wed 28 Oct" in Manila; court name prefixed when `showCourt`.
- Tokens: `--event-<tone>`, `--event-<tone>-soft`, `--event-<tone>-fg` for the five tones; Tailwind `colors.event.<tone>.{DEFAULT,soft,fg}`. Values: wait amber (`38 92% 50%` / `48 96% 89%` / `26 83% 30%`), confirmed blue (`221 83% 53%` / `214 95% 93%` / `224 64% 33%`), done green (`142 72% 29%` / `141 79% 85%` / `144 61% 20%`), bad red (reuse `--destructive` / `0 93% 94%` / `0 74% 35%`), neutral grey (`0 0% 45%` / `0 0% 96%` / `0 0% 25%`).

- [x] **Step 1: Failing tests:** renders newest first as given; every stored type has a title and chip word (loop over the enum); `MARKED_PAID` detail reads "₱1,320 · Cash"; `HOLD_PLACED` reads "Held until …" in Manila; chip of `CANCELLED` has the `bad` classes and the word "Cancelled"; `showCourt` prefixes "Court 2"; an empty list renders "No events yet."
- [x] **Step 2:** `npm test -- action-timeline` → FAIL.
- [x] **Step 3:** Implement.
- [x] **Step 4:** `npm test -- action-timeline && npm run typecheck` → PASS.
- [x] **Step 5:** Commit ("Add the action events timeline").

### Task 6: Wire it in, staff and customer (frontend)

**Files:**
- Modify: `src/features/admin-bookings/api.ts` (`events(id)`), `keys.ts` (`events: (id) => [...adminBookingKeys.detail(id), 'events']`), `hooks/use-admin-booking.ts` (`useAdminBookingEvents(id)`), `components/admin-booking-detail.tsx` (section "Action events" under `Summary`, `showCourt={false}`)
- Modify: `src/features/booking/api.ts` (`checkoutApi.events(id)`), `keys.ts` (`events: (id) => [...checkoutKeys.detail(id), 'events']`), `hooks/use-checkouts.ts` (`useCheckoutEvents(id, { poll })`, same poll rule as `useCheckout`), `components/result-view.tsx` (section under the status, `showCourt` when the checkout has more than one booking)
- Test: extend `tests/unit/admin-bookings/admin-booking-detail.test.tsx`, `tests/unit/booking/result-view.test.tsx`; create `e2e/action-events.spec.ts`

**Interfaces:**
- Consumes: Task 5 exports; Task 4 endpoints.

- [x] **Step 1: Failing tests:** detail page shows "Action events" with the mocked events, and after Cash is confirmed the events query refetches (assert `api.events` called twice); result view shows the customer timeline. E2E: desk books a walk-in → Cash → cancel, booking page lists Cancelled, Paid · Cash, Walk-in booked in that order; customer books and pays on the mock Maya page, result page lists Confirmed, Payment successful, Payment started, Awaiting payment (hold placed).
- [x] **Step 2:** Run → FAIL.
- [x] **Step 3:** Implement. Staff action invalidation already covers `adminBookingKeys.all`, which includes the new key.
- [x] **Step 4:** All frontend gates + `npm run e2e` → PASS; screenshot the staff page and the customer result page at 1400px and 375px and look at them.
- [x] **Step 5:** Commit and push `pikol-frontend` ("Show booking action events to staff and customers"), with `pikol-frontend/CLAUDE.md` updated (the B&W rule's one exception; `ActionTimeline` in the `bookings` row).

### Task 7: Record it (parent)

- [x] Root `CLAUDE.md` current state mentions action events and links the design and this plan; the B&W exception is noted where colour is discussed.
- [x] Commit and push `pikol-parent` ("Record the booking action events and their plan").

## What the build found

- `event_service.record` takes the checkout **model**, not its id: telling a
  customer from staff on a payment event needs the checkout's customer.
- The backfill SQL lives in the migration as one `INSERT … UNION ALL`
  (asyncpg runs one statement at a time); the test loads it from the
  migration file, since a migration never imports app code.
- `booking_repository.expire_lapsed` writes `EXPIRED` through the event
  repository directly (repository → repository), reading each deadline before
  the UPDATE clears it.
- Payment-failure cancellations and voids/refunds are SYSTEM events.
  `PAYMENT_SUCCEEDED` is recorded wherever money verifiably arrived; the
  mismatch and after-final paths record `FLAGGED` only.
- "Play started" also shows for COMPLETED bookings.
- Tailwind never sees class names assembled at runtime, so every tone has a
  static class map (dot, halo, chip).
- The e2e checks live in the existing `booking.spec` and
  `booking-decisions.spec` flows, not a new spec, to avoid claiming new hours
  on the shared e2e database.

Results: backend 489 tests; frontend 322 unit tests; e2e 26/26.
