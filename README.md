# Pikol

Pickleball court booking system — single owner, many venues, many courts,
clock-aligned hourly slots, online payments via PayMongo.

## Layout

```
pikol-parent/        # this repo — docs, plans, ADRs, system-level CLAUDE.md
├── pikol-backend/   # FastAPI + SQLAlchemy + Alembic + Postgres  (own git repo)
└── pikol-frontend/  # Next.js 15 App Router                      (own git repo)
```

The two sub-apps are separate git repositories that sit as siblings under this
parent, the same arrangement as `asima-parent`. Committed plan snapshots and
ADRs live **only** here, regardless of which repo the feature belongs to.

## Status

Design in progress — see
[`docs/plans/2026-09-09-court-booking-system-design.md`](docs/plans/2026-09-09-court-booking-system-design.md).
No application code yet.
