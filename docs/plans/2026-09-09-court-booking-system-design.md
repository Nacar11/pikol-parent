# Pikol — Pickleball Court Booking System — Design

**Date:** 2026-09-09
**Status:** ✅ Design complete and approved across all sections. Ready for an
implementation plan. No application code exists yet.
**Scope:** Full system design for a single-owner, multi-venue pickleball court
booking platform with online payments, ahead of any implementation.

---

## 1. Product

One business owner operates **many venues**. Each venue has **many courts**.
Courts are sold in **clock-aligned 1-hour slots**, 24 hours a day. Registered
players book and pay online; staff book walk-ins and collect cash at the
counter. Both paths write the same booking aggregate and pass the same
availability check.

**Users and scoping**

| Role | Scope |
|---|---|
| `SUPER_ADMIN` | All venues. This is the owner's account. |
| `VENUE_MANAGER` | Exactly one venue ("store owner"). |
| `VENUE_STAFF` | Exactly one venue ("store member"). |
| `PLAYER` | Self-service only; no venue scope. |

Scoping is a single nullable `users.venue_id` column — a manager runs exactly
one venue, so a join table would be unused ceremony. `NULL` for admin and
players.

Self-registration is **players only**. Staff accounts are created by an admin
or by their own venue's manager (§6.4), so registration can never become a
privilege-escalation path.

---

## 2. Decision ledger

Every decision below was made explicitly during brainstorming. Where a
decision closed off an obvious alternative, the reason is recorded.

| # | Decision | Notes |
|---|---|---|
| 1 | **Single-tenant**, one owner, many venues | Venue is a grouping + scheduling boundary, not a tenancy boundary |
| 2 | **PayMongo**, not Stripe | See §2.1 |
| 3 | **Clock-aligned hourly slots** | Not arbitrary ranges — see §2.2 |
| 4 | **24/7 bookable**; closures are the exception | No operating-hours config in v1 |
| 5 | **Maintenance closure is a booking row** | Gives one race-proof constraint for all collisions |
| 6 | Players **must register and log in** to use the app at all | No guest checkout, no anonymous browsing |
| 7 | Staff book walk-ins; **same aggregate**, cash/manual-GCash payment | No parallel flow |
| 8 | **Hold-then-confirm**, 15-min expiry | Full payment online |
| 9 | **No refunds**; cancellation returns the slot to inventory | Player may still cancel; they just aren't repaid |
| 10 | `booking_group` (payment + status) → `booking_slots` (uniqueness) | A multi-hour session is one payable, cancellable unit |
| 11 | **`parameters` table is the only source of default price** | Courts hold no price column |
| 12 | Per-court and peak pricing via **wildcard `price_rules`** | See §4.3 |
| 13 | Email adapter: **password reset + booking receipt** | No SMS in v1 |
| 14 | 30-day player horizon; no past slots; staff backdate ≤ 7 days | |
| 15 | Max **3 active holds** per player | Denial-of-inventory cap |
| 16 | **Player pays a flat convenience fee** online; walk-ins pay none | See §4.4 |
| 17 | Staff set `COMPLETED` / `NO_SHOW`; **`IN_PROGRESS` is derived** | |
| 18 | **Mock payment provider first** → PayMongo test → live | Validates the port/adapter boundary |
| 19 | **Three repos**, asima-style | parent + backend + frontend |
| 20 | All free tiers; cold starts accepted | Keep-alive is load-bearing, see §9.3 |
| 21 | Players **verify email at signup**; staff are **invited** | Manager may create `VENUE_STAFF` only |
| 22 | Staff may create closures and cancel bookings | With `cancelled_by_user_id` + required reason |
| 23 | Availability grid: **time vertical, courts horizontal** | One component for players and staff |
| 24 | Slots are **fixed at one hour**; no `slot_minutes` column | Rejected as speculative — it contradicted the on-the-hour `CHECK` |
| 25 | All day boundaries are **Manila-local**, not UTC | §4; otherwise the 12am slot lands on the wrong day |

### 2.1 Why PayMongo and not Stripe

Stripe operates in the Philippines, but three things disqualify it here:

1. **Payouts are PHP-only to PH banks**, and USD charges auto-convert with FX
   loss on every transaction.
2. **Stripe Connect is limited in PH** — irrelevant today (single owner, one
   bank account) but it removes the usual reason to tolerate the rest.
3. **The decisive one:** players will pay with **GCash**, not cards. Stripe's
   PH local-wallet coverage is thin. PayMongo treats GCash, Maya, GrabPay and
   QR Ph as first-class.

Xendit was the other serious candidate. Its advantage is **disbursements** —
paying out to third parties — which only matters in a marketplace where
independent venue owners must be remitted. This system has one owner and one
bank account, so that advantage is worth nothing, and PayMongo's public
pricing and PH-focused DX win.

Indicative PayMongo rates: cards ~3.5% + ₱15, GCash/Maya ~2.5%, payout T+1.

### 2.2 Why fixed slots and not arbitrary time ranges

Arbitrary `[start, end)` ranges need `tstzrange` plus a `GIST` exclusion
constraint, a timeline UI instead of a grid, and they create dead 30-minute
gaps nobody books. Fixed slots make availability a lookup and double-booking a
unique-index violation. Pickleball courts are sold in whole blocks anyway.

A configurable `slot_minutes` column was considered and rejected: it would
contradict the on-the-hour `CHECK` (a 90-minute court needs slots at 01:30),
and nothing asks for it. Variable slot length is a migration on the day a
venue requests it — which is the honest price, rather than carrying a column
that quietly cannot work.

---

## 3. Architecture

### 3.1 Repository layout

Three repos, mirroring asima:

```
pikol-parent/          # own git repo — docs, CLAUDE.md, tasks/, .github
├── docs/
│   ├── adr/                # architecture decision records
│   └── plans/              # committed plan snapshots (this file)
├── pikol-backend/     # own git repo — FastAPI + SQLAlchemy + Postgres
└── pikol-frontend/    # own git repo — Next.js 15 App Router
```

Working files (`tasks/plan.md`, `tasks/todo.md`) are gitignored per-repo;
committed plan snapshots live only in the parent, exactly as in asima.

### 3.2 Supabase is infrastructure, not a backend

This is the load-bearing carry-over from asima and it does not change:

- **Postgres** via the session pooler (port 5432, IPv4, prepared statements).
- **Storage** via the S3-compatible endpoint, if/when files are needed.
- **Data API disabled.** No Supabase client SDK. No Supabase Auth. No
  RLS-as-authorization.

FastAPI owns authentication, every domain invariant, and all migrations.
Supabase is a dial-tone. The frontend never holds a Supabase credential.

### 3.3 NestJS → FastAPI mapping

| asima (NestJS) | this system (FastAPI) |
|---|---|
| Nest module | router package, same four-folder layout |
| TypeORM entity + migration | SQLAlchemy 2.0 async + Alembic |
| class-validator DTO | Pydantic v2 schema |
| Nest DI providers | `Depends()` + ABC/Protocol repository |
| `@nestjs/swagger` | built-in OpenAPI |
| passport-jwt guard | `get_current_user` / `require_permission` dependency |
| `@nestjs/throttler` | slowapi |
| `@aws-sdk/client-s3` | aioboto3, same Supabase S3 endpoint |
| Jest | pytest |
| `forbidNonWhitelisted: true` | Pydantic `extra="forbid"` |
| snake_case transformer layer | **dropped** — snake_case is native to Python |

### 3.4 Layering: depth where invariants live

Every module keeps asima's four folders so navigation is identical:

```
src/<feature>/
├── controllers/    # HTTP edge; admin/:id vs /me split
├── domain/         # pure Python — no SQLAlchemy imports
├── dto/            # Pydantic request/response schemas
└── persistence/    # SQLAlchemy models, mappers, repositories
```

But the **domain layer is rich only where there is something to protect**:

- **Rich** — `bookings`, `pricing`, `payments`. Real aggregates, value
  objects, typed errors, and unit tests that touch no database.
- **Thin** — `venues`, `courts`, `users`, `roles`, `parameters`. Router →
  service → SQLAlchemy model. `domain/` holds enums and errors only; no
  mapper, no aggregate.

The reason for the split is the **mapper tax**: in TypeScript, entity↔domain
mapping is boilerplate the language absorbs. In Python it is a hand-written,
hand-tested file per entity. Asima earns that ceremony across leave balances
and approval chains. A `Venue { id, name, address }` does not.

### 3.5 Conventions carried from asima

- All routes under `/api/v1/...`; version bumped via a shared constant.
- List endpoints return `{ data, total, page, limit, has_more }`.
- **snake_case end-to-end** — DB column → domain → JSON wire payload.
- Every response carries `X-Request-ID`.
- Bearer JWT, 15-min access token, 7-day refresh with rotation and
  server-side revocation.
- **Admin vs self-service enforced by schema, never by `if (role)`.**
  `/admin/bookings/{id}` vs `/bookings/me`; identity comes from the token,
  never from the URL. Privileged operations get their own endpoints rather
  than extra fields on a shared body.

---

## 4. Data model

```
parameters      key PK, value, value_type, description, updated_at, updated_by
roles           id, code, name
permissions     id, code                       -- RESOURCE:Action
users           id, email UNIQUE, password_hash?, full_name, mobile,
                role_id, venue_id?, is_active, email_verified_at?
user_tokens     id, user_id, purpose, token_hash, expires_at, used_at?
                -- purpose: EMAIL_VERIFICATION | PASSWORD_RESET | INVITATION
venues          id, name, address, is_active
courts          id, venue_id, name, is_active
price_rules     id, court_id, day_of_week?, hour_of_day?, price_centavos
                -- UNIQUE (court_id, day_of_week, hour_of_day) NULLS NOT DISTINCT
booking_groups  id, court_id, player_id?, type, booking_status, payment_status,
                -- type: PLAYER_BOOKING | WALK_IN | MAINTENANCE
                price_centavos, fee_centavos, total_centavos, payment_method,
                expires_at?, created_by_user_id, cancelled_at?,
                cancelled_by_user_id?, cancellation_reason?,
                needs_resolution=false, notes?
booking_slots   id, booking_group_id, court_id, starts_at,
                price_centavos, booking_status
payments        id, booking_group_id, provider, provider_checkout_id,
                provider_payment_id?, amount_centavos, status, raw_payload
webhook_events  id, provider, provider_event_id UNIQUE, payload, processed_at?
```

Money is **integer centavos**, never float. Times are `timestamptz` stored
UTC, rendered Asia/Manila (PH has no DST, so the offset is a fixed +08).

**Every slot is exactly one hour.** `ends_at` is therefore derived
(`starts_at + 1 hour`), never stored — a stored copy only creates drift. A
configurable slot length was considered and **rejected**: it contradicts the
on-the-hour `CHECK` below (a 90-minute court needs slots at 01:30), and no
requirement asks for it. When a venue actually does, it is a migration.

**Dates are Manila-local, always.** This is not a rendering concern — it
changes query boundaries. Manila is UTC+8, so Manila's 15 September begins at
`2026-09-14T16:00Z`. The obvious `WHERE starts_at::date = :date` returns the
wrong 24 hours, and the slot it misplaces first is **12am–1am**, which lands
on the previous day's grid. Every day-bounded query uses:

```sql
starts_at >= (:date::timestamp AT TIME ZONE 'Asia/Manila')
AND starts_at <  ((:date::timestamp + interval '1 day') AT TIME ZONE 'Asia/Manila')
```

### 4.1 The constraint that makes the system correct

A **maintenance closure is a booking row** — `type = 'MAINTENANCE'`, no
player, no payment. That is a deliberate liberty, and it buys one thing:

```sql
CREATE UNIQUE INDEX uq_court_slot_active
    ON booking_slots (court_id, starts_at)
 WHERE booking_status NOT IN ('CANCELLED', 'EXPIRED');
```

That single partial index makes **every** collision impossible at the database
level — online vs online, online vs walk-in, and booking vs maintenance — with
no application locking, no `SELECT FOR UPDATE`, and no read-then-write race.
Two players tapping the same 6pm court in the same millisecond: Postgres picks
a winner, the loser's insert raises `IntegrityError`, and the router returns
`409 Slot no longer available`.

If closures lived in a separate table, the booking-vs-closure race would need
locking logic this design simply does not contain.

The predicate is **inverted deliberately** (`NOT IN (CANCELLED, EXPIRED)`
rather than `IN (PENDING, CONFIRMED)`) so that `COMPLETED` and `NO_SHOW` still
hold their slots — otherwise staff backdating a walk-in could collide with an
already-completed booking.

Cost: closing a court for a week writes 168 rows. Postgres does not care.

### 4.2 Status is denormalized onto slots — deliberately

`booking_slots` carries its own `booking_status`, duplicating the group's. That
is not an oversight: the partial unique index in §4.1 must live on
`booking_slots`, and a Postgres index predicate **cannot reference another
table**. Uniqueness therefore requires the status to sit on the row it
protects.

The invariant this creates must be honoured everywhere:

> **`booking_slots.booking_status` always mirrors its group, written in the
> same transaction, never independently. `booking_groups` is authoritative.**

A single repository method owns every status write and updates both tables
together. No other code path may set a status. Anything that violates this
corrupts availability itself, so it is worth a dedicated test.

`court_id` is duplicated for the same reason and carries the same rule: the
index needs it on `booking_slots`, and `booking_groups.court_id` is
authoritative. It also makes explicit that **a group belongs to exactly one
court** — booking two courts for the same hour is two groups and two payments.

`expires_at` is **not** duplicated — it lives only on `booking_groups`, so
expiry queries join.

### 4.3 Expiring holds without depending on a scheduler

An index predicate cannot reference `now()` either — Postgres requires
immutable predicates — so an abandoned `PENDING` row would otherwise block its
slot forever. The fix is that **the write path cleans up exactly the rows it
needs**, inside the same transaction, before inserting:

```sql
-- 1. expire any group holding this slot whose hold has lapsed
WITH lapsed AS (
    SELECT g.id
      FROM booking_groups g
      JOIN booking_slots s ON s.booking_group_id = g.id
     WHERE s.court_id = :court_id
       AND s.starts_at = :starts_at
       AND g.booking_status = 'PENDING'
       AND g.expires_at <= now()
), _g AS (
    UPDATE booking_groups SET booking_status = 'EXPIRED'
     WHERE id IN (SELECT id FROM lapsed)
)
UPDATE booking_slots SET booking_status = 'EXPIRED'
 WHERE booking_group_id IN (SELECT id FROM lapsed);

-- 2. then INSERT the new slots;  IntegrityError → 409
```

Both tables move together, preserving the §4.2 invariant. Note that expiring a
*group* releases **all** its slots, not only the contested one — a lapsed
6–8pm hold frees both hours, which is correct: the group is the unit that was
never paid for.

A hold is therefore released the instant anyone tries to take the slot,
whether or not any job ran.

**The consequence, and it is the sharpest trap in this design:** a lapsed hold
on a slot *nobody contests* keeps its `PENDING` row indefinitely. Cleanup is
lazy, so `booking_status = 'PENDING'` alone never means "currently holding".

> **Every read of `PENDING` must apply `AND expires_at > now()`.** There are no
> exceptions, and this is a required test.

Two places depend on it, and the second one bites:

- **Availability** — without the filter, abandoned holds block the grid for 15
  minutes. Annoying, self-healing.
- **The hold cap** — a naive `COUNT(*) WHERE player_id = ? AND
  booking_status = 'PENDING'` counts holds that lapsed weeks ago. A player who
  abandons checkout three times on unpopular slots is then **permanently
  unable to book**, and nothing ever clears it. Abandoning checkout is the
  common case, so this is a self-inflicted denial of service on paying
  customers.

**A scheduler is available and optional.** Supabase `pg_cron` runs on the free
tier and works even while the API is asleep; GitHub Actions cron is the asima
pattern. Use it for hygiene — tidying expired rows for clean reporting, and
later reminder emails — never for correctness.

### 4.4 Pricing: wildcard rules over a parameter fallback

Courts hold **no price column**. A court's "own price" is a wildcard rule:

| Rule | `day_of_week` | `hour_of_day` | Means |
|---|---|---|---|
| Court flat rate | `NULL` | `NULL` | ₱700 any time |
| Weekend rate | `6` | `NULL` | ₱800 all Saturday |
| Peak hour | `NULL` | `18` | ₱600 at 6pm daily |
| Weekend peak | `6` | `18` | ₱900 Saturday 6pm |

Resolution is most-specific-wins, then the parameter:

```
(day, hour) → (hour) → (day) → (NULL, NULL) → parameters.default_slot_price_centavos
```

One code path covers per-court pricing and peak pricing.

**`booking_slots.price_centavos` is frozen at creation** and never recomputed.
The owner raising the 6pm rate must not retroactively change what someone
already paid.

### 4.5 The `parameters` table

| key | default | governs |
|---|---|---|
| `default_slot_price_centavos` | `50000` | ₱500 fallback when no rule matches |
| `convenience_fee_percent` | `3.00` | flat fee players pay online |
| `hold_duration_minutes` | `15` | PENDING expiry window |
| `booking_horizon_days` | `30` | how far ahead players may book |
| `max_active_holds_per_player` | `3` | denial-of-inventory cap |
| `staff_backdate_days` | `7` | walk-in reconciliation window |
| `cancellation_cutoff_minutes` | `0` | how close to start a player may cancel |
| `min_lead_minutes` | `20` | earliest a player may book before a slot starts |
| `invitation_expiry_days` | `7` | staff invitation token lifetime |
| `webhook_payload_retention_days` | `90` | before raw provider payloads are purged |

**`min_lead_minutes` must be ≥ `hold_duration_minutes`**, and validation
enforces that when either is written. Otherwise a hold can outlive the slot it
holds: a player books a slot starting in 5 minutes, pays at minute 12, and the
webhook confirms a session that began 7 minutes ago. The "not past" check runs
at hold creation and never again. **Walk-ins are exempt** — staff are booking
someone standing at the counter.

The PayMongo checkout session expiry is **derived**, not configured:
`hold_duration_minutes − 3`. Deriving it guarantees the checkout can never
outlive the hold that backs it, which is the §5.5 hazard.

Values load through a Pydantic model with a short TTL cache, so `"abc"` in
`convenience_fee_percent` fails loudly at load rather than silently zeroing
every fee. Admin-only writes; `updated_by` gives an audit trail on the numbers
that move money.

**Why a flat fee percentage.** PayMongo's rate depends on the method (~2.5%
wallets, ~3.5% + ₱15 cards), but the player chooses their method on PayMongo's
hosted page *after* the amount is fixed. Charging exact pass-through would
require method selection in our own UI and a method-restricted checkout
session — more UI, more failure modes, and a few pesos of accuracy. One
tunable number wins.

`fee = round_half_up(price × convenience_fee_percent / 100)` in integer
centavos, **captured on `booking_groups.fee_centavos` at creation** and never
recomputed. Walk-ins get `fee_centavos = 0`.

---

## 5. Booking lifecycle and payment flow

### 5.1 Two independent axes

A single status column cannot express a staff-created walk-in that holds its
slot but has not been paid for yet. Two axes:

```
booking_status    PENDING · CONFIRMED · COMPLETED · NO_SHOW · CANCELLED · EXPIRED
payment_status    UNPAID · PAID · FAILED
```

- **Online:** `PENDING/UNPAID` → webhook → `CONFIRMED/PAID`
- **Walk-in:** `CONFIRMED/UNPAID` → staff collects → `CONFIRMED/PAID`

`booking_groups.payment_status` is **authoritative** — it is what guards read
and what the UI shows. `payments.status` tracks one provider attempt, and a
group may accumulate several rows when a player retries a failed checkout.

`IN_PROGRESS` is **derived, never stored**: `CONFIRMED ∧ now ∈ [starts_at,
starts_at + 1 hour)`. A stored value would be wrong between ticks — a 6:00pm
booking under a 5-minute cron reads `CONFIRMED` until 6:05. Derived is exact
at every read and cannot drift.

### 5.2 Transitions and guards

| From | To | Actor | Guard |
|---|---|---|---|
| — | `PENDING/UNPAID` | Player | Slots free, on the hour, within horizon, ≥ `min_lead_minutes` away, < `max_active_holds_per_player` **unexpired** holds |
| — | `CONFIRMED/UNPAID` | Staff | Slots free; may backdate ≤ `staff_backdate_days` |
| `PENDING` | `CONFIRMED` | **Webhook only** | Signature verified, event not already processed |
| `PENDING` | `EXPIRED` | System | `expires_at <= now()` |
| `PENDING` | `CANCELLED` | Player | Explicit abandon |
| `CONFIRMED/UNPAID` | `CONFIRMED/PAID` | Staff | Walk-in cash or manual GCash collected |
| `CONFIRMED` | `CANCELLED` | Player | Before `starts_at − cancellation_cutoff_minutes`; **no refund** |
| `CONFIRMED` | `CANCELLED` | Staff | Any time |
| `CONFIRMED` | `COMPLETED` | Staff | Only once `starts_at <= now()` |
| `CONFIRMED` | `NO_SHOW` | Staff | Only once `starts_at <= now()` |

Terminal: `COMPLETED`, `NO_SHOW`, `CANCELLED`, `EXPIRED`.
`payment_status` never moves backwards from `PAID`.

### 5.3 The online booking flow

1. Player selects a court, a date, and one or more **consecutive** hours.
2. `POST /api/v1/bookings` → **one transaction**:
   - validate slots are consecutive, on the hour, within horizon, and at
     least `min_lead_minutes` in the future
   - count the player's **unexpired** holds — `booking_status = 'PENDING'
     AND expires_at > now()` — against `max_active_holds_per_player`
   - resolve each slot's price through the rule chain; compute the fee
   - expire stale `PENDING` rows for those exact `(court, slot)` pairs
   - insert `booking_group` (`PENDING/UNPAID`, `expires_at = now + hold`)
   - insert one `booking_slot` per hour
   - `IntegrityError` → `409 Slot no longer available`
3. **Commit, then** call PayMongo to create a checkout session. The external
   call is deliberately outside the transaction — holding one open across a
   network round-trip is how connection pools die. If PayMongo fails, return
   `502` and let the hold expire naturally.
4. Store `provider_checkout_id`; return `checkout_url`; frontend redirects.
5. Player pays on PayMongo's hosted page.
6. **Webhook** `checkout_session.payment.paid` → verify signature → idempotency
   check → `CONFIRMED/PAID` → send receipt email.
7. Player is redirected back to our success page, which **reads booking state
   from our database**.

**The browser redirect never confirms a booking.** A player who closes the tab
after paying must still get their court; a player who hand-types the success
URL must not. The webhook is the only authority.

### 5.4 Webhook handling

- **Signature verified** before the payload is parsed or trusted.
- **Idempotent by construction:** insert `webhook_events.provider_event_id`
  first; a unique violation means "already processed" and the handler returns
  `200` without acting. PayMongo retries, and retries must be free.
- **Out-of-order safe:** only forward transitions are permitted, so a late
  duplicate cannot walk a `COMPLETED` booking back to `CONFIRMED`.
- **Always `200` once recorded.** A `500` triggers redelivery; a non-retryable
  application error should be recorded and drained, not retried forever.

### 5.5 The known hazard: payment landing after the hold expires

A player pays at 15:01 against a 15:00 hold. The webhook arrives for an
`EXPIRED` group whose slot may already belong to someone else. **This is the
one case where money can exist without a court.**

Mitigations, in order:

1. Set the **PayMongo session expiry shorter than our hold window** (~12 min
   against 15) so payment cannot normally land after the hold dies.
2. On a late `payment.paid`, **attempt to re-acquire the slots**. If they are
   still free, the booking is restored and nobody notices.
3. If re-acquisition fails, set `needs_resolution = true` and surface the
   group in an **admin resolution queue**. This is deliberately a *flag*, not a
   status: the booking genuinely is `EXPIRED`/`PAID`, and inventing a status
   for it would force every state-machine guard to handle a value that is not
   a state.

Case 3 is rare but real, and it is resolved by a human — refund or rebook. The
"no refunds" policy governs *player-initiated cancellation*; it does not apply
when the system failed to deliver. This is an accepted operational edge, not a
bug to design away.

### 5.6 The payment provider port

```python
class PaymentProvider(Protocol):
    async def create_checkout(self, *, amount_centavos: int,
                              reference: str, description: str) -> CheckoutSession: ...
    def verify_webhook(self, payload: bytes, signature: str) -> WebhookEvent: ...
```

Two adapters, selected by a `PAYMENT_PROVIDER` env var — never an `if (env)`
branch in application code, exactly as asima's storage adapter is switched:

- **`MockProvider`** — development and tests. Returns a local checkout URL and
  exposes a "simulate payment" endpoint that posts a correctly-signed webhook.
  The whole flow is testable with no PayMongo account.
- **`PayMongoProvider`** — test keys, then live keys.

This is what makes "mock mode first" a design property rather than a
temporary hack.

### 5.7 Where each rule is enforced

| Layer | Enforces |
|---|---|
| **Database** | Slot uniqueness, on-the-hour `CHECK`, unique `users.email`, unique `price_rules` specificity (`NULLS NOT DISTINCT`, or every wildcard row is trivially unique and the constraint does nothing), FK integrity, webhook event uniqueness, non-negative money |
| **Domain** (pure Python) | Every transition guard, price/fee computation, hold expiry, consecutive-slot validation — unit tested with no database |
| **Service** | Transaction boundary, orchestration, external calls kept outside the transaction |
| **Router + schema** | Shape validation, authentication, venue scoping |

The rule that must not be broken: **no invariant is enforced only in the
router.** Anything reachable by both the player and the staff path lives in the
domain layer or the database.

---

## 6. Auth, permissions, and venue scoping

### 6.1 Carried from asima unchanged

One `users` table — a player is `role = PLAYER`, not a separate identity
system. Bearer JWT, 15-minute access token, 7-day refresh with rotation and
server-side revocation (asima ADR 0002). Permission codes stay
`RESOURCE:Action`; the frontend gates UI from `GET /users/me/permissions`
rather than parsing roles client-side. `SUPER_ADMIN` is an unconditional
bypass axis, orthogonal to permissions, as in asima ADR 0001.

`user_tokens` serves both password reset and email verification via a
`purpose` column: single-use, hashed at rest, short-lived. Login,
registration, reset-request, and verification-resend are all rate limited.

### 6.2 Venue scoping — the new security surface

Asima is single-tenant and has nothing to copy here. The threat is concrete: a
`VENUE_MANAGER` at Venue A calling `GET /admin/bookings?venue_id=B`, or
`PATCH /admin/courts/{id}` for a court that is not theirs.

**The failure mode to design against is forgetting.** One repository method
that omits the venue filter leaks everything, and it looks entirely normal in
review. So scope is a **mandatory parameter, never optional**:

```python
class VenueScope:          # resolved once, from the token, by a dependency
    all_venues: bool       # SUPER_ADMIN
    venue_id: int | None   # VENUE_MANAGER / VENUE_STAFF
```

Every repository method touching venue-owned data takes `scope: VenueScope` as
a **required argument with no default**. Omitting it is a `TypeError` at call
time rather than a silent data leak in production. The insecure version must
fail to run, not fail quietly.

For `/{id}` routes the check is ownership-after-fetch, returning **404, not
403** — a 403 confirms existence, which tells a manager at Venue A exactly how
many courts Venue B has.

### 6.3 Role → permission map

| | SUPER_ADMIN | VENUE_MANAGER | VENUE_STAFF | PLAYER |
|---|---|---|---|---|
| Scope | all venues | own venue | own venue | self |
| `VENUE:Manage` | ✅ | — | — | — |
| `COURT:Manage` | ✅ | ✅ | — | — |
| `PRICE:Manage` | ✅ | ✅ | — | — |
| `CLOSURE:Manage` | ✅ | ✅ | ✅ | — |
| `BOOKING:ReadAny` | ✅ | ✅ | ✅ | — |
| `BOOKING:CreateWalkIn` | ✅ | ✅ | ✅ | — |
| `BOOKING:MarkPaid` | ✅ | ✅ | ✅ | — |
| `BOOKING:SetOutcome` | ✅ | ✅ | ✅ | — |
| `BOOKING:CancelAny` | ✅ | ✅ | ✅ | — |
| `USER:Manage` | ✅ | ✅ (staff only) | — | — |
| `PARAMETER:Manage` | ✅ | — | — | — |

`parameters` is global and moves money (fee percentage, default price), so it
stays admin-only. A venue manager adjusts pricing through `price_rules` on
their own courts instead.

### 6.4 Why a manager creating users needs two fixed fields

A manager creating his own staff is a requirement, not a risk. The risk is in
the naive implementation, which checks only *may this caller create users?* and
then takes `role_id` and `venue_id` from a body the manager controls:

```jsonc
POST /api/v1/admin/users
{ "email": "me2@x.com", "role_id": 1, "venue_id": null }   // 1 = SUPER_ADMIN
```

He accepts his own invitation and owns every venue. The quieter variant is
worse:

```jsonc
{ "email": "me3@x.com", "role_id": 3, "venue_id": 7 }      // someone else's venue
```

That produces a legitimate staff account at Venue 7. Every `VenueScope` check
passes, because the token genuinely says Venue 7. **Scoping is not bypassed —
it has been handed a valid credential**, and nothing looks wrong in any log.

So on the manager-facing endpoint both fields are fixed by the server:

```python
role_id  = VENUE_STAFF        # fixed by the endpoint, not chosen by the caller
venue_id = caller.venue_id    # from the token, never the body
```

`SUPER_ADMIN` keeps a separate endpoint where role and venue *are* parameters,
because an admin choosing them is the legitimate case. This is the same
principle that keeps `/users/me` free of an `:id` segment: **anything a caller
could use to widen their own authority comes from the token, never from input
they control.**

### 6.5 Two audit fields on cancellation

With no refunds, a wrongly cancelled paid booking is a customer-service
incident. `booking_groups` carries `cancelled_by_user_id` and a **required**
`cancellation_reason` alongside `cancelled_at`. Both are cheap and make staff
cancellations reviewable.

A closure can never evict a booking: `uq_court_slot_active` forbids it, so
staff must cancel first, deliberately and on the record. The constraint does
this; no extra logic exists.

### 6.6 Email verification and staff invitation

**Players self-register and verify at signup.** The account is created
inactive; the verification link activates it. Login before verification fails
with a distinct error that renders a resend action, so nobody is stranded by a
spam-foldered message.

**Staff are invited, not registered** — same email machinery, different shape.
The manager creates the account with **no password at all**
(`password_hash = NULL`, `is_active = false`) and the system emails an
invitation token. The staff member clicks, sets their *own* password, and the
account activates: verification and password creation in one step. The manager
never sets, knows, or transmits a password, which removes the temp-password-
over-Messenger habit before it starts.

Invitations expire after 7 days and are resendable and revocable.

**This puts email on the critical path for both signup and hiring.** A provider
outage or a spam-foldered message now costs a booking or a shift. Configure
SPF/DKIM on the sending domain from day one rather than after the first "I
never got the email" report, and keep the resend action obvious in the UI.

---

## 7. API surface

### 7.1 Authentication is required everywhere

Only the auth endpoints themselves and the payment webhook are reachable
without a valid token. The venue list, the court list, and the availability
grid are **not public** — a visitor registers, verifies, and logs in before
seeing any inventory.

This has one business consequence worth stating rather than discovering: the
app has no publicly crawlable pages, so it contributes nothing to organic
discovery. If that matters later, a separate marketing site can link into the
login page without any change to this design.

### 7.2 The endpoint that matters most

```
GET /api/v1/availability?venue_id=1&date=2026-09-15
```

One request returns the whole day grid — every court, 24 slots each, with
per-slot `status` (`AVAILABLE` · `BOOKED` · `CLOSED` · `PAST`) and resolved
`price_centavos`. The frontend renders it directly: no N+1 per court, no
client-side price resolution.

Three things this endpoint must get right:

- **`date` is Manila-local**, bounded as in §4 — otherwise the 12am–1am slot
  appears on the wrong day.
- **The hold-expiry filter applies** (§4.3), so abandoned holds are already
  gone from the grid.
- **Prices resolve in memory, not per cell.** A 6-court venue is 144 cells,
  each needing the four-level fallback of §4.4. Load the venue's `price_rules`
  once — tens of rows — and resolve against that set. Resolving per cell in
  SQL is 144 round trips wearing a single endpoint's clothing.

Slot `status` deliberately does **not** reveal *who* booked a slot. That
distinction is what lets any authenticated player read the grid while booking
identities stay on the staff endpoints.

### 7.3 Transitions are actions, not PATCHes

```
POST /api/v1/admin/bookings/{id}/cancel      { reason }
POST /api/v1/admin/bookings/{id}/mark-paid   { payment_method }
POST /api/v1/admin/bookings/{id}/outcome     { outcome: COMPLETED | NO_SHOW }
```

Not `PATCH /bookings/{id} { status: "COMPLETED" }`. A generic status patch
invites every transition and pushes the guards into a `match` statement
someone will eventually extend wrongly. A named action route carries one
guard, one permission, and one audit record — and the illegal transitions have
no URL at all.

### 7.4 Endpoint map

| Group | Endpoints |
|---|---|
| **Auth** (unauthenticated) | `register` (players only) · `login` · `refresh` · `logout` · `verify-email` (+ `/resend`) · `invitations/accept` · `password/forgot` · `password/reset` |
| **Self** | `GET/PATCH /users/me` · `PATCH /users/me/password` · `GET /users/me/permissions` |
| **Catalog** | `GET /venues` · `GET /venues/{id}/courts` · `GET /availability` |
| **Player bookings** | `POST /bookings` (hold + checkout) · `GET /bookings/me` · `GET /bookings/me/{id}` · `POST /bookings/me/{id}/cancel` |
| **Admin — setup** | `venues` · `courts` · `courts/{id}/price-rules` · `closures` (bulk range → N slot rows) · `parameters` |
| **Admin — ops** | `GET /admin/bookings` (venue/court/date/status filters) · `POST /admin/bookings/walk-in` · the three action routes above · `GET /admin/bookings/unfulfilled` |
| **Admin — staff** | `POST /admin/users` (invite) · `GET /admin/users` · `PATCH /admin/users/{id}` |
| **Webhook** (unauthenticated) | `POST /webhooks/paymongo` |
| **Health** | `GET /health` |

`GET /admin/bookings/unfulfilled` is the resolution queue from §5.5 — the
paid-but-unhousable cases a human must settle. Small endpoint, but without it
those failures stay invisible until a customer complains.

### 7.5 Raw provider payloads are personal data

`webhook_events.payload` and `payments.raw_payload` hold PayMongo's raw
payloads, which carry payer name, email, and mobile. They are kept because
disputes and webhook debugging need the original bytes, but they are personal
data under the Data Privacy Act (RA 10173) and must not accumulate forever.

- Purge after `webhook_payload_retention_days` (default 90). This is exactly
  the hygiene job `pg_cron` exists for (§4.3).
- **Payloads are never written to application logs.** Log the
  `provider_event_id` and the outcome; the payload stays in the table.

### 7.6 The webhook route needs three exemptions

Each is easy to miss and each breaks delivery silently:

- **No JWT requirement** — PayMongo has no token.
- **No rate limiting** — a retry storm during an incident is exactly when
  delivery matters most, and a 429 would make it permanent.
- **Raw request body preserved** for signature verification. Any middleware
  that parses and re-serializes JSON invalidates the signature.

---

## 8. Frontend

Same stack as asima: Next.js 15 App Router, TanStack Query, react-hook-form +
zod, Tailwind with shadcn/radix, sonner. Feature slices under `src/features/`,
shared primitives under `src/components/{ui,form,layout}`.

The structural difference from asima is **two audiences in one deployment**,
which lands on asima's own route-group shape because nothing is public:

```
src/app/
├── (auth)/       login · register · verify-email · accept-invite · forgot/reset
└── (app)/        everything else, JWT required
    ├── venues/              venue list → court list → availability → book
    ├── bookings/            my bookings, detail
    ├── checkout/return/     PayMongo redirect target
    └── staff/               permission-gated console
        └── schedule · bookings · courts · pricing · closures · users · parameters
```

`/` is a redirect: players to venues, staff to today's schedule, everyone else
to login. Navigation is driven from `GET /users/me/permissions`, never by
parsing a role client-side.

### 8.1 The availability grid

One component, both surfaces. **Time runs vertically (rows), courts run
horizontally (columns)**, with the court header row and the time column both
sticky.

```
┌──────┬──────┬──────┬──────┐
│ SEP  │  C1  │  C2  │  C3  │ ← sticky
├──────┼──────┼──────┼──────┤
│ 5 PM │ ₱500 │  ██  │ ₱500 │
│ 6 PM │ ₱600 │ ₱600 │  ██  │  ← peak pricing visible in-cell
│ 7 PM │  ██  │ ₱600 │ ₱600 │
└──────┴──────┴──────┴──────┘
        ↕ one natural scroll through 24h
```

Chosen because **vertical scrolling is the phone's native gesture** and PH
players are overwhelmingly mobile: 24 hours becomes one thumb scroll rather
than an easy-to-miss horizontal swipe. The same layout serves desktop and the
staff console unchanged — desktop simply shows more rows without scrolling —
so there is one grid component, not two.

Price is rendered **in the cell**, which makes peak pricing legible at a glance
without a legend and without a second lookup.

**Consecutive-hour selection** is tap-start then tap-end within one column, not
drag: every cell between must be free and in the same court. Tap ranges are far
more reliable than drag on touch, and the same interaction works with a mouse.

**Known limit:** beyond ~8 courts the columns squeeze on a phone. The mitigation
is horizontal scroll on the court axis with the time column pinned — deferred
until a venue actually has that many courts.

### 8.2 Four things that are easy to get wrong

**Slot selection must not be optimistic.** The `409` from
`uq_court_slot_active` is an expected outcome on a busy Friday, not an edge
case. Showing a slot as "yours" before the server confirms means walking it
back, which reads as a bug. Tap → server → *then* held.

**The hold needs a visible countdown.** The player has 15 minutes and no way to
know unless told. A timer on the checkout screen, and an honest expiry state
when it lapses — never a silent failure at PayMongo.

**The return page polls; it does not trust the redirect.** PayMongo bounces the
player back, but the webhook may not have landed — it often arrives *before*
the redirect, sometimes seconds after. The return page shows "confirming your
booking", polls `GET /bookings/me/{id}` briefly, then declares success or
points at My Bookings. This is the frontend half of *the webhook is the only
authority*.

**All times render Asia/Manila, never browser-local**, or a player booking from
Dubai books 4am. And the derived `IN_PROGRESS` status comes **from the server**,
never recomputed client-side — a skewed browser clock would otherwise show a
session in progress that has not started.

### 8.3 One build-time guard, learned from asima

A production build with `NEXT_PUBLIC_API_BASE_URL` unset must **fail**, not
succeed while baking `localhost` into the client bundle. Asima shipped that
silent failure once; it is cheap to prevent here from the first commit.

---

## 9. Deployment

All free tier, all Singapore (`ap-southeast-1`) — the closest region to PH.

| Component | Where | Why |
|---|---|---|
| `pikol-frontend` | Vercel | Next.js native |
| `pikol-backend` | Render, Singapore | long-lived process |
| Postgres + Storage | Supabase, Singapore | as in asima |

### 9.1 Why the backend is not on Vercel

The reason differs from asima's. Asima's blocker was the 4.5 MB function
payload cap on file uploads; this system has no uploads. Here it is
**connection pooling**: a serverless function opens and drops a connection per
invocation, and Supabase's pooler has a modest connection budget. A long-lived
Render process holds one properly-sized SQLAlchemy async pool instead of
fighting that on every request.

### 9.2 The Supabase pooler port — the gotcha specific to this stack

Supabase exposes two pooler ports and **the wrong one silently breaks
asyncpg**:

- **Port 6543** (transaction pooler) does *not* support prepared statements.
  SQLAlchemy async with asyncpg uses them by default, producing intermittent
  `prepared statement "__asyncpg_stmt_1__" does not exist` errors under load —
  passing locally, failing in production.
- **Port 5432** (session pooler) is IPv4 and supports prepared statements.
  **Use this**, as asima does.

If 6543 is ever required it needs both `prepared_statement_cache_size=0` and
`statement_cache_size=0`. This belongs in the config comments, because the
failure looks like a database problem and is not.

### 9.3 The sharpest free-tier consequence

Render's free tier sleeps after 15 idle minutes and cold-starts in ~50 s — and
that lands **squarely on the payment confirmation path**. A sleeping backend
means PayMongo's webhook times out and retries while the player watches the
return page spin. It resolves correctly, because retries and polling both
work, but it feels broken.

The UptimeRobot keep-alive ping is therefore **load-bearing here, not
hygiene** — unlike in asima, where it only protected a demo. It also stops
Supabase pausing after 7 idle days. Free, 5-minute interval, and it is the
difference between a 2-second confirmation and a 50-second one.

### 9.4 Carried from asima, already learned the hard way

- **CORS trailing-slash trap** — `https://pikol.vercel.app/` does not match
  `https://pikol.vercel.app`. Exact origins, no trailing slash.
- **Migrations run from a laptop** against a gitignored `.env.supabase.prod`,
  never from CI. Named `.prod` rather than `.local` for the same reason: on a
  file holding production credentials, `.local` reads as "the safe one", and
  that is the misreading that ends with a destructive command pointed at the
  live database. Alembic replaces the TypeORM CLI; the warning is identical.
- **`main.py` binds `0.0.0.0`** and respects Render's injected `PORT`.
- **One Render service only.** A leftover duplicate makes Render's router flap:
  ~50% of requests 404 with `x-render-routing: no-server` without ever reaching
  the container.
- **The frontend production build fails if `NEXT_PUBLIC_API_BASE_URL` is
  unset**, rather than baking `localhost` into the bundle (§8.3).

### 9.5 Secrets, CI, and phasing

`PAYMONGO_SECRET_KEY` and `PAYMONGO_WEBHOOK_SECRET` are **server-only** —
never `NEXT_PUBLIC_*`, which is compiled into the browser bundle. The frontend
holds only `NEXT_PUBLIC_API_BASE_URL`.

Rollout follows the provider port: **mock → PayMongo test → live**. The mock
adapter pays off immediately in development, where webhooks need no ngrok
tunnel because the mock posts a correctly-signed one to itself.

CI per repo via GitHub Actions: backend `ruff` + `mypy` + `pytest`; frontend
`eslint` + `tsc` + `build`.

---

## 10. Open questions

- **BIR official receipts.** PH businesses are legally required to issue them.
  Assumed handled outside this system — recorded as a decision, not a surprise.
- **PayMongo live-mode onboarding** requires a registered PH business (DTI/SEC
  + BIR). Test mode needs no paperwork. This is the long pole on the schedule
  and should start in parallel with development.
- **Free-tier behaviour:** Supabase pauses a project after ~7 days idle;
  Render free cold-starts in ~50s. Accepted. Neither breaks correctness —
  PayMongo retries webhooks — but a mid-payment spinner is poor UX. ~$7/mo on
  Render removes it when it matters.
