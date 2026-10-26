# ADR 0003: Venue-owned user groups replace fixed roles

**Date:** 2026-10-27
**Status:** Accepted
**Supersedes:**
- the four-role model in design §6;
- the `roles` and `role_permissions` tables;
- `users.role_id` and `users.venue_id`;
- "a player is role = PLAYER".

## Context

Identity shipped with four fixed roles: `SUPER_ADMIN`, `VENUE_MANAGER`,
`VENUE_STAFF` and `PLAYER`. They were seeded by migration 0002 from a
constant in code.

- **The owner cannot change what a role may do** without a deploy.
- **Each staff account belongs to exactly one venue** through
  `users.venue_id`, and `VenueScope` was built on that single column.

The owner has asked for three things:

- **Configurable groups.** An admin creates "Front desk", ticks its
  permissions, and assigns people.
- **Staff who work at several venues.** A venue master can run more than
  one venue, and a front-desk person can work at several.
- **Customers who can also be staff.** One account is a customer in the
  mobile app and staff in the web console.

The single `venue_id` column cannot express any of these. A role row
holds no venue, so "Front desk at A" and "Front desk at B" would collapse
into one group.

## Decision

### 1. Groups belong to a venue, and have one of three kinds

`user_groups(kind, venue_id, name)` has three kinds:

| Kind | Venue | Count | Edited by |
|---|---|---|---|
| `CUSTOMER` | none | exactly one | super admin (members only, no permissions) |
| `MASTER` | one | one per venue | super admin |
| `STAFF` | one | any | super admin, or anyone holding `USER:Manage` at that venue, within the escalation rules |

- Permissions attach to groups through `user_group_permissions`, and people
  through `user_group_members`. Both are many-to-many, so one person may be
  in many groups at many venues.
- Every new venue is created with a "Venue master" (`MASTER`) group and a
  "Front desk" (`STAFF`) group.
- `users.system_admin` stays the owner's unconditional bypass. **It is
  never a group.**

### 2. Permissions apply per venue, never summed

- **A staff request names its venue in `X-Venue-ID`.** A caller's
  permissions are the union of their groups **at that venue only**. Ana,
  master at A and front desk at B, can edit prices at A but only take
  walk-ins at B.
- **`VenueScope` keeps its single-venue shape.** `get_venue_access`
  resolves the header once per request. The repositories that take
  `scope` are unchanged, and ownership-after-fetch is still 404.

**Rejected: summing permissions across venues.** It is simpler, with one
permission set per user and no header. But it would let Ana edit prices
at B because she is a master at A. That breaks the venue wall the system
exists to keep, and it breaks it silently: every check passes, and the
logs look normal.

### 3. Customer is a group, and customer access is its own axis

- **Membership of the `CUSTOMER` group is the right to book and pay as
  yourself**: holds, pay, refresh-payment, own bookings, own cancel.
  Staff access is membership of a venue's `MASTER`/`STAFF` groups. The two
  axes are independent, and one account may hold both.
- **The server enforces groups, never the client.** "Customers use the
  mobile app and staff use the web" is how each app is *built*. It is not
  how the server *decides*, because a client identifier can be forged.
  The web app is now staff-only, and the customer routes stay for the
  mobile app.

**Rejected: deriving "customer" as "has no memberships".** That is what
`role = PLAYER` amounted to. It makes a customer who joins Front desk
lose their own bookings, and it gives the owner no way to stop someone
booking without deleting the account.

### 4. Escalation rules live in the group service

- **Owner-only codes are never grantable to any group**, by anyone:
  `VENUE:Manage` and `PARAMETER:Manage`.
- **A non-owner may only add or remove codes they hold at that venue.**
  Codes already on the group that they don't hold are kept.
- **Only the super admin edits `MASTER` and `CUSTOMER` membership**, and
  only the super admin edits or deactivates a master.
- **The server sets a new group's venue and kind** for anyone but the
  owner. A body carrying either is a 422.
- **Nobody may remove their own last `USER:Manage` at a venue.**
- These rules span joins, so they cannot be CHECK constraints. **Each one
  has a test that fails if the rule is removed.**

## Consequences

- **Data and code**
  - Migration 0006 adds the tables and backfills existing data:
    - `PLAYER` → Customer;
    - `VENUE_MANAGER` → master;
    - `VENUE_STAFF` → Front desk.
  - Migration 0007 drops `roles`, `role_permissions`, `users.role_id` and
    `users.venue_id`. Both migrations downgrade.
  - `require_player` becomes `require_customer`. The admin responses drop
    `role` and `venue_id` and carry `groups` instead.
- **Customer access during the gap**
  - Until the mobile app ships, **customers cannot book online**, because
    the web no longer offers the flow. The owner accepted this.
  - The backend customer routes stay fully tested.
- **Work the header creates for clients**
  - A multi-venue staff member must send `X-Venue-ID`. Otherwise the
    server answers 400 `VENUE_REQUIRED`.
  - A single-venue staff member may omit the header.
  - The web console sends the header on every request through the
    injected venue provider in `lib/api-client`.
- **Rules that do not change:** the four principles of the system brief,
  namely:
  - the venue wall;
  - 404 rather than 403;
  - privilege fields fixed by the server;
  - gating UI on `/users/me/permissions`.

  They now operate on groups instead of a role column.

Design and plan:
[`../plans/2026-10-27-user-groups-and-console-redesign-design.md`](../plans/2026-10-27-user-groups-and-console-redesign-design.md).
