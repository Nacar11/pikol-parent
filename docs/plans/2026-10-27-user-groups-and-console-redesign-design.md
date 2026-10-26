# User Groups and Staff Console Redesign — Design

Status: approved for planning, 2026-10-27. The plan built from this is
[`2026-10-27-user-groups-and-console-redesign.md`](2026-10-27-user-groups-and-console-redesign.md).

## Intent

This work has three parts:

1. **Configurable user groups replace the four fixed roles.**
   - Admins create groups, tick their permissions and assign people.
   - Today the roles are rows seeded by a migration, so every change needs
     a deploy. That ends.
   - A customer can become staff by being added to a group. They stay a
     customer.
2. **The web app is for staff only. Customers will use a mobile app.**
   - **Customer** is a real, built-in group. Membership is what lets an
     account book and pay as itself through the API. The mobile app will
     use those routes.
   - **Staff groups** are owned by a venue. Membership is what lets an
     account use the web console at that venue.
   - One account can be in both. It is a customer on mobile and staff on
     the web.
   - The web player pages are removed now. The backend's customer routes
     stay for the mobile app.
3. **The console looks like asima.**
   - The palette is black and white.
   - A dark left sidebar replaces the top navigation bar.
   - The Schedule page takes the reference screenshot's layout: date
     header, two-week strip, legend, one column per court and a "day at a
     glance" panel.
   - Only the layout is taken, **not the texture**: no hatched
     backgrounds, crosshair marks or blue tint.

## Decisions the owner made

Everything in this list came from the owner in the brainstorming
conversation.

- **Venue groups belong to a venue.** Venue A's "Front desk" and Venue
  B's "Front desk" are different groups, and each sees and handles only
  its own venue.
- **Each venue has one venue master group.** Its members are the venue
  masters, the successor of `VENUE_MANAGER`. One person can be master of
  several venues.
- **Groups are managed on two pages:**
  - **Masters → User groups** is for the super admin only. It sets who is
    master of each venue, what masters may do, and who is a customer.
  - **Setup → Venue staff** is for the venue master, and the super admin.
    It manages the venue's own staff groups: create groups, tick
    permissions, add and remove staff. It covers the active venue only.
- **A person can be in many groups, across venues.**
  - Front desk staff can be in Front desk at several venues.
  - A venue master can add anyone to their venue's staff groups. That
    includes staff of other venues and customers.
- **Permissions apply per venue, not summed.** If Ana is master at A and
  Front desk at B, she can edit prices at A but only take walk-ins at B.
- **Protecting masters.**
  - Only the super admin manages the master group.
  - Nobody else can edit or deactivate a master.
  - Nobody can grant a permission they don't hold.
  - Nobody can remove their own ability to manage groups.
- **Customer is a group.** Signing up puts an account in it. One account
  can be a customer and staff at once.
- **The web is staff-only now.**
- **The sidebar is the dark, asima-style one** (mockup option A).
- **The group pages are master-detail** (the approved mockup). The list is
  on the left. The selected group is on the right, with Permissions and
  Members tabs.

## Goals

- Groups are managed at runtime with no deploy. That covers create,
  rename, delete, permissions and members.
- Every venue-owned read and write is authorised against the **active
  venue**. People at Venue A still cannot see Venue B.
- No one but the super admin can do any of these:
  - make someone a master;
  - make someone a customer, or stop them being one;
  - grant an owner-only permission.
- No one can grant a permission they don't hold.
- Customer access and staff access are independent axes, each checked by
  the server.
- The staff console sits in a dark sidebar shell. The shell has a venue
  switcher, permission-filtered sections and a mobile drawer.
- The Schedule page matches the reference layout in black and white.
  Status is carried by shape and label, never by colour alone.

## Non-goals

- The mobile app itself.
- Removing the Staff page. It stays, renamed "Staff accounts", and its
  role picker becomes a group picker.
- A booking-detail side panel. Clicking a booking still opens
  `/staff/bookings/[id]`.
- Dark mode.
- New permission codes. The existing 11 are reused.
- The real Maya provider and deployment (Phase 9).

## Anti-goals

The work fails if any of these happen, even if it technically works:

- A venue master reaches another venue's bookings, groups or members. That
  includes learning they exist from a 403 or a count.
- Anyone other than the super admin mints a master or a customer, or gets
  more than they hold.
- A person's permissions at venue B apply at venue A. That is the summed
  model the owner rejected.
- The server decides customer access or staff access from which app sent
  the request. A client identifier can be forged, so only group
  membership counts.
- A venue master can list every customer's email.
- Status is hidden in colour, or text ships under 4.5:1 contrast.

## Consequence the owner should know

**Until the mobile app ships, customers cannot book online at all.** The
backend keeps every customer route, so the hold, pay, return and My
Bookings flows stay tested and ready. But nothing calls them from a
browser. Walk-ins at the counter are unaffected.

Two more things follow from this:

- The mock checkout page and the checkout return URL still point at the
  web. That only matters once the mobile app has its own return URL. Until
  then they are exercised by backend tests only.
- Verification and reset emails still link to the web, so `/verify-email`
  and `/reset-password` stay.

## Constraints

From the system brief and the existing code:

- **`docs/adr/pikol.dbml` changes first, then the migration.**
- **ADR 0003** records why fixed roles became groups and why "customer"
  is a group.
- The system brief changes in three places:
  - its roles table;
  - the line "a player is role = PLAYER";
  - its description of the frontend.
- **`VenueScope` stays a required argument with no default** on every
  venue-owned repository method. Ownership-after-fetch still returns
  **404, not 403**.
- **Privilege fields are fixed by the server, never read from a body.**
  That covers a group's venue, a group's kind and an invitee's venue.
- **`SUPER_ADMIN` stays the `users.system_admin` bypass.** It is never a
  group.
- UI gating reads `GET /users/me/permissions` (a flat array), never a
  role.
- The backend layered slice (domain → persistence → service →
  controller) lands together.
- Frontend slice rules apply:
  - `api.ts` zod-parses every response.
  - `keys.ts` owns the query keys.
  - `lib/` never imports a feature.
- Nothing is deployed, so the migration needs no expand/contract. It
  still needs a correct backfill and a working downgrade.

## The design

### Data model (decided)

The migration adds three tables. It drops `roles`, `role_permissions`,
`users.role_id` and `users.venue_id`. It keeps `permissions`.

```
user_groups
  id                  pk
  kind                group_kind enum: CUSTOMER | MASTER | STAFF, NOT NULL
  venue_id            fk venues, NULL only for CUSTOMER, on delete restrict
  name                varchar(64) NOT NULL
  created_at, updated_at
  created_by_user_id, updated_by_user_id   fk users, nullable

  CHECK ((kind = 'CUSTOMER') = (venue_id IS NULL))       -- Appendix A
  UNIQUE (kind) WHERE kind = 'CUSTOMER'                  -- exactly one customer group
  UNIQUE (venue_id) WHERE kind = 'MASTER'                -- one master group per venue
  UNIQUE (venue_id, LOWER(name)) WHERE venue_id IS NOT NULL

user_group_permissions
  group_id      fk user_groups on delete cascade
  permission_id fk permissions on delete cascade
  pk (group_id, permission_id)

user_group_members
  group_id          fk user_groups on delete cascade
  user_id           fk users on delete cascade
  added_by_user_id  fk users, nullable
  created_at
  pk (group_id, user_id), index (user_id)
```

**The three kinds of group**

- **CUSTOMER**
  - There is exactly one, it belongs to no venue, and it is built in.
  - It holds **no permission codes**. Its membership *is* the right to use
    the customer routes.
  - Only the super admin changes its members.
  - Registering adds the new account to it.
- **MASTER**
  - There is one per venue, created with the venue in the same
    transaction. It cannot be deleted.
  - It is seeded with today's `VENUE_MANAGER` codes.
  - Only the super admin edits its permissions and members.
- **STAFF**
  - A venue has any number of these.
  - Every new venue gets a "Front desk" group seeded with today's
    `VENUE_STAFF` codes.
  - The super admin, or a master of that venue, manages them. The exact
    rules are below.

**Owner-only codes are never grantable to any group.** `VENUE:Manage` and
`PARAMETER:Manage` belong to the `system_admin` bypass alone. The service
rejects them for every group, including when the caller is the super
admin. A test pins this, because a CHECK constraint cannot span the join.

**The migration backfills existing data.** For each existing venue it
creates the master group and Front desk. It then places each existing
user:

- `PLAYER` → Customer
- `VENUE_MANAGER` → that venue's master group
- `VENUE_STAFF` → that venue's Front desk
- `SUPER_ADMIN` → no group (`system_admin` covers it)

The downgrade reverses this. It picks one membership per user, master
first, then staff, then customer, and restores `role_id` and `venue_id`
from it.

### Two independent axes (decided)

| Axis | Granted by | Checked by | Used by |
|---|---|---|---|
| Customer | Customer membership | `require_customer` (replaces `require_player`) | holds, pay, refresh-payment, `/bookings/me`, own cancel. These are the mobile app's routes. |
| Staff at venue V | memberships in V's MASTER/STAFF groups | `get_venue_access` + `require_permission` | every `/admin/*` route |
| Owner | `users.system_admin` | the existing bypass | everything |

- Being staff never blocks being a customer, and the reverse is also
  true. Kim can book a court as herself on mobile and take walk-ins at
  Venue A on the web.
- Removing someone from Customer stops new customer actions. It does not
  delete their bookings. Staff still handle those through `/admin`.

### The active venue (decided)

A person may hold permissions at several venues, so every staff request
must say which venue it acts in.

- **The console sends `X-Venue-ID: <id>` on every request.**
  `lib/api-client` gets an injected venue provider, the same way it gets
  the token, so `lib/` stays feature-agnostic.
- **`get_venue_access` resolves the header** into
  `VenueAccess(scope: VenueScope, permissions: frozenset[str],
  is_master: bool)`:

  | Caller | Header | Result |
  |---|---|---|
  | super admin | ignored | Scope is all venues, with every code. The owner still filters with the existing `?venue_id=`. |
  | anyone else | names a venue where they have a MASTER or STAFF membership | Scope is that venue. Permissions are the union of their groups **at that venue only**. |
  | anyone else | names any other venue | **403**. Venues are public through the catalog, so this leaks nothing. |
  | anyone else | missing, and they are staff at exactly one venue | That venue is used. |
  | anyone else | missing, and they are staff at several venues | **400 `VENUE_REQUIRED`** |
  | anyone else | missing, and they are staff nowhere | **403** |

- **`require_permission(code)` reads the active venue's set.**
  `get_venue_scope` wraps `get_venue_access`, so `VenueScope` keeps its
  shape and **no repository changes**.
- **`GET /users/me/permissions`**
  - With the header, it returns that venue's codes.
  - Without the header, it returns the union. Only coarse client checks
    use the union, such as "is this person staff anywhere?". **The server
    never authorises on the union.**
- **New `GET /users/me/venues`** returns
  `[{venue_id, name, is_master, groups: [name]}]`. For the super admin it
  returns every venue with `groups: []`. It feeds the sidebar switcher.
- **`GET /users/me` gains `is_customer: bool`.** `role` and `venue_id`
  are removed from `UserOut`. In their place, admin user reads carry
  `groups: [{id, name, kind}]`, filtered to the caller's view.

### Group management rules (decided)

The API stays one resource, `/api/v1/admin/user-groups`. The two pages are
two views of it.

| Action | Super admin | Master of venue V | Holder of `USER:Manage` at V who is not a master |
|---|---|---|---|
| Customer group: read or change members | ✅ | ❌ (404) | ❌ (404) |
| Master group of V: change permissions or members | ✅ | read-only | read-only |
| Create, rename or delete a STAFF group at V | ✅ | ✅ | ✅ |
| Set a STAFF group's permissions | any non-owner code | see the permission-setting rule below | same as master |
| Add a member to a STAFF group at V | any user | any user, found by exact email | same |
| Remove a member from a STAFF group at V | ✅ | ✅ | ✅, but never a master of V |
| Edit, deactivate, resend or revoke the invitation of a **master of V** | ✅ | ❌ | ❌ |
| Edit or deactivate anyone else | ✅ | only if all their memberships are at V | same |

**Creating groups**

- When anyone but the super admin creates a group, the **server** sets
  `venue_id` from the active venue and `kind = STAFF`.
- A body carrying `venue_id` or `kind` is a 422 (`extra="forbid"`).
- The super admin's create takes `venue_id`.

**Setting permissions**

- `PUT /admin/user-groups/{id}/permissions {codes}` replaces the whole
  set.
- A non-super-admin may **add or remove only codes they hold at V**.
- Codes already on the group that they don't hold are **kept**,
  untouched. The UI shows them ticked and disabled, with "you don't have
  this permission".

**No self-lockout**

The server refuses two changes with 409 `SELF_LOCKOUT`:

- removing `USER:Manage` from a group the caller belongs to;
- removing the caller from their last `USER:Manage` group at V.

**Deleting**

- A group with members cannot be deleted. The server returns 409
  `GROUP_NOT_EMPTY`.
- Master and Customer groups cannot be deleted at all.

**Members**

- `GET /admin/user-groups/{id}/members` lists the members.
  `POST /admin/user-groups/{id}/members {email}` adds one.
  `DELETE /admin/user-groups/{id}/members/{user_id}` removes one.
- A member can be any non-deleted user who is active or has a pending
  invitation. Deactivated users are refused.
- Non-super-admins add members by **exact email**. An exact match finds
  that one person and lists nobody else. The super admin also gets
  `GET /admin/users?q=` search.
- A master never sees a person's memberships at other venues.

**Invitations**

- `POST /admin/venue-staff {email, first_name, last_name, mobile,
  group_id}` invites someone new. The group must be a STAFF group at the
  active venue.
- `POST /admin/users` is the super admin's version. It takes
  `{..., group_id}` and accepts any group, master included.
- An email that already has an account is refused with **409
  `EMAIL_EXISTS`**: "Already registered — add them from Venue staff."
  Invitations never add to an existing account.

### Web is staff-only (decided)

- **These routes are removed from the frontend:**
  - `/venues`
  - `/venues/[venueId]`
  - `/bookings`
  - `/bookings/[bookingId]`
  - `/checkout/return`
  - `/register`
- **These slices are removed with them:**
  - `features/bookings`
  - `features/availability`
  - `features/catalog`

  The exception is code the console reuses: the day grid's selection
  reducer and the catalog venue names. That code moves into the slice
  that uses it rather than being deleted.
- **These routes stay:**
  - `/login`
  - `/forgot-password`
  - `/reset-password`
  - `/verify-email`
  - `/accept-invite`
- **Landing**
  - A signed-in user with staff access at any venue goes to
    `/staff/schedule`. Staff access means `system_admin`, or the union
    permissions contain `BOOKING:ReadAny`.
  - Anyone else lands on `/customer-app`, a plain page: "Pikol for
    players is coming to mobile." It has a sign-out button.
- **The login page loses its "Create an account" link.** Registration
  stays an API route, `POST /auth/register`, for the mobile app.

### Sidebar shell (decided)

- **`(app)/layout.tsx` keeps `AuthGate`.** `staff/layout.tsx` renders the
  new `StaffShell`. The old top-nav `AppShell` and `AppNav` are deleted.
- **`StaffShell` follows asima's `app-shell` shape:**
  - a fixed `w-64` `bg-neutral-950` sidebar, persistent from `md` up;
  - below `md`, a drawer with a backdrop that closes on Escape and on
    route change;
  - content offset by `md:pl-64`;
  - no desktop top bar; a slim mobile top bar with the hamburger and the
    brand.
- **The sidebar, top to bottom:**
  - Brand: "pikol" with a small "STAFF" label.
  - Venue switcher, fed by `/users/me/venues`. It is a control when there
    is more than one venue or the caller is the super admin. Otherwise it
    is a static label.
  - **Operations:** Schedule, To resolve (the queue count as a badge),
    Closures.
  - **Setup:** Courts, Venue staff, Staff accounts, Venues, Parameters.
  - **Masters:** User groups. It is shown only to the super admin, gated
    on `VENUE:Manage`. That code is owner-only by construction.
  - At the bottom, set apart: the signed-in person (initials, name, their
    groups at the active venue, or "Super admin") and sign-out.
  - Each item appears only when its permission is held at the active
    venue. An empty section has no header.
- **The active venue lives in the URL as `?venue=`**, so it can be
  deep-linked. It is remembered in `localStorage` inside try/catch. The
  fallback is the first entry of `/users/me/venues`. Switching venue
  changes the header and invalidates every staff query, including
  permissions.
- **Pricing has no sidebar item.** It is reached per court from Courts, as
  today.

### Palette (decided)

- **Tokens in `globals.css` become neutral**, asima's black on white:

  | Token | Value |
  |---|---|
  | background | `#fafafa` |
  | surface | `#fff` |
  | foreground, primary | `#0a0a0a` |
  | muted | `#f5f5f5` |
  | muted-foreground | `#525252` |
  | border | `#e5e5e5` |
  | ring | `#0a0a0a` |

- `accent` and `success` collapse to neutrals.
- **`destructive` stays red.** It is the only hue, used only for errors
  and destructive actions, as in asima.
- No raw colours appear in components. The sidebar uses neutral-scale
  utilities, as asima's does.
- Inter stays as the font. Numbers in grids, KPIs and times use
  `tabular-nums`.

### Schedule page (decided)

The layout follows the reference, flat and monochrome. The page has six
parts.

**1. Header**

- The breadcrumb `SCHEDULE / <venue>`.
- The Manila date, e.g. "Monday, 7 September", with "2026 · Week 37" next
  to it.
- On the right: **Today**, previous and next day, and the one primary
  CTA, **New booking**.

**2. Two-week strip**

- Fourteen days starting from the Monday of the selected week, with ‹ ›
  to page by week.
- Each day shows its weekday, its date and a thin utilisation bar.
- The selected day is inverted.

**3. Legend**

| Status | Drawn as |
|---|---|
| Open | white |
| Confirmed | black outline |
| Awaiting payment | dashed outline |
| In progress (derived) | black fill |
| Completed | grey fill |
| No-show | grey fill with a struck label |
| Closed | diagonal hatch |

- The hatch is the one sanctioned pattern, because it is status, not
  texture.
- Right-aligned on the same row: "9 of 48 court-hours booked".

**4. Court grid**

- Time runs down the left, with one column per court.
- Each court header shows:
  - a large two-digit number;
  - the name;
  - "₱500 / hr";
  - a booked-hours bar;
  - "4 of 24 h booked".
- A booking is **one block spanning its hours**. It shows the time range,
  the player or "Walk-in", and an uppercase status label. The label is
  always present.
- The grid scrolls inside its frame. Today it starts at the current hour.
- Tapping an open cell starts the existing walk-in selection.
- **New booking** selects the next open hour on the first court and
  raises the existing walk-in bar.
- A block links to `/staff/bookings/[id]`.

**5. "Day at a glance" panel**

- At `xl` and above it sits on the right, 320 px wide. Below `xl` it
  stacks under the grid.
- Four KPI tiles:
  - **Booked:** court-hours, of N bookable.
  - **Utilisation:** a percentage, with the booking count.
  - **Revenue:** frozen totals including unpaid, excluding cancelled and
    expired.
  - **Awaiting payment:** the sum and count of live UNPAID.
- Below the tiles, the day's booking list: time, player, "Court 1 · 2 h",
  and a status chip.
- A footer hint.

**6. Keyboard:** ← and → change the day when focus is not in a field.

**Data**

- The grid and per-court figures come from the existing availability read
  and `/admin/bookings?date=`. The availability read becomes staff-usable:
  `GET /venues/{id}/availability` keeps its auth and adds no scope, since
  it is public catalog data.
- The strip needs one new read:
  - `GET /admin/bookings/occupancy?from=YYYY-MM-DD&days=14` returns
    `[{date, booked_hours, bookable_hours}]`.
  - It is scoped by `VenueScope`, with `venue_id` for the super admin.
  - It uses **Manila day boundaries**.
  - It applies the **PENDING trap**: PENDING counts only while
    `expires_at > now()`.
  - MAINTENANCE hours are subtracted from bookable, not counted as booked.
  - Bookable is active courts × 24. The venue stores no opening hours.

### Masters → User groups (super admin)

`/staff/groups` is gated on `VENUE:Manage` and uses the master-detail
layout of the approved mockup.

**Left list**

- **Customer** (built-in, all venues), then one row per venue:
  "Venue master · Makati Courts", with its member count.
- A filter box above the list.

**Customer group**

- It has only a **Members** tab, with no permissions to set.
- Members are found by name or email search. They are paginated, with
  add and remove.
- One line explains: "Customers can book and pay in the mobile app."

**Venue master group**

- **Permissions** tab:
  - plain-English checkboxes, in three sections:
    - **Bookings:** see all, walk-ins, mark paid, check in / no-show,
      cancel.
    - **Venue setup:** closures, courts, pricing, users & groups.
    - **Owner only:** venues, parameters. These are always disabled, with
      "super admin only".
  - Save and Discard in a footer. Leaving with unsaved changes asks
    first.
- **Members** tab:
  - search, then add;
  - remove, with a confirmation.

### Setup → Venue staff (master and super admin)

`/staff/venue-staff?venue=` is gated on `USER:Manage`. It covers the
active venue only, with the same master-detail layout.

**Header**

- The breadcrumb `SETUP / VENUE STAFF / <venue>`.
- The primary **New group** button. It opens an inline name field and
  creates a STAFF group with no permissions.

**Left list**

- The venue's STAFF groups, each with its member and permission counts.
- The master group pinned at the top, read-only, labelled "Managed by the
  owner". It is visible so the master can see who the masters are.

**Group detail**

- The group name, with **Rename** and **Delete**.
- Delete is red and asks for confirmation. When the group has members it
  is disabled, with the reason shown.
- **Permissions** tab:
  - the same checkboxes as the master group;
  - codes the caller lacks are disabled;
  - owner-only codes are always disabled;
  - a self-lockout attempt shows the server's message inline.
- **Members** tab:
  - "Add by email". A not-found result gets a clear message, not an empty
    list.
  - Each member shows initials, name, email, and an "also a customer"
    tag.
  - Remove asks for confirmation.
- Saving is never optimistic. The server response is written back.

**Empty and loading states**

- When the venue has only Front desk, an empty state explains what groups
  are for.
- Loading shows skeletons, not spinners.

### Staff accounts (renamed from "Staff")

- The invite form's role select becomes a **group select**. It lists the
  active venue's STAFF groups, plus every group for the super admin.
- The list shows each person's groups at the active venue.
- `EMAIL_EXISTS` renders as an inline field error pointing to Venue staff.

### UI quality bar

These come from the ui-ux-pro-max review and apply to every screen in this
work.

- Contrast is at least 4.5:1. Secondary grey is no lighter than `#525252`
  on white.
- Focus rings are always visible, and the sidebar has a skip-to-content
  link.
- Every nav item has a lucide icon (one stroke width) and a text label.
  The active item sets `aria-current`.
- Touch targets are at least 44 px in the drawer and toolbars.
- Status is never shown by colour alone.
- Destructive actions confirm and stay visually separate. Saves show
  loading, then a toast.
- Motion is 150–200 ms, on transform and opacity only, and respects
  `prefers-reduced-motion`.
- There is no horizontal page scroll at 375 px. Only the court grid
  scrolls sideways, inside its frame.

## Left to the builder

- Component decomposition and file names inside slices.
- How `VenueAccess` is cached per request. The exact occupancy SQL is also
  open, provided it obeys the Manila and PENDING rules.
- The venue switcher's control type. It must be keyboard-operable.
- The empty, loading and error copy.
- Test layout. Every row of the management-rules table needs a test that
  fails when that rule is removed.

## Calls the builder made

These are marked for the owner to check:

1. Non-super-admins add members by **exact email**, not search, so the
   picker cannot list every customer.
2. A venue master can **deactivate** only people who work solely at their
   venue. For anyone else they remove them from their groups.
3. A group with members **cannot be deleted**. The members must be moved
   first.
4. Codes a caller doesn't hold are **kept** on a group they edit, shown
   ticked and disabled.
5. Inviting an email that already exists is **refused** (`EMAIL_EXISTS`),
   pointing to Venue staff.
6. People with a **pending invitation** can be added to more groups.
   Deactivated people can't.
7. The old "Staff" page is renamed **Staff accounts**, so it isn't
   confused with Venue staff.
8. **Pricing** has no sidebar item. It stays reached from Courts.
9. **`destructive` stays red**, as in asima.
10. Every venue starts with **Venue master + Front desk**.
11. Web registration is removed. `POST /auth/register` stays for mobile.
