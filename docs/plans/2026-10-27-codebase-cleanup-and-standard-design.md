# Codebase cleanup and the cleanup standard — design

Date: 2026-10-27. Covers `pikol-backend` and `pikol-frontend`; the standard
itself lives in this parent repo.

## Intent

The console redesign landed fast, page by page. It left repeated UI, dead
leftovers from the removed web booking flow, pages that fan out one request
per row, and a lot of comments that narrate rather than explain. The owner
wants the codebase to get **measurably smaller and tighter without changing
what staff see or do**, and wants that pass to be **repeatable**: a written
standard that a future session (human or agent) can pick up and run again.

Success, in the owner's words: less code, shared components, no dead code,
Playwright tests.

## Deliverables

1. **`docs/standard/`** in the parent repo, with one file specialised for
   this task: `docs/standard/cleanup-pass.md`, a playbook for a
   cleanup/optimisation pass across both repos.
2. **The first cleanup pass**, executed against that playbook, across both
   repos.
3. **A Playwright end-to-end suite** in `pikol-frontend`, running against the
   real backend and a Postgres test database.

## Goals

- **Less code.** Net line count of `src/` goes down in both repos; reported
  before and after.
- **Shared components** for every UI pattern repeated across pages (survey of
  2026-10-27): section label with the "01" step number, stat strip (`<dl>`
  of label/value cells), dashed "add" tile, status pill, initials avatar,
  link styled as a button, loading skeleton, the "no active venue yet" guard,
  and a card shell shared by Courts and Venues.
- **No dead code.** `knip` reports zero unused exports, files and
  dependencies on the frontend; `vulture` reports no true positives on the
  backend (false positives — routes, validators, Pydantic fields, migration
  hooks — go in a whitelist file, not in code changes).
- **Comments explain why, nothing else.** Keep comments that state a
  non-obvious reason or an invariant (the PENDING trap, 404-not-403,
  per-venue permissions, money as centavos, Manila day boundaries, prepared
  statements on the session pooler). Delete comments that restate the code,
  narrate history ("Phase 5 added…"), or explain a framework idiom.
- **Faster pages.**
  - Courts and Venues stop fanning out. One backend read returns, per
    court of a venue, the current resolved rate and today's booked hours;
    one returns, per venue, active-court count and today's booked hours.
  - React Query gets a deliberate default `staleTime` so page-to-page
    navigation does not refetch everything; the Schedule keeps its 30-second
    refresh.
  - Bundle splitting only where `next build` output shows a page carrying a
    clearly avoidable cost.
- **Playwright suite** covering, against the real stack: sign in, venue
  switch, sidebar collapse, and the core flow of each redesigned page
  (Closures create + reopen, Courts add/rename/reorder/deactivate, Venues
  add/edit, Staff accounts search/filter/invite, Venue staff group
  create + permission save, Parameters edit, the Select dropdown by keyboard).

## Non-goals

- No product changes: no new screens, fields, flows or copy beyond what
  shared components force (e.g. identical labels).
- No change to any existing endpoint's request or response shape. The mobile
  app will consume the customer routes; they stay compatible.
- No CI wiring for Playwright in this pass (migrations deliberately run only
  from a laptop; the suite runs locally with one command).
- No visual redesign. Tokens, layout and copy stay as they are now.

## Anti-goals (a failure even if it "works")

- A behaviour change hidden inside a refactor. Each simplification commit
  must be behaviour-preserving and pass the existing tests unchanged, except
  where a test asserted an implementation detail that was removed.
- Abstractions with one caller, or "shared" components that need a prop per
  page to cover their differences. Extract only what is genuinely the same.
- Deleting a "why" comment that guards a real trap (see list above).
- Weakening any venue-scope, permission or money rule while simplifying.
- A Playwright suite that is flaky, depends on wall-clock luck, or leaves
  data behind in the developer's database.

## Constraints

- Repo rules from `CLAUDE.md` (parent, backend, frontend) apply: layered
  backend slices move together; `VenueScope` stays a required argument;
  `docs/adr/pikol.dbml` changes before any migration (none expected);
  commits go straight to `main`, one concern each, checks green before
  every push.
- Gates per commit: backend `ruff`, `mypy --strict`, `pytest`; frontend
  `lint:ci`, `format:check`, `typecheck`, `test`, `build`; plus the
  Playwright suite once it exists.
- `npm run build` must not run under the developer's live `npm run dev`
  (it overwrites `.next`); e2e uses its own build output directory and port.

## Decided

- **Playwright runs against the real backend and a Postgres test database**
  (owner's decision).
- **Only "why" comments stay** (owner's decision).
- **"Optimise" means both faster pages and cleaner code** (owner's
  decision).
- **New backend reads are allowed when they replace a frontend fan-out**,
  following the layered slice pattern with tests and `VenueScope`
  (my call, agreed in conversation).

## The e2e harness (my calls)

- **Database:** a separate database `pikol_e2e` in the existing
  `docker-compose` Postgres (host port 5433). The harness drops and
  recreates it, runs `alembic upgrade head`, then seeds through a small
  backend CLI command (`python -m src.e2e.seed` or similar) that creates the
  owner, two venues, courts and a known staff member. Seed data lives in
  code, not SQL dumps.
- **Processes:** Playwright's `webServer` starts the backend (uvicorn, port
  8100, `DATABASE_URL` pointed at `pikol_e2e`, mock payment provider, memory
  email sender) and the frontend (`next build` into a separate `distDir`,
  `next start` on port 3100, `NEXT_PUBLIC_API_BASE_URL` at 8100). CORS for
  3100 is set through the backend's env for that run only.
- **Backend env for the run** (all existing settings, no new code paths):
  `DATABASE_URL=…/pikol_e2e`, `CORS_ALLOWED_ORIGINS=["http://localhost:3100"]`,
  `FRONTEND_BASE_URL=http://localhost:3100`, `EMAIL_PROVIDER=console`,
  `PAYMENT_PROVIDER=mock`, `RATE_LIMIT_ENABLED=false` (every spec logs in
  from one IP, which would trip the login limiter).
- **The seed command refuses to run** unless the database name ends in
  `_e2e`, so it can never be pointed at a real database. It lives in the
  backend (it needs the services and password hashing) but is a dev tool,
  not an API route.
- **Isolation:** tests that write create their own uniquely named rows
  (timestamp suffix), so tests are order-independent; the database is reset
  at the start of each full run, never mid-run.
- **Location:** `pikol-frontend/e2e/`, `playwright.config.ts` at the repo
  root, `npm run e2e`. Chromium only.
- **Auth in tests:** one global setup logs in through the UI once per role
  and stores `storageState`; specs reuse it.

## The standard (`docs/standard/cleanup-pass.md`)

A checklist-driven playbook, written for an agent to execute, with:

1. **Measure first:** the exact commands for line counts, `knip`,
   `vulture`, comment density, largest files, `next build` route sizes, and
   per-page request counts (from the Playwright run).
2. **What to look for**, in priority order: dead code; duplication across
   pages (with the extraction bar: two or more real callers, no
   per-caller flags); fan-out requests; comments that are not "why";
   oversized files; tests that assert implementation details.
3. **How to change things safely:** one concern per commit, behaviour
   preserved, existing tests green before and after, layered slices moved
   together, never weaken scope/permission/money rules.
4. **The "why"-comment rule** with keep/delete examples from this codebase.
5. **Shared component catalogue:** where shared UI lives
   (`src/components/ui`, `src/components/layout`) and the current list,
   so the next pass extends it instead of re-inventing.
6. **Verification:** the gate commands for both repos plus `npm run e2e`,
   and the before/after report format.
7. **Report:** a short before/after table (lines, unused exports, comment
   lines, requests per page) appended to the pass's plan in `docs/plans/`.

The `docs/standard/` folder gets a short `README.md` index so later
standards (e.g. testing, API) can sit beside it.

## Settled in the builder check (my calls)

- **Permissions on the summary reads:** the courts summary is gated like the
  Courts page (`COURT:Manage` at the active venue); rate and booked-hours
  fields are filled only when the caller also holds `PRICE:Manage` /
  `BOOKING:ReadAny`, matching what the page shows today. The venues summary
  is gated like the Venues page (`VENUE:Manage`, owner only).
- **Booked hours on the server** use the same rule the frontend uses now:
  confirmed, completed and no-show hours count; PENDING counts only while
  `expires_at > now()`; maintenance never counts. The frontend helper
  `booked-hours.ts` is then deleted.
- **Dead-code tools become repo scripts:** `npm run deadcode` (knip, as a dev
  dependency with a config) and a backend `vulture` invocation with a
  whitelist file, so the standard's commands are one line each.
- **Dead backend code confirmed in the survey:**
  `ParameterRepository.get_row`, `BookingOut.with_url`, and
  `can_confirm_online_payment` (only its own test calls it) — removed with
  their tests.

## Left to the builder

- The exact names and props of shared components, within the extraction bar.
- The exact shape of the two summary endpoints (paths under
  `/api/v1/admin/...`, field names in snake_case), as long as they are
  venue-scoped, owner-safe and tested.
- Which large files to split versus simplify in place.
- The default `staleTime` value, justified in the commit message.
- The order of commits, as long as the standard and the e2e harness land
  before the bulk of the refactor so every later commit is checked by them.
