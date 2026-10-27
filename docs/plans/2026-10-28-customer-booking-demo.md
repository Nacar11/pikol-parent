# Customer booking demo — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Customers book and pay on the web, end to end, through the same
`/checkouts/me` API the mobile app will use, with every ADR 0004 rule and all
17 of its scenarios exercised against the mock Maya.

**Architecture:** A new `checkouts` row is the unit of payment: one customer,
one venue, N `booking_groups` (one per consecutive run on one court), one
Maya attempt, one hold deadline. `sync_payment` is rewritten around the
checkout and gains the simple refund flow (void same Manila day, else
refund). The frontend gains a customer-only `booking` slice under `/book`.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 async, Alembic, Postgres enums,
pytest; Next.js 15 App Router, TanStack Query, zod, vitest, Playwright.

**Spec:** [`2026-10-27-customer-booking-demo-design.md`](2026-10-27-customer-booking-demo-design.md)
and [ADR 0004](../adr/0004-online-payment-flow-and-statuses.md). Read both;
this plan does not repeat their rationale.

## Global Constraints

- `docs/adr/pikol.dbml` changes **first**, then the migration (system brief).
- Five customer-visible statuses, exactly: `PENDING`, `SUCCESSFUL`, `FAILED`, `CANCELLED`, `REFUNDED`.
- `hold_duration_minutes` = **5**; `min_lead_minutes` may be **0**; the `min_lead_minutes >= hold_duration_minutes` validator is removed (design call 1).
- An hour whose start has passed can never be held or re-acquired.
- One venue per checkout; creating one is all-or-nothing (409 `SLOT_UNAVAILABLE`, detail names the taken hours).
- `max_active_holds_per_player` counts live pending **checkouts** (`status = 'PENDING' AND expires_at > now`).
- Every `PENDING` read adds `expires_at > now` — checkouts included.
- A paid booking is final: cancel on `SUCCESSFUL` is 409 `PAID_IS_FINAL`.
- Status is decided only by the re-fetched payment, never the webhook body or the browser redirect.
- Provider calls happen outside a transaction that holds a hold; the refund call happens after the "refund pending" state commits.
- Money is integer centavos; Maya pesos exist only in `maya_wire.py`.
- Customer routes take identity from the token: `/api/v1/checkouts/me/...`. Retry route `POST /bookings/me/{id}/checkout` is removed, as is the rest of the old `/bookings` player router.
- Mock outcomes: **Pay, Decline, Back out, Expire**, each delivering the webhook through `sync_payment`.
- Frontend: black and white, Manila times, `notify` for feedback, existing `PageHeader`/`Pill`/`Skeleton`/`StatStrip`; no favorites UI; no sign-up page.
- Result page polls every **3 s** for up to **2 minutes** while `PENDING`, then offers "Check again".
- Git: commit straight to `main` in the repo whose files changed; run that repo's full gate before pushing.

## Review Focus

1. **Double "Continue to Maya"** (double click, reload, my-bookings button): must not open a second Maya session that could also be paid. `/pay` is idempotent — test in Task 5.
2. **Customer cancels, then the payment they already made lands**: the cancelled checkout takes the late path (re-acquire if free, else refund), never drops the money — test in Task 7.
3. **Late success after a held hour has started**: refund, not re-acquire, even if the slot is free — test in Task 7.
4. **Hold page left open past zero**: "Continue to Maya" disables and says the hold ended; the server also refuses `/pay` with 409 `HOLD_ENDED` — tests in Tasks 5 and 13.
5. **Double "Pay now"**: the button disables while the request is in flight, so a customer never collides with their own first hold — test in Task 12.

---

## File map

**pikol-parent:** `docs/adr/pikol.dbml`, `docs/adr/0004-…md` (drop the
lead ≥ hold line, mark Accepted), `CLAUDE.md` (state, routes, vocabulary).

**pikol-backend (new):**
- `src/checkouts/domain/rules.py` — pure: `CheckoutItem`, item checks, fee allocation, display status, undo kind.
- `src/checkouts/persistence/models.py` — `CheckoutModel`, `RefundModel`.
- `src/checkouts/persistence/repository.py` — `checkout_repository` (the only `checkouts.status` writer).
- `src/checkouts/service.py` — create, get/list mine, pay (open the Maya attempt), cancel.
- `src/checkouts/dto.py`, `src/checkouts/controllers/customer.py` — `/checkouts/me` routes.
- `src/favorites/{persistence/models.py,persistence/repository.py,controllers/me.py}` — favorite venues.
- `src/devtools/demo_seed.py` — demo customers.
- `src/database/migrations/versions/0008_checkouts.py`, `0009_favorite_venues.py`.

**pikol-backend (changed):** `payments/domain/{enums,port}.py`,
`payments/adapters/{maya_wire,mock}.py`, `payments/persistence/{models,repository}.py`,
`payments/service.py`, `payments/controllers/mock_maya.py`,
`bookings/{service,dto}.py`, `bookings/domain/rules.py`,
`bookings/persistence/models.py`, `parameters/domain/parameters.py`,
`email/templates.py`, `main.py`, `devtools/e2e_seed.py`; delete
`bookings/controllers/player.py`; `CLAUDE.md`, `vulture_whitelist.py`.

**pikol-frontend (new):** `src/features/booking/` (`api.ts`, `schemas.ts`,
`keys.ts`, `picks.ts`, `hooks/`, `components/`, `index.ts`);
`src/app/(app)/book/{layout,page}.tsx`, `book/[venueId]/page.tsx`,
`book/checkout/[id]/page.tsx`, `book/result/[id]/page.tsx`,
`book/mine/page.tsx`; `e2e/booking.spec.ts`; unit tests under
`tests/unit/booking/`.

**pikol-frontend (changed):** `features/auth/landing.ts` (+ callers),
`features/catalog` (venue list), `features/availability/schemas.ts`
(fee percent), `playwright.config.ts`, `e2e/fixtures.ts`; delete
`app/(app)/customer-app/`; `CLAUDE.md`.

---

## Backend

### Task 1: Schema — checkouts, refunds, payment outcomes, parameters

**Files:**
- Modify: `../docs/adr/pikol.dbml` (first), `../docs/adr/0004-online-payment-flow-and-statuses.md`
- Create: `src/database/migrations/versions/0008_checkouts.py`, `src/checkouts/persistence/models.py`
- Modify: `src/payments/domain/enums.py`, `src/payments/persistence/models.py`, `src/bookings/persistence/models.py`, `src/parameters/domain/parameters.py`, `src/database/migrations/env.py` (model import)
- Test: `tests/checkouts/test_checkout_schema.py`, `tests/parameters/test_parameters_domain.py`, `tests/payments/test_payments_schema.py`

**Interfaces:**
- Produces: `PaymentOutcome(StrEnum)` = `PENDING, SUCCESSFUL, FAILED, CANCELLED, REFUNDED` (replaces `ProviderPaymentStatus`; `TERMINAL` is removed); `RefundKind(StrEnum)` = `VOID, REFUND`;
  `CheckoutModel` (`checkouts`): `id, reference_code varchar(12) unique, customer_id → users RESTRICT, venue_id → venues RESTRICT, status payment_outcome, price_centavos, fee_centavos, total_centavos, expires_at timestamptz NOT NULL, created_at, updated_at`, relationships `customer` (joined), `venue` (joined), `bookings: list[BookingGroupModel]` (selectin, ordered by group id);
  `RefundModel` (`refunds`): `id, payment_id → payments unique CASCADE, kind refund_kind, amount_centavos, provider_reference varchar(128), created_at`;
  `PaymentModel`: `checkout_id → checkouts CASCADE` (replaces `booking_group_id`), `status: PaymentOutcome`, new `provider_status varchar(32) NULL`, new `checkout_url text NOT NULL`;
  `BookingGroupModel.checkout_id → checkouts RESTRICT NULL`, indexed.
- Indexes: `checkouts (customer_id, status)`, `checkouts (status, expires_at)`.
- Appendix A additions: `ck_checkout_money` (`price, fee, total >= 0 AND total = price + fee`); `ck_checkout_only_on_player_booking` on `booking_groups`: `(type = 'PLAYER_BOOKING') = (checkout_id IS NOT NULL)`.

- [ ] **Step 1: Update `pikol.dbml`.** Add the two enums, `checkouts` and `refunds` tables (TableGroup `money`), the payments/booking_groups columns, refs, and Appendix A §14–15 above. Notes to carry: "The checkout is the unit of PAYMENT; the booking group stays the unit of OCCUPANCY (one court, ADR 0001)"; "`checkouts.expires_at` is kept after the hold ends — it is the record of the deadline, and `PENDING` never means live without `expires_at > now()`". In Appendix B: `hold_duration_minutes` 15 → **5**, `booking_horizon_days` 30 → **14**, delete "MUST be >= hold_duration_minutes" (also in the `parameters` table note). In ADR 0004: delete the "`min_lead_minutes` (20) must stay ≥ …" paragraph, add "An hour whose start has passed can never be held (design call 1)", Status → **Accepted**.

- [ ] **Step 2: Write failing tests.**
  - `test_parameters_domain.py`: replace `test_min_lead_below_hold_duration_is_rejected` with `test_zero_lead_with_a_five_minute_hold_is_valid` — `Parameters.model_validate(VALID | {"min_lead_minutes": "0", "hold_duration_minutes": "5"})` succeeds.
  - `tests/checkouts/test_checkout_schema.py`: inserting a `PLAYER_BOOKING` group with `checkout_id=None` raises `IntegrityError` naming `ck_checkout_only_on_player_booking`; a `WALK_IN` with a checkout id raises the same; a checkout with `total != price + fee` raises `ck_checkout_money`; two refunds for one payment raise the unique violation.
  - `test_payments_schema.py`: `payment_outcome` enum labels are exactly the five, in order.
  - `tests/parameters/test_admin_parameters.py`: the "cross-field break is 422" test now needs a different cross-field case — there is none left, so delete it and add `test_owner_sets_min_lead_to_zero` (PATCH `min_lead_minutes` `"0"` → 200).

- [ ] **Step 3: Run** `uv run pytest tests/checkouts tests/parameters tests/payments/test_payments_schema.py -q` — expect FAIL (no table, validator still present).

- [ ] **Step 4: Write `0008_checkouts.py`**, frozen literals only (no `src` imports of values). Order:
  1. Create enums `payment_outcome`, `refund_kind` (`create(..., checkfirst=True)`).
  2. Create `checkouts`, `refunds` with constraints above.
  3. `booking_groups.checkout_id` (nullable FK + index).
  4. Backfill one checkout per existing `PLAYER_BOOKING` group: `INSERT INTO checkouts (reference_code, customer_id, venue_id, status, price_centavos, fee_centavos, total_centavos, expires_at, created_at) SELECT bg.reference_code, bg.player_id, c.venue_id, <status>, bg.price_centavos, bg.fee_centavos, bg.total_centavos, COALESCE(bg.expires_at, bg.created_at), bg.created_at FROM booking_groups bg JOIN courts c …`, where `<status>` is `SUCCESSFUL` if `payment_status='PAID' AND NOT needs_resolution`, `PENDING` if `booking_status='PENDING'`, `FAILED` if `payment_status='FAILED'`, else `CANCELLED`. Then `UPDATE booking_groups SET checkout_id = ck.id FROM checkouts ck WHERE ck.reference_code = booking_groups.reference_code AND booking_groups.type = 'PLAYER_BOOKING'`.
  5. Add the `ck_checkout_only_on_player_booking` check.
  6. `payments`: add `checkout_id`, `provider_status`, `checkout_url` (backfill `''`, then NOT NULL); set `checkout_id` from the group; NOT NULL; drop `booking_group_id`; convert `status` with `USING (CASE status::text WHEN 'PAID' THEN 'SUCCESSFUL' WHEN 'EXPIRED' THEN 'CANCELLED' ELSE status::text END)::payment_outcome`; drop type `provider_payment_status`.
  7. `UPDATE parameters SET value='5' WHERE key='hold_duration_minutes' AND value='15'`; same for `booking_horizon_days` `'30'` → `'14'`; update `min_lead_minutes`'s description to drop the "must be ≥" sentence.
  Downgrade restores 0007's shape (`payments.booking_group_id` = the checkout's lowest group id; statuses mapped back `SUCCESSFUL→PAID`, `REFUNDED→PAID`) and drops the new objects.

- [ ] **Step 5: Models and enums.** Update the models to match; remove `lead_time_must_cover_the_hold`. Fix every import of `ProviderPaymentStatus` so the suite imports cleanly (behaviour changes come in Tasks 3–7; payments tests that break here are rewritten there — mark them `xfail(reason="rewritten in Task 7", strict=False)` only if needed to keep this commit green, and remove every such mark by Task 7).

- [ ] **Step 6: Verify** `uv run alembic upgrade head && uv run alembic downgrade -1 && uv run alembic upgrade head`, then the Step 3 tests pass.

- [ ] **Step 7: Commit** parent (`dbml`, ADR) and backend separately; run the backend gate (`ruff format . && ruff check . && mypy src && pytest && vulture`) first.

---

### Task 2: Checkout domain rules (pure)

**Files:**
- Create: `src/checkouts/domain/rules.py`
- Test: `tests/checkouts/test_checkout_domain.py`

**Interfaces:**
- Produces:
  - `MAX_CHECKOUT_HOURS = 24`
  - `@dataclass(frozen=True) class CheckoutItem: court_id: int; starts_at: datetime; hours: int`
  - `check_items(items: Sequence[CheckoutItem]) -> None` — raises `BookingError` (`invalid`): `EMPTY_CHECKOUT` (no items), `TOO_MANY_HOURS` (sum > 24), `OVERLAPPING_ITEMS` (two items share a (court, hour)). Each item also passes `check_on_the_hour`.
  - `allocate_fee(fee_centavos: int, prices: Sequence[int]) -> list[int]` — proportional floor shares, remainder on the last; sum always equals `fee_centavos`.
  - `checkout_display_status(status: PaymentOutcome, expires_at: datetime, now: datetime) -> PaymentOutcome` — `PENDING` with `expires_at <= now` reads `CANCELLED`.
  - `is_live(status, expires_at, now) -> bool`.
  - `undo_kind(paid_at: datetime, now: datetime) -> RefundKind` — `VOID` when both fall on the same Manila day, else `REFUND`.

- [ ] **Step 1: Write failing tests** (names and assertions):
  - `test_items_may_not_overlap_on_one_court` — `(c1, 10:00, 2h)` + `(c1, 11:00, 1h)` → `OVERLAPPING_ITEMS`; same hours on `c2` is fine.
  - `test_twenty_five_hours_is_too_many` → `TOO_MANY_HOURS`; 24 passes.
  - `test_empty_checkout_is_invalid` → `EMPTY_CHECKOUT`.
  - `test_fee_allocation_sums_to_the_checkout_fee` — `allocate_fee(46, [50000, 50000, 60000]) == [14, 14, 18]`, and for 200 random price lists `sum(...) == fee`.
  - `test_lapsed_pending_reads_cancelled` / `test_live_pending_reads_pending` / `test_final_statuses_read_as_stored`.
  - `test_void_until_manila_midnight` — `paid_at=2026-10-28T15:59:00Z` (23:59 Manila), `now=15:59:30Z` → `VOID`; `now=16:00:00Z` (00:00 Manila next day) → `REFUND`. This is scenario 10's rule.
- [ ] **Step 2: Run** `uv run pytest tests/checkouts/test_checkout_domain.py -q` → FAIL (module missing).
- [ ] **Step 3: Implement** in `rules.py`. No SQLAlchemy, no clock.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `Add the checkout's pure rules`.

---

### Task 3: Create and read checkouts (`/checkouts/me`), retire the old player router

**Files:**
- Create: `src/checkouts/persistence/repository.py`, `src/checkouts/service.py`, `src/checkouts/dto.py`, `src/checkouts/controllers/customer.py`
- Modify: `src/bookings/service.py` (remove `create_hold`, `get_for_player`, `checkout_target`, `cancel_by_player`; keep `bookable_court`, `_priced_slots` as `priced_slots`), `src/bookings/domain/rules.py` (remove `can_cancel_by_player`, `can_start_checkout`), `src/bookings/persistence/repository.py` (remove `count_live_holds`, `list_for_player`, `get_for_player`), `src/bookings/dto.py` (remove `HoldRequest`'s player use if unused — `WalkInRequest` keeps its fields; `Availability` gains `convenience_fee_percent: Decimal`), `src/main.py`
- Delete: `src/bookings/controllers/player.py`, `tests/bookings/test_my_bookings.py`, `tests/payments/test_checkout.py` (their cases move to `tests/checkouts/`)
- Modify: `tests/bookings/conftest.py` (`hold` helper), every test using it
- Test: `tests/checkouts/test_create.py`

**Interfaces:**
- Consumes: Task 2 rules; `booking_repository.insert/expire_lapsed/occupancy`; `pricing_service.resolvers_for`; `convenience_fee_centavos`.
- Produces:
  - `CheckoutCreate(BaseModel, extra="forbid")`: `items: list[CheckoutItemIn]` (1..24), `CheckoutItemIn` = `court_id: int, starts_at: AwareDatetime, hours: int (1..12)`.
  - `CheckoutOut`: `id, reference_code, venue_id, venue_name, status: PaymentOutcome` (**display** status, Task 2), `expires_at, price_centavos, fee_centavos, total_centavos, checkout_url: str | None` (the attempt's URL while `status == PENDING`, else null), `refund_kind: RefundKind | None`, `bookings: list[CheckoutBookingOut]`, `created_at`. `CheckoutBookingOut`: `id, reference_code, court_id, court_name, starts_at, ends_at, hours, price_centavos, booking_status`.
  - `checkout_repository`: `insert(session, *, fields) -> CheckoutModel` (reference-code retry like `booking_repository.insert`), `set_status(session, checkout, status) -> None` (**only** writer of `checkouts.status`), `count_live(session, customer_id, now) -> int`, `list_for_customer(session, customer_id, *, page, limit)`, `get_for_customer(session, checkout_id, customer_id)`, `get_for_update(session, checkout_id) -> CheckoutModel` (row lock, `of=CheckoutModel`).
  - `checkout_service.create(session, customer, body) -> CheckoutModel`, `get_mine(session, customer, checkout_id) -> CheckoutModel` (404 `NOT_FOUND` for another customer's), `CheckoutOut.of(checkout, latest_payment, now)`.
  - Routes (all `require_customer`): `POST /checkouts/me` 201, `GET /checkouts/me` → `Page[CheckoutOut]` (newest first), `GET /checkouts/me/{id}`.
  - Test helper `hold(client, headers, court_id, starts_at, hours=1) -> dict` now POSTs one item to `/checkouts/me` and returns `{**checkout_json["bookings"][0], "checkout_id": checkout_json["id"], "checkout": checkout_json}` so existing `booking["id"]` reads still mean the group id.

`create` order, one transaction: load params → for each item resolve `bookable_court(scope=CATALOG)` → `MIXED_VENUES` (422) if venues differ → `check_items` → `check_player_window` per item → hold cap via `count_live` (409 `HOLD_LIMIT`, message unchanged) → price every slot, `fee = convenience_fee_centavos(sum, percent)`, `allocate_fee` across groups → one savepoint: `checkout_repository.insert` (status `PENDING`, `expires_at = now + hold_duration_minutes`), then per item `expire_lapsed` + `booking_repository.insert` (group `PENDING/UNPAID`, same `expires_at`, `checkout_id`) → commit. On `SLOT_UNAVAILABLE` the savepoint rolls back, then `occupancy` over the requested pairs builds the detail: `"These hours were just taken: Court 1 · Wed 29 Oct, 6 PM; …"` (Manila), code `SLOT_UNAVAILABLE`, 409.

- [ ] **Step 1: Write failing tests** in `tests/checkouts/test_create.py`:
  - `test_one_payment_covers_two_courts_on_two_days` — items on `a.court` day+1 10:00 ×2 and `a.court2` day+2 18:00 ×1 → 201 with **two** bookings (one per item), all `PENDING`, all `expires_at == checkout.expires_at`; `total == price + fee`; sum of group fees == checkout fee.
  - `test_hold_lasts_hold_duration_minutes` — `expires_at - created_at` ≈ 5 min (± 5 s).
  - `test_two_venues_in_one_payment_is_refused` (scenario 16) → 422 `MIXED_VENUES`, nothing inserted.
  - `test_second_customer_is_told_the_hour_is_taken` (scenario 2) — customer 1 creates; customer 2 asks for the same hour plus a free one → 409 `SLOT_UNAVAILABLE`, detail contains `"Court"` and the hour; customer 2 holds **nothing** (the free hour stays `AVAILABLE` in `/availability`).
  - `test_hold_cap_counts_checkouts_not_courts` — with `max_active_holds_per_player=1`, a checkout covering two courts succeeds; a second checkout → 409 `HOLD_LIMIT`.
  - `test_lapsed_checkouts_do_not_count_toward_the_cap` — lapse it (update `expires_at` to the past on checkout and groups) → a new checkout succeeds. **Mutation check:** this test must fail if `count_live` drops `expires_at > now`.
  - `test_zero_lead_books_the_current_hours_next_hour` — `min_lead_minutes=0`, starting at the next whole hour → 201; an hour that already started → 422 `TOO_SOON`.
  - `test_midnight_hour_is_priced_on_its_manila_day` (scenario 17) — a price rule for Saturday hour 0 only; book Saturday 00:00 Manila → that rule's price, booking listed under Saturday in `/availability?date=<Saturday>`.
  - `test_someone_elses_checkout_is_404`.
  - `test_list_is_newest_first_and_paginated` — envelope keys `data,total,page,limit,has_more`.
  - `test_availability_reports_the_fee_percent` — `convenience_fee_percent == "3.00"`.
- [ ] **Step 2: Run** `uv run pytest tests/checkouts -q` → FAIL.
- [ ] **Step 3: Implement** repository, service, DTOs, router; register the router in `main.py`; delete the player router and the dead service/repository/rule functions listed above; rewrite the `hold` helper. Update `tests/bookings/test_holds.py` to cover only what still exists (`expire_lapsed`, occupancy filters) and port anything about `/bookings` creation to checkouts.
- [ ] **Step 4: Run** the whole suite: `uv run pytest -q` → PASS; `uv run vulture` clean.
- [ ] **Step 5: Commit** `Create customer checkouts across courts and retire the per-booking routes`.

---

### Task 4: Provider port — void, refund, Maya statuses; mock outcomes

**Files:**
- Modify: `src/payments/domain/port.py`, `src/payments/adapters/maya_wire.py`, `src/payments/adapters/mock.py`, `src/payments/controllers/mock_maya.py`
- Test: `tests/payments/test_maya_wire.py`, `tests/payments/test_mock_provider.py`

**Interfaces:**
- Produces:
  - `ProviderPayment` gains `provider_status: str` (Maya's wording) and `paid_at: datetime | None`; `status: PaymentOutcome`.
  - `PaymentProvider.void(payment_id: str) -> str` and `refund(payment_id: str, amount_centavos: int) -> str` — return the provider's reference; raise `ProviderError` on failure.
  - `maya_wire.MAYA_STATUSES` exactly the ADR table: `PENDING_TOKEN, PENDING_PAYMENT, FOR_AUTHENTICATION, AUTHENTICATING, AUTH_SUCCESS, PAYMENT_PROCESSING → PENDING`; `PAYMENT_SUCCESS → SUCCESSFUL`; `PAYMENT_FAILED, AUTH_FAILED → FAILED`; `PAYMENT_CANCELLED, PAYMENT_EXPIRED → CANCELLED`; `VOIDED, REFUNDED → REFUNDED`. Unknown → `PENDING` and `parse_payment` logs the unknown wording once (no payload).
  - `parse_payment` reads `paid_at` from `paymentAt` (ISO 8601) when present.
  - `MockProvider.settle(checkout_id, maya_status, *, paid_at: datetime | None = None)` (sets `paymentAt` on success, default now); `fail_undo: bool = False` attribute; `void`/`refund` set the Maya status to `VOIDED`/`REFUNDED`, record `undone[payment_id] = (RefundKind, reference)`, and raise `ProviderError` when `fail_undo`.
  - Mock page: four buttons **Pay, Decline, Back out, Expire** → outcomes `success, failure, cancel, expire` (routes unchanged).

- [ ] **Step 1: Write failing tests:** `test_every_adr_maya_status_maps` (parametrised over the table above), `test_unknown_status_stays_pending`, `test_payment_at_is_parsed`; `test_mock_void_marks_voided_and_records_it`, `test_mock_refund_records_amount`, `test_mock_undo_failure_raises_provider_error`; `test_mock_page_offers_four_outcomes` (GET the page with `mock_checkout_enabled`, assert the four labels).
- [ ] **Step 2: Run** `uv run pytest tests/payments/test_maya_wire.py tests/payments/test_mock_provider.py -q` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `Give the provider port void and refund, and the mock every outcome`.

---

### Task 5: Continue to Maya and Cancel

**Files:**
- Modify: `src/checkouts/service.py`, `src/checkouts/controllers/customer.py`, `src/payments/service.py` (`start_checkout`), `src/payments/persistence/repository.py`
- Test: `tests/checkouts/test_pay_and_cancel.py`

**Interfaces:**
- Consumes: Task 3 service, Task 4 port.
- Produces:
  - `payment_repository.add_checkout(session, *, checkout_id, provider, session_: CheckoutSession, amount_centavos) -> PaymentModel` (stores `checkout_url`), `latest_for_checkout(session, checkout_id) -> PaymentModel | None`, `by_provider_id(session, provider, payment_id) -> PaymentModel | None` (FOR UPDATE).
  - `payment_service.start_checkout(session, checkout, provider) -> str` — if a payment row already exists for the checkout, return its `checkout_url` without calling the provider; else create the Maya session (`amount = checkout.total_centavos`, `reference = checkout.reference_code`, `return_url = f"{frontend_base_url}/book/result/{checkout.id}"`), store it, commit, return the URL.
  - `checkout_service.pay(session, customer, checkout_id, provider) -> CheckoutModel` — 409 `HOLD_ENDED` ("Your 5-minute hold has ended. Pick your hours again.", minutes from the parameter) unless `is_live`.
  - `checkout_service.cancel(session, customer, checkout_id) -> CheckoutModel` — `SUCCESSFUL` → 409 `PAID_IS_FINAL` ("Paid bookings are final."); not live → 409 `HOLD_ENDED`; else checkout `CANCELLED`, every group `CANCELLED` via `booking_repository.set_status` with `cancelled_at=now, cancelled_by_user_id=customer.id, cancellation_reason="Cancelled by customer"`, the pending payment row (if any) `CANCELLED`; commit.
  - Routes: `POST /checkouts/me/{id}/pay`, `POST /checkouts/me/{id}/cancel` → `CheckoutOut`.

- [ ] **Step 1: Write failing tests:**
  - `test_pay_returns_a_maya_url_for_the_checkout_total` — one `payments` row, `amount_centavos == total`, mock payment's `requestReferenceNumber == checkout.reference_code`, URL in the response's `checkout_url`.
  - `test_pay_twice_reuses_the_same_maya_session` (Review Focus 1) — same URL, still one payments row, provider `create_checkout` called once.
  - `test_pay_after_the_hold_ended_is_refused` (Review Focus 4) → 409 `HOLD_ENDED`, no payments row.
  - `test_cancel_releases_every_hour_at_once` (design: "Cancel and release hours") — status `CANCELLED`, all groups `CANCELLED` with reason, hours `AVAILABLE` in `/availability`.
  - `test_cancel_a_paid_checkout_is_refused` (scenario 12) — after a successful pay (use the Task 7 helper once it exists; until then set statuses directly in the test) → 409 `PAID_IS_FINAL`, nothing changes.
  - `test_cancel_after_the_hold_ended_is_refused` → 409 `HOLD_ENDED`.
- [ ] **Step 2: Run** `uv run pytest tests/checkouts/test_pay_and_cancel.py -q` → FAIL.
- [ ] **Step 3: Implement.** The provider call happens after the checkout row is committed (Task 3) and outside any lock.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `Open Maya for a held checkout once, and let the customer release it`.

---

### Task 6: Receipt email per checkout

**Files:**
- Modify: `src/email/templates.py`
- Test: `tests/email/test_templates.py` (create if absent; else the existing receipt test)

**Interfaces:**
- Produces: `@dataclass(frozen=True) class ReceiptLine: court: str; starts_at_manila: datetime; hours: int; price_centavos: int`; `receipt_email(*, to, first_name, reference, venue, lines: Sequence[ReceiptLine], fee_centavos, total_centavos) -> EmailMessage`. Subject unchanged: `"Your Pikol receipt — {reference}"`. Body lists each line (`"Court 1 — Wed 29 Oct 2026, 06:00 PM, 2 h — ₱1,000.00"`), the fee, the total, and "Paid bookings are final."

- [ ] **Step 1: Failing test** `test_receipt_lists_every_court_and_the_total` — two lines → both court names and `"₱1,545.00"` in the body; subject has the reference.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** with Task 7 (its only caller), not alone — vulture would flag an unused parameter shape otherwise.

---

### Task 7: `sync_payment` around the checkout, with the simple refund flow

**Files:**
- Modify: `src/payments/service.py`, `src/checkouts/controllers/customer.py` (refresh route), `src/payments/controllers/mock_maya.py` (unchanged path, new statuses)
- Delete/replace: `tests/payments/test_late_payment.py`, `tests/payments/test_mock_flow.py`, `tests/payments/test_webhook.py` → `tests/checkouts/test_payment_outcomes.py` (keep webhook-specific tests — allowlist, bad body, 502 — in `test_webhook.py`)
- Modify: `tests/payments/helpers.py`

**Interfaces:**
- Consumes: Tasks 2–6.
- Produces:
  - `payment_service.sync_payment(session, provider, payment_id, sender, *, event_payload=None) -> None` — same signature; event id `"{payment_id}:{provider_status}"`; `PENDING` outcome records nothing.
  - Route `POST /checkouts/me/{id}/refresh-payment` (`10/minute`) → `CheckoutOut`: syncs the latest attempt if any, then re-reads.
  - Helpers: `pay(client, session, provider, checkout: dict, maya_status="PAYMENT_SUCCESS", *, paid_at=None)` — settles the checkout's attempt (calling `/pay` first if no attempt exists) and posts the webhook; `lapse(session, checkout: dict)` — moves the checkout's and its groups' `expires_at` 20 minutes into the past.

`_apply(session, provider, payment)` on the checkout locked `FOR UPDATE` (then its groups locked `of=BookingGroupModel`). Always stamps the payment row's `provider_status`, `provider_payment_id`, `raw_payload`. Then by `payment.status`:

| Outcome | Checkout now | Action |
|---|---|---|
| `FAILED` | `PENDING` | checkout + payment `FAILED`; still-`PENDING` groups → `CANCELLED` (`cancelled_by` = customer, reason `"Payment failed"`); groups already `EXPIRED` stay |
| `CANCELLED` | `PENDING` | checkout + payment `CANCELLED`; still-`PENDING` groups → `CANCELLED` if the hold is live (reason `"Backed out of payment"`), else `EXPIRED` |
| `FAILED`/`CANCELLED` | anything else | payment row only; checkout unchanged |
| `SUCCESSFUL` | any, but amount ≠ total or reference ≠ checkout's | flag every group (`needs_resolution`, `[system]` note); nothing else |
| `SUCCESSFUL` | `PENDING` and live | every group `CONFIRMED/PAID/ONLINE`; checkout + payment `SUCCESSFUL`; receipt after commit |
| `SUCCESSFUL` | `PENDING` lapsed, or `CANCELLED` | **re-acquire** if every slot of every group starts after `now`: one savepoint, per group `expire_lapsed` then `set_status(CONFIRMED, PAID, ONLINE, cancelled_at=None, cancelled_by_user_id=None, cancellation_reason=None)`; on `SLOT_CONSTRAINT` or any started slot → **refund flow** |
| `SUCCESSFUL` | `FAILED`, `REFUNDED` or `SUCCESSFUL` | flag every group ("payment after a final status"); nothing else |
| `REFUNDED` | any | checkout + payment `REFUNDED` if a `refunds` row exists for the payment; else flag every group ("refund we did not make") |

**Refund flow** (scenarios 8–11): (1) in the current transaction: still-`PENDING` groups → `EXPIRED`; checkout + payment status stay as they are, every group flagged with `"[system] refund pending for {payment_id}"`; commit. (2) Outside the transaction: `kind = undo_kind(payment.paid_at or now, now)`; call `provider.void(id)` or `provider.refund(id, payment.amount_centavos)`. (3) On success: insert `RefundModel`, checkout + payment `REFUNDED`, clear the flags and replace the note with `"[system] {kind} {reference} — paid after the hold ended and an hour was taken"`; commit. On `ProviderError`: leave the flags, append `"[system] {kind} failed — refund by hand"`, commit, log a warning (id + outcome only). No booking is ever confirmed on this path.

- [ ] **Step 1: Write failing tests** (`tests/checkouts/test_payment_outcomes.py`), one per ADR scenario not already covered:
  - S1 `test_paid_within_the_hold_confirms_every_booking` — two courts; `SUCCESSFUL`; groups `CONFIRMED/PAID`; one receipt email listing both courts.
  - S3 `test_backing_out_frees_the_hours_at_once` — `PAYMENT_CANCELLED` → `CANCELLED`; `/availability` shows them `AVAILABLE` before any expiry.
  - S4 `test_decline_fails_and_frees_the_hours` — `PAYMENT_FAILED` → `FAILED`; another customer can create a checkout for the same hour immediately.
  - S4b `test_auth_failed_is_a_failure`.
  - S5 `test_after_a_failure_pay_now_makes_a_new_checkout` — same customer creates again → new checkout id, new reference, `PENDING`; the failed one stays `FAILED`.
  - S6 `test_hold_end_reads_cancelled_and_maya_expiry_changes_nothing` — lapse → `GET` shows `CANCELLED`, hour `AVAILABLE`; then `PAYMENT_EXPIRED` webhook → still `CANCELLED`, no group touched beyond `EXPIRED`.
  - S7 `test_late_payment_with_free_hours_is_booked` — lapse, pay → `SUCCESSFUL`, groups `CONFIRMED/PAID`.
  - S8 `test_late_payment_with_a_taken_hour_is_voided_same_day` — lapse; customer 2 takes the hour; customer 1 pays → `REFUNDED`, `refund_kind == "VOID"`, groups `EXPIRED`, no flag, customer 2's booking untouched, `provider.undone` has the void.
  - S9 `test_one_taken_hour_refunds_the_whole_payment` — two courts, only one taken → `REFUNDED`; **neither** group confirmed; the free hour is `AVAILABLE`.
  - S10 `test_late_payment_after_midnight_is_refunded_not_voided` — `paid_at` = yesterday Manila → `refund_kind == "REFUND"`, amount == total.
  - S11 `test_void_failure_is_flagged_for_staff` — `provider.fail_undo = True` → checkout not `REFUNDED`; every group `needs_resolution`; appears in `GET /admin/bookings/unfulfilled` for venue A's staff.
  - S13 `test_duplicate_webhook_applies_once` — post the same webhook twice → one `webhook_events` row, one receipt.
  - S14 `test_body_claiming_success_is_not_trusted` — mock still `PENDING_PAYMENT`, webhook body forged with `paymentStatus: PAYMENT_SUCCESS` → `PENDING`, no event row.
  - S15 `test_failure_redirect_still_shows_successful` — pay success with `deliver_webhook=false`, then `refresh-payment` → `SUCCESSFUL` (the redirect target is the same page; the status comes from the re-fetch).
  - Review Focus 2 `test_cancel_then_payment_lands_is_rebooked_if_free` and `…_is_refunded_if_taken`.
  - Review Focus 3 `test_late_payment_after_an_hour_started_is_refunded` — slot moved into the past, still free → `REFUNDED`.
  - `test_amount_mismatch_is_flagged_not_confirmed`.
  - `test_refresh_payment_is_rate_limited` — 11th call in a minute → 429.
- [ ] **Step 2: Run** `uv run pytest tests/checkouts tests/payments -q` → FAIL.
- [ ] **Step 3: Implement** `_apply` and the refund flow as tabled; keep `_flag`; receipts use Task 6. Remove any `xfail` left from Task 1.
- [ ] **Step 4: Run** the full gate → PASS.
- [ ] **Step 5: Commit** `Settle payments per checkout, voiding or refunding late ones that lost an hour` (includes Task 6).

---

### Task 8: Favorite venues API

**Files:**
- Modify: `../docs/adr/pikol.dbml` (first: `favorite_venues`)
- Create: `src/database/migrations/versions/0009_favorite_venues.py`, `src/favorites/persistence/models.py`, `src/favorites/persistence/repository.py`, `src/favorites/controllers/me.py`
- Modify: `src/main.py`
- Test: `tests/favorites/test_favorites.py`

**Interfaces:**
- Produces: table `favorite_venues (user_id → users CASCADE, venue_id → venues CASCADE, created_at)`, PK `(user_id, venue_id)`. Routes, all `require_customer`: `GET /users/me/favorite-venues` → `list[VenueOut]` (the catalog's shape, active venues only, newest favorite first); `PUT /users/me/favorite-venues/{venue_id}` → 204, idempotent, 404 for a missing or inactive venue; `DELETE /users/me/favorite-venues/{venue_id}` → 204, idempotent.
- [ ] **Step 1: Failing tests:** add → listed; add twice → one row; remove → gone; remove absent → 204; unknown venue → 404; a non-customer → 403; another customer's favorites are not visible.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement** (`INSERT … ON CONFLICT DO NOTHING`). **Step 4: Run** gate → PASS.
- [ ] **Step 5: Commit** parent dbml, then backend `Add favorite venues for the mobile app`.

---

### Task 9: Demo seed and e2e customers

**Files:**
- Create: `src/devtools/demo_seed.py`
- Modify: `src/devtools/e2e_seed.py`
- Test: `tests/devtools/test_demo_seed.py`, `tests/devtools/test_e2e_seed.py`

**Interfaces:**
- Produces: `python -m src.devtools.demo_seed --customers N` (default 3) → `customer1..N@example.com`, password `Demo-password-1`, verified, active, in the Customer group; prints the logins and password. `assert_demo_database(url: str) -> None` exits 2 when the database name ends in `_e2e` or the host contains `supabase`. Idempotent: existing accounts are skipped (and re-added to the Customer group if missing). `seed_customers(session, n) -> list[str]` returns the emails.
- e2e seed adds `customer1.e2e@example.com`, `customer2.e2e@example.com` (password `E2e-password-1`) in the Customer group.
- [ ] **Step 1: Failing tests:** `test_refuses_e2e_database`, `test_refuses_supabase_host`, `test_seeds_customers_once` (run twice → N users, each `is_customer`), e2e seed test asserts the two customers exist and are customers.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** gate → PASS.
- [ ] **Step 5: Commit** `Seed demo and e2e customers`.

---

### Task 10: Backend brief

**Files:** Modify `pikol-backend/CLAUDE.md`, `vulture_whitelist.py` if routes need it.
- [ ] Update "What exists now" (checkouts module, favorites, demo seed, port's `void`/`refund`, removed player routes), "Payment rules in practice" (the outcome table's one-line summary, refund flow commits before the provider call, `checkouts.status` has one writer), "Booking rules in practice" (hold cap counts checkouts), "Testing" (`hold` helper returns a booking plus `checkout_id`; `pay`, `lapse`; `provider.fail_undo`).
- [ ] Run gate, commit `Document checkouts and the refund flow`, push backend.

---

## Frontend

### Task 11: Booking slice foundation — schemas, API, picks model

**Files:**
- Create: `src/features/booking/{schemas.ts,api.ts,keys.ts,picks.ts,index.ts}`, `src/features/booking/hooks/use-checkouts.ts`
- Modify: `src/features/availability/schemas.ts` (`convenience_fee_percent: z.string()`), `src/features/catalog/{api,schemas,keys,index}.ts` + `hooks/use-venues.ts` (`GET /venues`)
- Test: `tests/unit/booking/picks.test.ts`, `tests/unit/booking/api.test.ts`

**Interfaces:**
- Produces:
  - `CheckoutStatusSchema = z.enum(['PENDING','SUCCESSFUL','FAILED','CANCELLED','REFUNDED'])`; `CheckoutSchema` mirroring `CheckoutOut` (Task 3) exactly, snake_case.
  - `checkoutApi.create(items)`, `.list(page)`, `.get(id)`, `.pay(id)`, `.cancel(id)`, `.refresh(id)` — each zod-parsed.
  - `checkoutKeys.all/list(page)/detail(id)`.
  - Hooks: `useCheckout(id, { poll?: boolean })`, `useMyCheckouts(page)`, `useCreateCheckout()`, `usePay()`, `useCancelCheckout()`, `useRefreshPayment()` — mutations write the detail into the cache and invalidate the list.
  - `picks.ts`: `type Pick = { court_id: number; court_name: string; day: string; starts_at: string; price_centavos: number }`; `MAX_CHECKOUT_HOURS = 24`; `togglePick(picks, pick): Pick[]` (keyed by `court_id`+`starts_at`; unchanged when adding past 24); `toItems(picks): { court_id; starts_at; hours }[]` (consecutive hours on one court merge, ≤ 12 per item, sorted by court then time; `starts_at` is the server's string); `feeCentavos(subtotal: number, percent: string): number` (half-up, integer arithmetic on the percent's digits — same results as the backend's `convenience_fee_centavos`); `summarize(picks, percent): { hours; courts; subtotal; fee; total }`; `byDay(picks): { day: string; picks: Pick[] }[]` (sorted); `dropUnavailable(picks, grid): Pick[]` (removes that grid day's picks whose cell is no longer `AVAILABLE`).
- [ ] **Step 1: Failing tests:** toggle on/off; 25th pick ignored; `toItems` merges 10,11,12 on court 1 into one 3-hour item and keeps court 2 separate; a 13-hour run splits into 12 + 1; `feeCentavos(50000,'3.00') === 1500`, `feeCentavos(16650,'3.00') === 500` (half-up), `feeCentavos(16649,'3.00') === 499`; `summarize` over picks on two days and two courts; `byDay` order; `dropUnavailable` keeps other days' picks. API test: `create` posts `{items}` to `/checkouts/me` and parses the fixture.
- [ ] **Step 2: Run** `npm test -- booking` → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `Add the customer booking slice's model and API`.

### Task 12: Customer shell, landing, venue cards, the courts × hours grid

**Files:**
- Create: `src/features/booking/components/{customer-shell.tsx,customer-gate.tsx,venue-cards.tsx,court-hour-grid.tsx,pick-bar.tsx,venue-booking.tsx}`; `src/app/(app)/book/{layout.tsx,page.tsx}`, `src/app/(app)/book/[venueId]/page.tsx`
- Modify: `src/features/auth/landing.ts` and every caller; delete `src/app/(app)/customer-app/`
- Test: `tests/unit/auth/landing.test.ts`, `tests/unit/booking/venue-booking.test.tsx`

**Interfaces:**
- Produces: `isStaff(permissions: readonly string[]): boolean` (`BOOKING:ReadAny`); `landingPath(permissions: readonly string[], isCustomer: boolean): '/book' | '/staff/schedule'` — a customer → `/book` (even staff who are customers); staff who are not customers → `/staff/schedule`; anyone else → `/book`, where `CustomerGate` explains. `StaffOnly` uses `isStaff` and sends non-staff to `/book` (it used to send them to `/customer-app`; landing on the staff route would ping-pong). Callers: `app/page.tsx`, `guest-gate.tsx`, `login-form.tsx` pass `user.is_customer`.
- `CustomerGate`: if `!user.is_customer` → "This account can't book courts" + sign out. `CustomerShell`: plain top bar — "Pikol" (link `/book`), "My bookings" (`/book/mine`), "Sign out".
- `/book`: `PageHeader` "Book a court"; `VenueCards` — one card per active venue (name, address if present), link `/book/{id}`.
- `/book/[venueId]?date=`: `VenueBooking` holds `picks` in state (remounts per venue, so switching venue clears them); `DateStrip` for today … +`booking_horizon_days` (14 days; the strip length is a constant here, the server enforces the real horizon); `CourtHourGrid` — rows = hours 0–23 (Manila), columns = courts; `AVAILABLE` cells show the price and are buttons (`aria-pressed` when picked), `BOOKED`/`CLOSED` are hatched (`bg-hatch`) and labelled "Taken"/"Closed", `PAST` muted "—". `PickBar` (sticky black bar, hidden with no picks): `"{hours} hours · {courts} courts · {formatPeso(subtotal)} + {formatPeso(fee)} fee"`, picks grouped by day, **Pay now** (disabled while the mutation is pending). On success → `router.push('/book/checkout/{id}')`. On 409 `SLOT_UNAVAILABLE`/`HOLD_LIMIT` → `notify.error(detail)`, refetch availability, `dropUnavailable`.
- [ ] **Step 1: Failing tests:** landing — customer staff → `/book`, staff only → `/staff/schedule`, neither → `/book`; `StaffOnly` sends a non-staff user to `/book`; `CustomerGate` shows "This account can't book courts" for `is_customer: false`. VenueBooking (mock hooks): tapping two free cells on two courts shows `"2 hours · 2 courts"`; switching date keeps the bar's earlier picks under their day heading; a hatched cell is not a button; **Pay now** is disabled while pending (Review Focus 5); a 409 shows the detail via `notify.error` and removes the taken pick.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** `npm run lint:ci && npm run typecheck && npm test && npm run deadcode` → PASS.
- [ ] **Step 5: Commit** `Add the customer booking home and the courts-by-hours grid`.

### Task 13: Hold page

**Files:**
- Create: `src/features/booking/components/{hold-page.tsx,countdown.tsx}`, `src/features/booking/hooks/use-seconds-left.ts`, `src/app/(app)/book/checkout/[id]/page.tsx`
- Test: `tests/unit/booking/hold-page.test.tsx`

**Interfaces:**
- Produces: `useSecondsLeft(expiresAt: string): number` (ticks each second, floors at 0); `formatCountdown(seconds): string` (`"4:52"`).
- Copy (exact): heading "Pay within {m:ss}"; warning "Pay within 5 minutes. After that the hours reopen for everyone, and a late payment is voided." (minutes from the checkout's own `expires_at - created_at`, rounded); line items grouped by day: "Court 1 · 6 PM – 8 PM · ₱1,000"; "Convenience fee"; "Total"; buttons **Continue to Maya** (POST pay → `window.location.assign(checkout_url)`) and **Cancel and release hours** (POST cancel → `/book/result/{id}`). If the checkout is not `PENDING`, `router.replace('/book/result/{id}')`. At 0: both buttons disabled, text "Your hold ended. These hours are open to everyone again." and a "Book again" link to `/book/{venue_id}` (Review Focus 4).
- [ ] **Step 1: Failing tests:** renders countdown from a fixture 292 s out as `4:52`; line items per day and total; Continue calls `pay` then assigns the URL; at 0 both buttons are disabled and the ended text shows; a non-pending fixture redirects.
- [ ] **Step 2–4:** run → FAIL, implement, run gate → PASS.
- [ ] **Step 5: Commit** `Add the hold page with its countdown`.

### Task 14: Result page and My bookings

**Files:**
- Create: `src/features/booking/components/{result-view.tsx,my-bookings.tsx,status-pill.tsx}`, `src/app/(app)/book/result/[id]/page.tsx`, `src/app/(app)/book/mine/page.tsx`
- Test: `tests/unit/booking/result-view.test.tsx`, `tests/unit/booking/my-bookings.test.tsx`

**Interfaces:**
- `useCheckout(id, { poll: true })`: `refetchInterval` 3000 ms while `status === 'PENDING'` and less than 120 s since mount, else `false`.
- Result copy (exact):
  - `PENDING`: "Confirming your payment…", "This page updates by itself." and **Check again** (calls `refresh`).
  - `SUCCESSFUL`: "Payment successful", reference code, line items, "Paid bookings are final."
  - `FAILED`: "Payment failed", "Your payment didn't go through, so the hours were released.", **Book again** → `/book/{venue_id}`.
  - `CANCELLED`: "Payment cancelled", "The hours are free again.", **Book again**.
  - `REFUNDED`: "Paid too late — fully refunded", "Your hold had ended and an hour was taken, so nothing was booked. The full amount was {voided|refunded}."
- `StatusPill`: `Pill` + text label per status (shape + text, never colour alone).
- My bookings: `PageHeader` "My bookings"; one row per checkout — pill, reference, venue, hours count, total; a `PENDING` row shows its countdown and **Continue to Maya** (pay → assign URL). Paginated with the list envelope.
- [ ] **Step 1: Failing tests:** each status renders its exact heading; polling stops after 120 s (fake timers: no fetch at 123 s); **Check again** calls `refresh`; my bookings shows a pending row's countdown and button, and no button on a successful one.
- [ ] **Step 2–4:** run → FAIL, implement, run gate → PASS.
- [ ] **Step 5: Commit** `Add the payment result page and my bookings`.

### Task 15: Playwright — the customer flow on the real stack

**Files:**
- Modify: `playwright.config.ts` (`MOCK_CHECKOUT_ENABLED: 'true'`), `e2e/fixtures.ts` (`customer1Page`, `customer2Page`, `CUSTOMER1`, `CUSTOMER2`, `setParameter(request, key, value)` as owner)
- Create: `e2e/booking.spec.ts`

Each spec books an hour on **E2E Lakeside**, two days out, at a distinct hour, so specs don't collide on the shared database. Mock buttons are on the backend's page: click by role name.
- [ ] `book and pay` — customer1: `/book` → venue → pick 2 cells on 1 court → Pay now → hold page shows `Pay within 4:` → Continue to Maya → **Pay** → result "Payment successful" → My bookings lists it as Successful.
- [ ] `two customers race for one hour` — customer1 holds hour H; customer2 picks H (already open page) → Pay now → snackbar names the taken hour; customer2's grid shows H hatched.
- [ ] `decline releases the hours` — Continue → **Decline** → "Payment failed"; customer2 can then hold the same hour.
- [ ] `back out releases the hours` — Continue → **Back out** → "Payment cancelled"; hour available again.
- [ ] `pay late, hour still free` — owner sets `hold_duration_minutes=1` (restored to `5` in `finally`); customer1 Continue → wait 65 s → **Pay** → "Payment successful".
- [ ] `pay late, hour taken` — as above, but customer2 books and pays H during the wait → customer1 **Pay** → "Paid too late — fully refunded".
- [ ] `my bookings shows each status` — after the above, customer1's list shows Successful, Failed, Cancelled and Refunded pills.
- [ ] Run `npm run e2e` → all specs pass (old and new). Use the browser to look at `/book/[venueId]` at 390 px and desktop widths (memory: verify frontend in a browser).
- [ ] Commit `Cover the customer booking flow end to end`.

### Task 16: Docs and push

**Files:** `pikol-frontend/CLAUDE.md`, `../CLAUDE.md`, `../docs/standard/cleanup-pass.md` §5 (catalogue: `StatusPill` if it qualifies; otherwise nothing)
- [ ] Frontend brief: the web now has the customer booking pages (`/book…`), the `booking` slice row, the landing rule, e2e customers, mock checkout enabled in e2e.
- [ ] System brief: Current state (customers can book on the web demo; Maya adapter still deferred), checkout vocabulary under "Cross-cutting concepts" (a checkout is the unit of payment; a booking group stays the unit of occupancy; five payment statuses; paid bookings are final), the authoritative-docs list gains this plan.
- [ ] Gates green in both repos; commit each repo separately; push backend, frontend, parent.

---

## Self-review notes

- Spec coverage: goals (Tasks 3–7, 11–15), ADR scenarios 1–17 (2,16,17 → T3; 12 → T5; 1,3–11,13–15 → T7), favorites (T8), parameters (T1), demo seed (T9), mock outcomes (T4), polling (T14), Playwright list (T15).
- Calls this plan makes that the design left open: fee is computed on the checkout subtotal and allocated across groups (so the bar's total equals the charge); the availability read carries `convenience_fee_percent` so the bar can show the fee before Pay now; `booking_horizon_days` becomes 14 per the design's "Parameters" line; an account that is neither customer nor staff lands on `/book`, whose gate says it cannot book (replacing `/customer-app`).
