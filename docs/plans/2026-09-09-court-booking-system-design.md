# Pikol — Pickleball Court Booking System — Design

**Date:** 2026-09-09
**Status:** 🚧 In progress — product, data model, booking lifecycle, auth, API
surface, and frontend designed and approved. Deployment pending.
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

Self-registration is **players only**. Staff accounts are created by the
admin, so registration can never become a privilege-escalation path.

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
| 20 | All free tiers; cold starts accepted | |

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

`courts.slot_minutes` exists from day one (default 60), so per-venue slot
lengths later are a config change rather than a migration and a rewrite.

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
users           id, email, password_hash?, full_name, mobile,
                role_id, venue_id?, is_active, email_verified_at?
user_tokens     id, user_id, purpose, token_hash, expires_at, used_at?
                -- purpose: EMAIL_VERIFICATION | PASSWORD_RESET | INVITATION
venues          id, name, address, is_active
courts          id, venue_id, name, slot_minutes=60, is_active
price_rules     id, court_id, day_of_week?, hour_of_day?, price_centavos
booking_groups  id, court_id, player_id?, type, booking_status, payment_status,
                price_centavos, fee_centavos, total_centavos, payment_method,
                expires_at?, created_by_user_id, cancelled_at?,
                cancelled_by_user_id?, cancellation_reason?, notes?
booking_slots   id, booking_group_id, court_id, starts_at, ends_at,
                price_centavos, booking_status
payments        id, booking_group_id, provider, provider_checkout_id,
                provider_payment_id?, amount_centavos, status, raw_payload
webhook_events  id, provider, provider_event_id UNIQUE, payload, processed_at?
```

Money is **integer centavos**, never float. Times are `timestamptz` stored
UTC, rendered Asia/Manila (PH has no DST, so the offset is a fixed +08).

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

### 4.2 Expiring holds without depending on a scheduler

An index predicate cannot reference `now()` — Postgres requires immutable
predicates — so an abandoned `PENDING` row would otherwise block its slot
forever. The fix is that **the write path cleans up exactly the row it needs**,
inside the same transaction:

```sql
UPDATE booking_slots SET booking_status = 'EXPIRED'
 WHERE court_id = :court_id AND starts_at = :starts_at
   AND booking_status = 'PENDING' AND expires_at <= now();
-- then INSERT ...;  IntegrityError → 409
```

A hold is therefore released the instant anyone tries to take the slot,
whether or not any job ran. The availability *read* applies the same rule as a
filter, so expired holds vanish from the grid immediately rather than 15
minutes late.

**A scheduler is available and optional.** Supabase `pg_cron` runs on the free
tier and works even while the API is asleep; GitHub Actions cron is the asima
pattern. Use it for hygiene — tidying expired rows for clean reporting, and
later reminder emails — never for correctness.

### 4.3 Pricing: wildcard rules over a parameter fallback

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

### 4.4 The `parameters` table

| key | default | governs |
|---|---|---|
| `default_slot_price_centavos` | `50000` | ₱500 fallback when no rule matches |
| `convenience_fee_percent` | `3.00` | flat fee players pay online |
| `hold_duration_minutes` | `15` | PENDING expiry window |
| `booking_horizon_days` | `30` | how far ahead players may book |
| `max_active_holds_per_player` | `3` | denial-of-inventory cap |
| `staff_backdate_days` | `7` | walk-in reconciliation window |
| `cancellation_cutoff_minutes` | `0` | how close to start a player may cancel |
| `default_slot_minutes` | `60` | slot length for new courts |

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

`IN_PROGRESS` is **derived, never stored**: `CONFIRMED ∧ now ∈ [starts_at,
ends_at)`. A stored value would be wrong between scheduler ticks — a 6:00pm
booking under a 5-minute cron reads `CONFIRMED` until 6:05. Derived is exact
at every read and cannot drift.

### 5.2 Transitions and guards

| From | To | Actor | Guard |
|---|---|---|---|
| — | `PENDING/UNPAID` | Player | Slots free, on the hour, within horizon, not past, < 3 active holds |
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
   - validate slots are consecutive, on the hour, within horizon, not past
   - check the player's active hold count against `max_active_holds_per_player`
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
3. If re-acquisition fails, mark the group `PAID_UNFULFILLED` and surface it
   in an **admin resolution queue**.

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
| **Database** | Slot uniqueness, on-the-hour `CHECK`, FK integrity, webhook event uniqueness, non-negative money |
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
`price_centavos`. A single query, with the hold-expiry rule applied as a
filter so abandoned holds are already gone. The frontend renders it directly:
no N+1 per court, no client-side price resolution.

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

### 7.5 The webhook route needs three exemptions

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

## 9. Still to design

- **Deployment** — Vercel + Render + Supabase, environments, CI

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
