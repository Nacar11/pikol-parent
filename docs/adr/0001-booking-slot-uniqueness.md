# ADR 0001: A maintenance closure is a booking row

**Date:** 2026-09-09
**Status:** Accepted
**ERD:** [`pikol.dbml`](./pikol.dbml) — paste into https://dbdiagram.io to render

## Context

A court-hour can be occupied by four different things: a player's online
booking, a staff-entered walk-in, a hold that has not been paid for yet, and a
maintenance closure. Any two of them landing on the same `(court, hour)` is a
correctness failure — and the online-vs-online case is not hypothetical, it is
a normal Friday evening.

The obvious model gives closures their own table. That leaves at least one
collision pair (booking vs closure) spanning two tables, which no single
constraint can cover. Preventing it then requires application-level locking:
`SELECT FOR UPDATE`, or advisory locks, or a read-check-write sequence with a
race window in the middle.

## Decision

**A maintenance closure is a row in `booking_groups` with
`type = 'MAINTENANCE'`**, no player and no payment. Everything that occupies a
court-hour lives in one table, and one partial unique index protects it:

```sql
CREATE UNIQUE INDEX uq_court_slot_active
    ON booking_slots (court_id, starts_at)
 WHERE booking_status NOT IN ('CANCELLED', 'EXPIRED');
```

The predicate is **inverted deliberately** — `NOT IN (cancelled, expired)`
rather than `IN (pending, confirmed)` — so `COMPLETED` and `NO_SHOW` keep
holding their slots. Otherwise staff backdating a walk-in could collide with
an already-finished booking.

Two consequences follow that are not optional:

**Status is denormalized onto `booking_slots`.** A Postgres index predicate
cannot reference another table, so the status must sit on the row the index
protects. `booking_groups` stays authoritative, and a **single repository
method owns every status write**, updating both tables in one transaction.
Nothing else may set a status.

**`price_rules` needs `NULLS NOT DISTINCT`.** Its uniqueness constraint spans
two nullable wildcard columns. Postgres treats NULLs as distinct by default,
so without it every wildcard row is trivially unique, the constraint does
nothing, and two equally-specific rules resolve by arbitrary row order.

## Consequences

**Good.** Every collision is impossible at the database level — online vs
online, online vs walk-in, and booking vs maintenance — with no application
locking, no `SELECT FOR UPDATE`, and no read-then-write race. The loser of a
race gets an `IntegrityError`, which the router turns into `409`. A closure
can never evict a booking, so staff must cancel first, deliberately and on the
record. Availability is one query.

**Bad.** Closing a court for a week writes 168 rows. Postgres does not care,
and bulk closure is one loop in a transaction.

**Watch.** The denormalized status is the sharpest edge in the schema. If any
code path writes `booking_slots.booking_status` independently of its group,
availability itself is corrupted. This warrants a dedicated test, not a
comment.

**Related.** Hold expiry is lazy — cleanup happens on the write path, not a
scheduler — so a `PENDING` row on an uncontested slot persists indefinitely.
`booking_status = 'PENDING'` therefore never means "currently holding", and
every read of it must also filter `expires_at > now()`. See spec §4.3.

## The ERD is a picture, not the source of truth

`pikol.dbml` renders the schema, but **DBML cannot express either constraint
above** — it has no `WHERE` clause and no `NULLS NOT DISTINCT`. Both render as
plain unique constraints. Generating DDL from that file would forbid ever
rebooking a cancelled slot, and would silently drop the price-rule guard.

Alembic migrations are authoritative. The `.dbml` is for reading.
