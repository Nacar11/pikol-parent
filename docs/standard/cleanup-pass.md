# Standard: the cleanup pass

A cleanup pass makes `pikol-backend` and `pikol-frontend` smaller, more
shared and better tested **without changing what anyone sees or does**. Run
it after a feature push, before the next one starts. One pass is one plan in
`docs/plans/YYYY-MM-DD-cleanup-pass.md` that follows this file and ends with
its report.

First run: [2026-10-27](../plans/2026-10-27-codebase-cleanup-and-standard.md).

## 1. Measure first

Record every number before touching code; the report compares against them.

```bash
# frontend (pikol-frontend/)
find src -name '*.ts*' | xargs wc -l | tail -1          # src lines
find tests e2e -name '*.ts*' | xargs wc -l | tail -1    # test lines
npm run deadcode                                         # knip: unused files/exports/deps
grep -rcE '^\s*(//|/\*|\*)' src | awk -F: '{s+=$2} END {print s}'   # comment lines
find src -name '*.ts*' | xargs wc -l | sort -n | tail -10            # largest files
NEXT_DIST_DIR=.next-e2e npm run build                    # route table: First Load JS

# backend (pikol-backend/)
find src -name '*.py' | xargs wc -l | tail -1
uv run vulture                                           # dead code (whitelist: vulture_whitelist.py)
grep -rcE '^\s*#' src --include='*.py' | awk -F: '{s+=$2} END {print s}'
find src -name '*.py' | xargs wc -l | sort -n | tail -10
```

Requests per page: run `npm run e2e` and count `/api/v1` requests on first
load of each console page (Playwright `page.on('request')`). A page whose
count grows with its rows is a fan-out.

## 2. What to look for, in this order

1. **Dead code.** Anything `knip` or `vulture` reports, after checking it is
   not a framework entry point. Remove the code and its tests together. A
   false positive goes in `knip.json` or `vulture_whitelist.py` with no
   further ceremony — never keep dead code "just in case".
2. **Duplication across pages.** The same markup, class string or logic in
   two or more places. Extract only when every caller is genuinely the same
   thing: if the shared version needs a flag prop per caller, leave them
   apart.
3. **Fan-out requests.** A page that fires one request per row. Replace with
   one read that returns the rows' data, added as a backend slice (domain →
   persistence → service → controller) with tests, scoped like the page.
4. **Comments that are not "why".** See §4.
5. **Oversized files.** Over ~250 lines, or doing two jobs. Simplify in place
   first (inline single-use helpers, flatten conditionals, drop forwarding
   props); split only along a real seam.
6. **Tests that assert implementation.** A test that breaks on a refactor
   with no behaviour change is asserting the wrong thing; rewrite it against
   what the user sees or what the API returns.

## 3. Changing things safely

- **Behaviour is frozen.** No new screens, fields, copy or endpoint shapes.
  The only new code a pass adds is a read that replaces a fan-out, a shared
  component, a test, or tooling.
- **One concern per commit**, in the repo it touches, straight to `main`,
  gates green before each push (§6).
- **Existing tests stay green unchanged**, except a test that asserted a
  removed implementation detail — rewrite it, say so in the commit.
- **Never weaken a rule while simplifying:** `VenueScope` stays required with
  no default; `/{id}` misses stay 404; permissions stay per venue; money stays
  integer centavos; day boundaries stay Manila; PENDING is live only while
  `expires_at > now()`.
- **Backend slices move together.** A half-landed layer is worse than none.

## 4. The "why" comment rule

Keep a comment only when it says something the code cannot: a reason, an
invariant, a trap. Delete everything else.

| Keep | Delete |
|---|---|
| `# PENDING never means "holding" on its own — add expires_at > now()` | `# Get the bookings for the venue` |
| `// 404, not 403: a 403 confirms Venue B's court exists` | `// Phase 5 added this when the mock landed` |
| `# Session pooler on 5432: asyncpg's prepared statements break on 6543` | `// useEffect runs after render` |
| `// Price is frozen at creation; never recompute from price_rules` | `/** The Courts component. */` |

A docstring follows the same rule: one line of why, or none.

## 5. Shared component catalogue

Shared UI lives in `pikol-frontend/src/components/ui` (primitives) and
`src/components/layout` (page structure). Before writing markup on a page,
check here; after a pass, update this list.

| Component | Use for |
|---|---|
| `Button`, `buttonVariants` | every button, and links that look like one |
| `Input`, `Label`, `Select` | form fields; `Select` replaces every native `<select>` |
| `Alert` | errors (`role=alert`) and notices |
| `PageHeader` | breadcrumb, title, one line of context, the page's primary action |
| `SectionLabel` | the "01 TITLE" heading of a numbered section |
| `StatStrip` | label/value cells under a card heading, with optional meter |
| `AddTile` | the dashed "add a thing" tile at the end of a grid or list |
| `Pill` | status labels — text, never colour alone |
| `Avatar` | initials for a person |
| `Skeleton`, `PageLoading` | loading placeholders |
| `VenueRequired` | pages that need the active venue before rendering |

## 6. Verification

Every commit:

```bash
# backend
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest && uv run vulture
# frontend
npm run lint:ci && npm run format:check && npm run typecheck && npm test && npm run deadcode
NEXT_DIST_DIR=.next-e2e npm run build     # never plain `npm run build` under a running `npm run dev`
npm run e2e                               # real backend :8100, Next :3100, database pikol_e2e
```

`npm run e2e` resets `pikol_e2e`, seeds it with
`python -m src.devtools.e2e_seed` (which refuses any database not named
`*_e2e`) and runs the Playwright suite. A pass is not done until it is green
twice in a row.

## 7. Report

Append to the pass's plan:

| Metric | Frontend before → after | Backend before → after |
|---|---|---|
| `src` lines | | |
| test lines (unit + e2e) | | |
| dead-code findings | | |
| comment lines | | |
| requests on load, per fan-out page | | |
| largest file | | |

Then one line per notable change, and anything deliberately left for the
next pass.
