# Phase 3 — Venues, Courts, Pricing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The owner creates venues; managers create courts and set wildcard
price rules on their own venue's courts; players browse active venues and
courts; price resolution returns the right price for any hour.

**Architecture:** `venues` and `courts` are thin (router → service → model).
`pricing` is rich (spec §3.4): a pure `PriceResolver` with no SQLAlchemy
imports, unit-tested without a database, plus a thin persistence/admin layer
for `price_rules`. One migration (`0003_courts_pricing`) adds `courts` and
`price_rules` with Appendix A §4 and §5 verbatim.

**Tech Stack:** as Phase 2. No new dependencies.

**Spec:** [`2026-09-09-court-booking-system-design.md`](./2026-09-09-court-booking-system-design.md)
§4.4, §6.2, §7.4 (Catalog and Admin-setup rows). Schema:
[`../adr/pikol.dbml`](../adr/pikol.dbml) `courts`, `price_rules`, Appendix A
§4–5.

## Global Constraints

Everything in Phase 2's Global Constraints, plus:

- Money is integer centavos; `price_centavos` is `0 ≤ n ≤ 10_000_000` (₱100,000) at the API, `≥ 0` in the database.
- `day_of_week` is **0 = Sunday … 6 = Saturday**, `hour_of_day` 0–23, both **Manila-local**. `NULL` is a wildcard.
- Resolution is most-specific-wins: `(day, hour) → (hour) → (day) → (NULL, NULL) → parameters.default_slot_price_centavos`.
- `uq_price_rule_specificity` is `UNIQUE NULLS NOT DISTINCT (court_id, day_of_week, hour_of_day)` — Appendix A §4 verbatim.
- Courts and venues are never deleted, only deactivated (`is_active`) — bookings will reference them with `ON DELETE RESTRICT`.
- Every repository method touching venue-owned data takes `scope: VenueScope` keyword-only with no default. The catalog passes `VenueScope.everything()` **explicitly**, so "unscoped" is a visible decision, never an omission.

## Decisions this plan makes

| Decision | Why |
|---|---|
| `VenueScope.everything()` for catalog reads | Keeps "scope is mandatory" true for the public catalog too |
| Catalog hides inactive venues and courts; an inactive venue's id → 404 | A player must not book a closed venue |
| A price rule's specificity is immutable: `PATCH` changes price only | Changing day/hour is a different rule; delete + create keeps the unique index the only arbiter |
| Duplicate specificity → `409 PRICE_RULE_EXISTS` | The NULLS NOT DISTINCT index is the check — no SELECT-then-INSERT |
| `GET /admin/courts/{id}/prices?date=` returns 24 resolved prices | The phase deliverable ("right price for any hour") observable through the API; the pricing screen previews with it |
| The convenience fee function lands here (pure, in `pricing`) | §4.4 pricing domain; Phase 4 consumes it |

## Review Focus

1. **12am–1am Manila** — a Saturday `hour_of_day = 0` rule must price `Fri 16:00Z`, not `Sat 16:00Z`. Test in Task 1.
2. **Wildcard collisions** — two `(NULL, NULL)` rules on one court must 409, not coexist. Test in Task 2.
3. **Cross-venue court ids** — a manager `PATCH`ing or pricing another venue's court gets 404, same body as a missing id. Test in Task 3 and Task 4.
4. **Inactive venue in the catalog** — `GET /venues/{id}/courts` for a deactivated venue is 404. Test in Task 3.
5. **Naive datetimes** — `PriceResolver.price_at` with a naive datetime raises instead of silently treating it as UTC. Test in Task 1.

---

### Task 1: The pricing domain (pure)

**Files:** Create `src/pricing/domain/pricing.py`. Test `tests/pricing/test_pricing_domain.py`.

**Interfaces — Produces:**
- `@dataclass(frozen=True) PriceRule(day_of_week: int | None, hour_of_day: int | None, price_centavos: int)`.
- `class PriceResolver: __init__(self, rules: Iterable[PriceRule], default_centavos: int)`; `price_at(self, starts_at: datetime) -> int` — converts to Manila, `day = isoweekday() % 7`, looks up the four keys in order, else default. Naive datetime → `ValueError`. Built once per court and reused for every cell (spec §7.2).
- `convenience_fee_centavos(price_centavos: int, percent: Decimal) -> int` — `ROUND_HALF_UP` to whole centavos.

- [ ] **Tests:** each fallback level wins over the next (four tests on the §4.4 table: flat ₱700, Saturday ₱800, 6pm ₱600, Saturday 6pm ₱900, default ₱500); `test_saturday_midnight_rule_prices_friday_1600_utc`; `test_naive_datetime_rejected`; fee: `50000 @ 3.00 → 1500`, `33333 @ 3.00 → 1000` (999.99 rounds up), `50 @ 3.00 → 2` (1.5 rounds half up), `x @ 0 → 0`.
- [ ] Fail → implement → gate → commit `Add the pure pricing domain`.

### Task 2: `courts` and `price_rules` schema

**Files:** Create `src/courts/persistence/models.py`, `src/pricing/persistence/models.py`, `src/database/migrations/versions/0003_courts_pricing.py`. Modify `env.py`. Test `tests/pricing/test_pricing_schema.py`.

**Interfaces — Produces:** `CourtModel` and `PriceRuleModel`, columns per dbml; `courts.venue_id → venues.id RESTRICT`, `price_rules.court_id → courts.id CASCADE`, audit FKs to users. Constraints named exactly `uq_price_rule_specificity`, `ck_price_rule_ranges`, `ck_price_rule_non_negative`. Index `ix_courts_venue_id_sort_order`. Declared on the models too, so `alembic check` reports no drift.

- [ ] **Tests:** `test_duplicate_wildcard_rule_rejected` (two `(NULL, NULL)` rows on one court → `IntegrityError` naming `uq_price_rule_specificity`); `test_same_specificity_allowed_on_different_courts`; `test_hour_out_of_range_rejected` (`ck_price_rule_ranges`); `test_negative_price_rejected`.
- [ ] Up → down → up; `alembic check` clean; gate; commit `Add courts and price rules schema`.

### Task 3: Venues and courts — admin and catalog

**Files:** Create `src/venues/{dto,service}.py`, `src/venues/persistence/repository.py`, `src/venues/controllers/{admin,catalog}.py`, `src/courts/{dto,service}.py`, `src/courts/persistence/repository.py`, `src/courts/controllers/admin.py`. Modify `src/auth/scope.py`, `src/main.py`. Test `tests/venues/test_venues.py`, `tests/courts/test_courts.py`.

**Interfaces:**
- `VenueScope.everything() -> VenueScope` (classmethod).
- `VenueOut {id, name, address, is_active}`; `CourtOut {id, venue_id, name, sort_order, is_active}`.
- Admin venues (`VENUE:Manage`): `POST /admin/venues {name, address?}` → 201; `GET /admin/venues` → `Page[VenueOut]`; `GET /admin/venues/{id}`; `PATCH /admin/venues/{id} {name?, address?, is_active?}`.
- Admin courts (`COURT:Manage` + scope): `POST /admin/venues/{venue_id}/courts {name, sort_order=0}` → 201; `GET /admin/venues/{venue_id}/courts` → `Page[CourtOut]` including inactive, ordered `(sort_order, id)`; `GET /admin/courts/{id}`; `PATCH /admin/courts/{id} {name?, sort_order?, is_active?}`. Out-of-scope venue or court → 404 `NOT_FOUND`.
- Catalog (any authenticated user, `VenueScope.everything()`): `GET /venues` → `Page[VenueOut]` active only, ordered by name; `GET /venues/{id}/courts` → `Page[CourtOut]` active only; inactive or missing venue → 404.
- Repositories: `VenueRepository.list(session, *, scope, page, limit, active_only: bool)`, `.get(session, venue_id, *, scope, active_only: bool)`; `CourtRepository.list_for_venue(session, venue_id, *, scope, page, limit, active_only)`, `.get(session, court_id, *, scope)` (joins venue for the scope check).
- Writes stamp `created_by_user_id` / `updated_by_user_id` from the caller.

- [ ] **Tests:** `test_manager_cannot_create_venue` (403); `test_admin_creates_and_deactivates_venue`; `test_catalog_hides_inactive_venue_and_its_courts` (list omits it, courts → 404); `test_player_can_browse_catalog`; `test_catalog_requires_auth` (401); `test_manager_creates_court_at_own_venue`; `test_manager_cannot_touch_other_venue_court` (create under B → 404, `GET`/`PATCH` B's court → 404, same body as id 999999); `test_staff_cannot_manage_courts` (403); `test_courts_ordered_by_sort_order_then_id`; `test_catalog_hides_inactive_court`; `test_court_audit_columns_stamped`.
- [ ] Fail → implement → gate → commit `Add venue and court management and the public catalog`.

### Task 4: Price rules — admin routes and the resolved-price preview

**Files:** Create `src/pricing/{dto,service}.py`, `src/pricing/persistence/repository.py`, `src/pricing/controllers/admin.py`. Modify `src/main.py`. Test `tests/pricing/test_price_rules_api.py`.

**Interfaces (`PRICE:Manage` + scope, via the court's venue):**
- `PriceRuleOut {id, court_id, day_of_week, hour_of_day, price_centavos}`.
- `GET /admin/courts/{court_id}/price-rules` → `list[PriceRuleOut]` ordered most-specific first.
- `POST /admin/courts/{court_id}/price-rules {day_of_week?: 0..6, hour_of_day?: 0..23, price_centavos: 0..10_000_000}` → 201; duplicate → 409 `PRICE_RULE_EXISTS`.
- `PATCH /admin/price-rules/{id} {price_centavos}` → 200; `DELETE /admin/price-rules/{id}` → 204.
- `GET /admin/courts/{court_id}/prices?date=YYYY-MM-DD` → `list[{starts_at, hour, price_centavos}]`, 24 entries for that Manila day, from one rules query and one `PriceResolver`.
- `PriceRuleRepository.list_for_court(session, court_id) -> list[PriceRuleModel]` is called only after the court passed the scoped `CourtRepository.get`; `get(session, rule_id, *, scope)` joins court → venue.

- [ ] **Tests:** `test_prices_follow_the_spec_table` (create the four §4.4 rules; Saturday 18:00 → 90000, Saturday 10:00 → 80000, Monday 18:00 → 60000, Monday 10:00 → 70000); `test_prices_fall_back_to_parameter` (no rules → 50000 × 24); `test_prices_day_is_manila` (the 24 `starts_at` run from `date − 1 day 16:00Z` to `date 15:00Z`); `test_duplicate_rule_is_409`; `test_patch_changes_price_only` (`day_of_week` in body → 422); `test_delete_rule`; `test_manager_cannot_price_other_venue_court` (404 on list, create, patch, delete); `test_staff_cannot_manage_prices` (403); `test_invalid_hour_is_422`.
- [ ] Fail → implement → gate → commit `Add price rule management and resolved-price preview`.
- [ ] Update `pikol-backend/CLAUDE.md` "What exists now" and the parent `CLAUDE.md` current state; commit; push both; confirm CI.

## Out of scope

- Availability grid and slot status (Phase 4).
- Bulk price editing, copying rules between courts.
- Venue deletion, court deletion.
