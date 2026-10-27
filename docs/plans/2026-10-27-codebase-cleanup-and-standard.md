# Codebase Cleanup and Cleanup Standard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Shrink and tighten both repos without changing behaviour, add a real-stack Playwright suite, replace the Courts/Venues request fan-out with two summary reads, and write the repeatable cleanup standard.

**Architecture:** The standard and the safety nets (dead-code scripts, e2e seed, Playwright harness) land first; every later refactor commit is then checked by them. Backend gets one new slice, `src/summaries/`, composing existing pricing and booking reads. Frontend gets shared UI in `src/components/ui` and `src/components/layout`, adopted page by page.

**Tech Stack:** FastAPI, SQLAlchemy 2 async, Postgres 17, pytest, ruff, mypy, vulture · Next.js 15, React 19, TanStack Query 5, Tailwind, Vitest, Playwright, knip.

**Spec:** `docs/plans/2026-10-27-codebase-cleanup-and-standard-design.md`

## Global Constraints

- No behaviour change visible to staff; no change to any existing endpoint's request/response shape.
- Only "why" comments survive: non-obvious reasons and invariants (PENDING trap, 404-not-403, per-venue permissions, centavos, Manila day bounds, session pooler). Delete restatements, history, framework explanations.
- Extract a shared component only with two or more real callers and no per-caller flag props.
- `VenueScope` stays a required, default-less argument on every repository method touching venue data.
- Commit straight to `main` in the repo whose files changed; one concern per commit; gates green before each push:
  - backend: `uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest`
  - frontend: `npm run lint:ci && npm run format:check && npm run typecheck && npm test && npm run build` (the build only via the e2e `distDir`, never under the developer's `npm run dev` on 3000), plus `npm run e2e` once Task 5 lands.
- Playwright: real backend on port 8100, Next on port 3100, database `pikol_e2e` on the compose Postgres (host port 5433), Chromium only, not in CI.
- Commit trailer: `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

1. A venue with no courts, or a court with no bookings today — summaries return `[]` / `booked_hours: 0`, never a 500. (Task 7)
2. A caller with `COURT:Manage` but not `PRICE:Manage` or `BOOKING:ReadAny` — the courts summary returns `rate_centavos: null` / `booked_hours: null`, not 403. (Task 7)
3. A lapsed PENDING hold and a maintenance closure today — neither counts toward booked hours. (Task 7)
4. The 12 AM–1 AM slot — counted on its own Manila day, not the previous UTC day. (Task 7)
5. The e2e seed pointed at a database not ending in `_e2e` — exits non-zero and writes nothing. (Task 4)

---

### Task 1: The standard and the baseline

**Files:**
- Create: `docs/standard/README.md`, `docs/standard/cleanup-pass.md` (parent repo)
- Modify: `CLAUDE.md` (parent) — link `docs/standard/` under "Where authoritative docs live"

**Interfaces:**
- Produces: the playbook later tasks follow, and the **baseline table** (section "Baseline 2026-10-27" at the end of this plan) every later task compares against.

- [ ] **Step 1: Record the baseline.** Run and paste results into "Baseline 2026-10-27" below:
  - `find src -name '*.ts*' | xargs wc -l | tail -1` (frontend), `find src -name '*.py' | xargs wc -l | tail -1` (backend)
  - `npx --yes knip@5 --reporter compact | grep -c ':'` (unused exports/types)
  - `grep -rcE '^\s*(//|/\*|\*)' src | awk -F: '{s+=$2} END {print s}'` and `grep -rcE '^\s*#' src --include='*.py' | awk -F: '{s+=$2} END {print s}'`
  - requests on first load of `/staff/courts` and `/staff/venues` (from the throwaway Playwright mock: 1 + 2 per row)
- [ ] **Step 2: Write `docs/standard/README.md`** — one paragraph on what a standard is here, and an index with one line per file.
- [ ] **Step 3: Write `docs/standard/cleanup-pass.md`** with the seven sections the spec lists (measure, what to look for in priority order, safe-change rules, the why-comment rule with keep/delete examples from this codebase, shared component catalogue, verification commands, report format). Commands must be copy-pasteable and reference the scripts Tasks 2–3 add (`npm run deadcode`, `uv run vulture`).
- [ ] **Step 4: Commit** (parent): `docs/standard/*`, `CLAUDE.md`, this plan and the design — `Add the cleanup-pass standard and its first plan`.

### Task 2: Frontend dead-code script and removal

**Files:**
- Create: `pikol-frontend/knip.json`
- Modify: `pikol-frontend/package.json` (devDependency `knip`, script `"deadcode": "knip"`), every file knip flags
- Test: existing Vitest suite

- [ ] **Step 1:** Add knip, configure entries `src/app/**/{page,layout,not-found,error}.tsx`, `next.config.ts`, `tailwind.config.ts`, `vitest.config.ts`, `tests/**/*.ts*`; `project: src/**/*.{ts,tsx}`.
- [ ] **Step 2:** `npm run deadcode` — expect the 32 findings from the survey.
- [ ] **Step 3:** Remove each: drop barrel re-exports nothing imports (tests import from deep paths and keep working), delete unused schemas/types in `features/bookings/schemas.ts` (`CheckoutBooking*`, `BookingPage*`, `HoldRequest`, unused enum schemas — keep what `BookingSchema` composes), `Brand`, `MANILA` export (make it module-private), `SIDEBAR_STORAGE_KEY` export, `DayKpis`/`StaffActionRequest`/`CheckboxState`/`RequestOptions`/`ButtonProps` exports where nothing imports them.
- [ ] **Step 4:** `npm run deadcode` → `✓ No issues`; full frontend gates green.
- [ ] **Step 5: Commit** — `Remove dead exports and add the knip dead-code check`.

### Task 3: Backend dead-code check and removal

**Files:**
- Create: `pikol-backend/vulture_whitelist.py`
- Modify: `pikol-backend/pyproject.toml` (dev dependency `vulture`; `[tool.vulture] paths = ["src", "vulture_whitelist.py"]`, `min_confidence = 60`, `exclude = ["src/database/migrations"]`), `src/parameters/persistence/repository.py`, `src/bookings/dto.py`, `src/bookings/domain/rules.py`
- Test: `tests/bookings/test_booking_domain.py`

- [ ] **Step 1:** `uv run vulture` lists routes, validators, Pydantic fields and middleware hooks; generate the whitelist with `uv run vulture --make-whitelist > vulture_whitelist.py`, then delete from it the three true positives below so they still report.
- [ ] **Step 2:** Delete `ParameterRepository.get_row`, `BookingOut.with_url`, `can_confirm_online_payment` and its three asserts in `test_booking_domain.py`.
- [ ] **Step 3:** `uv run vulture` → no output; backend gates green.
- [ ] **Step 4: Commit** — `Remove dead helpers and add a vulture check`.

### Task 4: E2E seed command

**Files:**
- Create: `pikol-backend/src/devtools/__init__.py`, `pikol-backend/src/devtools/e2e_seed.py`
- Test: `pikol-backend/tests/devtools/test_e2e_seed.py`

**Interfaces:**
- Produces: `uv run python -m src.devtools.e2e_seed` — idempotent on an empty `pikol_e2e`. Seeds, with these exact values (Task 5 depends on them):
  - owner `owner@e2e.test` / `E2e-password-1`, `system_admin=True`, verified
  - venues `E2E Riverside` (courts `Court 1`–`Court 3`, sort 1–3) and `E2E Lakeside` (`Court 1`)
  - front desk `desk@e2e.test` / `E2e-password-1` in Riverside's "Front desk" group
- `assert_e2e_database(url: str) -> None` raises `SystemExit(2)` unless the database name ends in `_e2e`.

- [ ] **Step 1: Write failing tests**

```python
def test_refuses_a_database_not_named_e2e() -> None:
    with pytest.raises(SystemExit) as exit_:
        assert_e2e_database("postgresql+asyncpg://p:p@localhost:5433/pikol")
    assert exit_.value.code == 2

def test_accepts_an_e2e_database() -> None:
    assert_e2e_database("postgresql+asyncpg://p:p@localhost:5433/pikol_e2e")
```

- [ ] **Step 2:** `uv run pytest tests/devtools -v` → FAIL (module missing).
- [ ] **Step 3:** Implement using existing services (`venue_service.create`, `court_service.create`, the CLI's super-admin creation, group membership through the group service) so seeded data passes the same invariants as real data; run the guard before opening any session.
- [ ] **Step 4:** Tests PASS; backend gates green.
- [ ] **Step 5: Commit** — `Add the e2e seed command, refusing any non-e2e database`.

### Task 5: Playwright harness and the shell smoke spec

**Files:**
- Create: `pikol-frontend/playwright.config.ts`, `pikol-frontend/e2e/global-setup.ts`, `pikol-frontend/e2e/fixtures.ts`, `pikol-frontend/e2e/shell.spec.ts`, `pikol-frontend/scripts/e2e-db.sh`
- Modify: `pikol-frontend/package.json` (devDependency `@playwright/test`; scripts `"e2e": "playwright test"`, `"e2e:db": "bash scripts/e2e-db.sh"`), `next.config.ts` (`distDir: process.env.NEXT_DIST_DIR ?? '.next'`), `.gitignore` (`.next-e2e`, `playwright-report`, `test-results`, `e2e/.auth`), `vitest.config.ts` (exclude `e2e/**`)

**Interfaces:**
- Consumes: Task 4's seed command and credentials.
- Produces: `test` from `e2e/fixtures.ts` with fixtures `ownerPage` and `deskPage` (pre-authenticated via `storageState` files written by global setup) and `uid()` returning a short unique suffix for names.

- [ ] **Step 1:** `scripts/e2e-db.sh`: `docker compose -f ../pikol-backend/docker-compose.yml up -d --wait`, drop/create `pikol_e2e` via `docker compose exec postgres`, `uv run alembic upgrade head` and `uv run python -m src.devtools.e2e_seed` in `../pikol-backend` with `DATABASE_URL` for `pikol_e2e`.
- [ ] **Step 2:** `playwright.config.ts`: `globalSetup` runs `e2e:db` then logs in both users through `/login` and saves `e2e/.auth/{owner,desk}.json`; two `webServer`s — backend `uv run uvicorn src.main:app --port 8100` (cwd `../pikol-backend`, env from the design's list, `JWT_SECRET` a fixed 32+ char test value) with `url: http://localhost:8100/health/ready`, and frontend `NEXT_DIST_DIR=.next-e2e NEXT_PUBLIC_API_BASE_URL=http://localhost:8100/api/v1 npm run build && NEXT_DIST_DIR=.next-e2e npx next start -p 3100`; `use.baseURL: http://localhost:3100`; `workers: 1` (one shared database).
- [ ] **Step 3: Write `e2e/shell.spec.ts`**:
  - owner lands on `/staff/schedule` after sign-in; heading visible
  - switching venue via the `Venue` combobox to `E2E Lakeside` puts `venue=` of Lakeside in the URL
  - "Collapse sidebar" shrinks `#staff-sidebar` to 64px; reload keeps it; "Expand sidebar" restores 256px
  - front desk sees no `Courts`, `Venues`, `Parameters` links
- [ ] **Step 4:** `npm run e2e` → 4 passed; vitest still green.
- [ ] **Step 5: Commit** — `Add a real-stack Playwright suite with the console shell spec`.

### Task 6: Page specs

**Files:**
- Create: `pikol-frontend/e2e/{closures,courts,venues,staff-accounts,venue-staff,parameters}.spec.ts`

**Interfaces:**
- Consumes: Task 5 fixtures. Every created row is named with `uid()`.

- [ ] **Step 1: Write the specs** (each test independent):
  - closures: pick `Court 1` + `Court 2`, "All day tomorrow", reason chip "Maintenance", Close courts → one card with both court chips and "· 24 h"; Reopen with a reason → card gone
  - courts: add `Court <uid>` from the dashed tile → card numbered last; rename it; move it earlier → order changes after reload; deactivate → `Inactive` pill
  - venues: add `Venue <uid>` with address → card shows it; Edit name → updated; Courts link opens `/staff/courts?venue=<id>`
  - staff accounts: search `desk@` → one row; `Invited` filter after inviting `inv-<uid>@e2e.test` into Front desk shows that row
  - venue staff: New group `Crew <uid>` → selected with `0 of 9`; tick "Manage courts", Save permissions → toast, stat reads `1 of 9`
  - parameters: edit Hold length to 20 → "Unsaved"; Save → big value `20 min`; set back to 15
  - select keyboard: on Prices, focus Hour, ArrowDown ×2 + Enter → trigger reads `1 AM`
- [ ] **Step 2:** `npm run e2e` → all pass twice in a row (proves isolation).
- [ ] **Step 3: Commit** — `Cover each console page end to end`.

### Task 7: Backend summary reads

**Files:**
- Create: `pikol-backend/src/summaries/{__init__,dto,service}.py`, `pikol-backend/src/summaries/controllers/{__init__,admin}.py`
- Modify: `pikol-backend/src/bookings/persistence/repository.py` (add method), `src/main.py` (include router)
- Test: `pikol-backend/tests/summaries/test_summaries.py` (reuse `tests/venues/conftest.World`)

**Interfaces:**
- Produces:
  - `BookingRepository.booked_hours(session, court_ids: Sequence[int], start: datetime, end: datetime, now: datetime, *, scope: VenueScope) -> dict[int, int]` — reuses `occupancy`'s filters (not RELEASING, PENDING only while `expires_at > now`) and excludes `BookingType.MAINTENANCE`; courts with none are absent.
  - `GET /api/v1/admin/venues/{venue_id}/courts/summary` (`COURT:Manage`) → `list[CourtSummary]`, `CourtSummary{court_id: int, rate_centavos: int | None, booked_hours: int | None}` — rate is the resolved price of the current Manila hour (`pricing_service.resolvers_for`), `None` unless the caller holds `PRICE:Manage`; hours are today's Manila day (`manila_day_bounds`), `None` unless `BOOKING:ReadAny`. Out-of-scope venue → 404.
  - `GET /api/v1/admin/venues/summary` (`VENUE:Manage`) → `list[VenueSummary]`, `VenueSummary{venue_id: int, active_courts: int, booked_hours: int}`.
  - Both declared before any `/{id}` route that would capture `summary`.

- [ ] **Step 1: Write failing tests** — names and assertions:
  - `test_courts_summary_rates_and_hours`: court with a 1 h confirmed walk-in today and a rule at the current hour of 70000 → `{"rate_centavos": 70000, "booked_hours": 1}`; other court → default price, `0`
  - `test_courts_summary_hides_what_the_caller_cannot_see`: caller with only `COURT:Manage` → both fields `None`, status 200
  - `test_booked_hours_ignore_lapsed_holds_and_closures`: lapsed PENDING (expires_at in the past) + maintenance today → `0`
  - `test_booked_hours_count_the_midnight_slot_on_its_manila_day`: slot at `T00:00+08:00` today counts today
  - `test_courts_summary_other_venue_is_404` and `test_venues_summary_is_owner_only` (master → 403)
  - `test_venues_summary_counts_active_courts_and_hours`: venue with 2 active + 1 inactive court → `active_courts == 2`; venue with no courts → `0, 0`
- [ ] **Step 2:** `uv run pytest tests/summaries -v` → FAIL (404 route missing).
- [ ] **Step 3:** Implement repository method, `SummaryService` (`courts_summary(session, venue_id, access: VenueAccess, now) -> list[CourtSummary]`, `venues_summary(session, now, *, scope) -> list[VenueSummary]`), and controller using `get_venue_access`.
- [ ] **Step 4:** Tests PASS; backend gates green; `uv run vulture` clean.
- [ ] **Step 5: Commit** — `Add court and venue summary reads to replace per-card requests`.

### Task 8: Frontend uses the summaries; query defaults

**Files:**
- Modify: `src/features/setup/api.ts`, `schemas.ts`, `keys.ts`, `hooks/use-setup.ts`, `components/court-admin.tsx`, `components/venue-admin.tsx`, `src/lib/query-client.ts`
- Delete: `src/features/setup/booked-hours.ts`
- Test: `tests/unit/setup/court-admin.test.tsx`, `venue-admin.test.tsx`

**Interfaces:**
- Consumes: Task 7 endpoints.
- Produces: `useCourtSummaries(venueId: number)`, `useVenueSummaries()`; keys `setupKeys.courtSummaries(venueId)`, `setupKeys.venueSummaries()`, both invalidated by the existing court/venue/booking/price mutations.

- [ ] **Step 1:** Rewrite the two "rate and booked hours" unit tests to mock `setupApi.courtSummaries` / `venueSummaries` and assert the same rendered text (`₱500`, `3 h`, `1 active`, `2 h`); assert `dayPrices` and `adminBookingsApi.day` are not called.
- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Replace per-card `useDayPrices` / `useVenueDayBookings` / per-venue `useAdminCourts` with the summary hooks; delete `booked-hours.ts` and `useVenueDayBookings` if now unused.
- [ ] **Step 4:** In `query-client.ts` set `defaultOptions.queries.staleTime: 30_000` (matches the Schedule's refresh; mutations already invalidate what they change). Keep any existing per-query overrides.
- [ ] **Step 5:** Unit tests + `npm run e2e` green; Courts makes 2 API calls on load and Venues 2 (record in the plan's results).
- [ ] **Step 6: Commit** — `Load Courts and Venues stats from one summary request`.

### Task 9: Shared UI components

**Files:**
- Create in `pikol-frontend/src/components/ui/`: `section-label.tsx`, `stat-strip.tsx`, `add-tile.tsx`, `pill.tsx`, `avatar.tsx`, `skeleton.tsx`; in `src/components/layout/`: `venue-required.tsx`
- Modify: `button.tsx` (export `buttonVariants`), every page using the patterns (closures form/list, courts, venues, team list, group detail/list, member panel, permission checklist, parameters, venue switcher)
- Test: `tests/unit/lib/ui.test.tsx`

**Interfaces:**
- Produces (exact props):
  - `SectionLabel({ n?: number, children, aside?: ReactNode })` — "01 TITLE … aside"
  - `StatStrip({ items: { label: string; value: ReactNode; meter?: number }[] })` — `<dl>`; `meter` is 0–1 and draws the bar
  - `AddTile({ title, hint, onClick, className? })` — the dashed button
  - `Pill({ tone: 'solid' | 'outline' | 'dashed', children })`
  - `Avatar({ first: string, last: string, shape?: 'square' | 'round' })`
  - `Skeleton({ className? })`
  - `VenueRequired({ children: (venueId: number) => ReactNode })` — the "no venue / loading / error" guard
  - `buttonVariants` used by `Link`s styled as buttons (replaces the three hand-written class strings)

- [ ] **Step 1:** Write `ui.test.tsx`: `SectionLabel n={3}` renders `03` hidden from AT; `StatStrip` renders `dt`/`dd` pairs and a meter only when given; `Avatar` renders `AL` for Ana Lim; `VenueRequired` renders its child with the venue id once loaded.
- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement; adopt everywhere the survey listed; no page-specific props.
- [ ] **Step 4:** Unit + e2e green; `npm run deadcode` clean; frontend `src` line count lower than after Task 8.
- [ ] **Step 5: Commit** — `Extract the console's repeated UI into shared components`.

### Task 10: Simplify the largest frontend files

**Files:**
- Modify: `court-admin.tsx`, `venue-admin.tsx`, `closure-form.tsx`, `select.tsx`, `team-list.tsx`, `group-detail.tsx`, `staff-shell.tsx`, `use-setup.ts`
- Test: existing unit + e2e

- [ ] **Step 1:** Apply `agent-skills:code-simplification` per file: inline single-use helpers, collapse prop drilling where a child only forwards, extract `EntityCard` only if Courts and Venues cards share it without flags, split hooks in `use-setup.ts` by entity if it reads better.
- [ ] **Step 2:** Unit + e2e green after each file; no test changes except removed implementation-detail asserts.
- [ ] **Step 3: Commit** — one commit per file group, e.g. `Simplify the Courts and Venues pages`.

### Task 11: Comment pass, frontend

**Files:** all of `pikol-frontend/src`

- [ ] **Step 1:** Delete non-"why" comments per the standard; keep invariant comments. Target: comment lines at most half the baseline.
- [ ] **Step 2:** Gates green.
- [ ] **Step 3: Commit** — `Keep only comments that explain why`.

### Task 12: Backend simplification and comment pass

**Files:** `pikol-backend/src/**` (focus `bookings/service.py`, `bookings/persistence/repository.py`, `auth/service.py`, `payments/service.py`), `pyproject.toml`

- [ ] **Step 1:** `agent-skills:code-simplification` on the four files: remove duplicated query fragments into private helpers already used twice or more, flatten nested conditionals. Layered slices move together.
- [ ] **Step 2:** Comment pass as Task 11; shorten the B008 essay in `pyproject.toml` to one line.
- [ ] **Step 3:** Backend gates + `npm run e2e` green.
- [ ] **Step 4: Commit** — one per concern (`Simplify the bookings service`, `Keep only comments that explain why`).

### Task 13: Bundle check, review, report, docs

**Files:**
- Modify: this plan (Results section), `pikol-frontend/CLAUDE.md` (e2e commands, shared components, summary hooks), `pikol-backend/CLAUDE.md` (summaries slice, devtools, vulture), parent `CLAUDE.md` current-state line

- [ ] **Step 1:** `next build` route table: split with `next/dynamic` only a route whose First Load JS exceeds the others by more than 30 kB due to a single component; otherwise record "no split needed".
- [ ] **Step 2:** Run `agent-skills:code-review-and-quality` over the full diff since the baseline commit in each repo; fix findings in place.
- [ ] **Step 3:** Fill "Results" with the same measurements as the baseline.
- [ ] **Step 4: Commit** per repo — `Record the cleanup pass and its tooling`.

---

## Baseline 2026-10-27

| Metric | Frontend | Backend |
|---|---|---|
| `src` lines | 9,551 | 8,031 |
| test lines | 4,582 | 5,867 |
| unused exports (knip / vulture true positives) | 32 | 3 |
| comment lines | ~497 | ~234 + ~170 docstrings |
| requests on load: Courts / Venues | 1 + 1 + 2/court · 1 + 2/venue | — |

## Results

| Metric | Frontend before → after | Backend before → after |
|---|---|---|
| `src` lines | 9,551 → 9,336 | 8,031 → 8,210 (+ summaries slice and e2e seed) |
| test lines | 4,582 unit → 4,619 unit + 243 e2e | 5,867 → 6,043 |
| dead-code findings | 32 → 0 (knip, normal and `--production`) | 3 → 0 (vulture) |
| comment lines | 495 → 379 | 234 → 221 |
| requests on load: Courts / Venues | 1 + 1 + 2/court · 1 + 2/venue → 2 · 2 | — |
| largest file | Courts 403 → Select 336 | bookings service 444 → 432 |
| Playwright | none → 12 specs, real stack | — |

Notable changes: shared UI (`SectionLabel`, `StatStrip`, `AddTile`, `Pill`,
`Avatar`, `Skeleton`, `VenueRequired`, `buttonVariants`, `setup-card`); two
summary reads; dead player-flow code removed (`navigate.ts`,
`formatCountdown`, checkout types, `can_confirm_online_payment`,
`get_row`); `lib/storage.ts`; `BookingService._cancel`; route sizes 164–194 kB
First Load JS with no outlier, so no route split.

Left for the next pass: venue switch does not write `?venue=` to the URL in
the production build (found by e2e, pre-existing); the reviewer's deferred
minors (keyed Fragment in `VenueRequired`, summary tests' clock, 100-row
caps, the `javascript:` URL guard the mobile app will need, the e2e
sign-out case).
