# Phase 1: Backend Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up `pikol-backend` as a running FastAPI service with typed
configuration, an async Postgres connection, Alembic migrations, the seeded
`parameters` table behind a validated typed accessor, and green CI.

**Architecture:** FastAPI with per-feature packages (`controllers/ domain/
dto/ persistence/`), mirroring asima's NestJS module layout. SQLAlchemy 2.0
async over asyncpg; Alembic for migrations. Configuration and parameters are
two distinct things: **config is environment** (secrets, connection strings,
read once at boot) and **parameters are business rules** (prices, windows,
caps, stored in Postgres and tunable without a deploy).

**Tech Stack:** Python 3.12+, FastAPI, SQLAlchemy 2.0 (async) + asyncpg,
Alembic, Pydantic v2 + pydantic-settings, pytest + pytest-asyncio + httpx,
ruff, mypy, uv, Docker Compose (local Postgres only).

**Spec:** [`2026-09-09-court-booking-system-design.md`](./2026-09-09-court-booking-system-design.md)
**Roadmap:** [`2026-09-09-implementation-roadmap.md`](./2026-09-09-implementation-roadmap.md)

## Global Constraints

These apply to every task in every Pikol backend plan.

- **snake_case end-to-end** — DB column → domain → JSON wire payload. No
  camelCase translation layer at the boundary. (spec §3.5)
- **All API routes under `/api/v1/...`**, from the shared `API_PREFIX`
  constant. `/health` is deliberately unversioned — it is infrastructure, not
  API. (spec §3.5)
- **Money is integer centavos**, never float, never `Decimal` in the database.
- **Times are `timestamptz` stored UTC.** Every day-bounded query uses
  Manila-local boundaries, never `::date`. (spec §4)
- **Request schemas set `extra="forbid"`** — the equivalent of asima's
  `forbidNonWhitelisted: true`. (spec §3.3)
- **No `if (env)` branches in application code.** Environment differences are
  expressed as configuration and adapter selection only. (spec §5.6)
- **Enums are native PostgreSQL types**, not `varchar`. These are closed sets
  defined by a state machine, and the design leans on the database refusing
  bad data — a typo'd `'CONFIRMEND'` must be impossible, not merely unlikely.
  The cost: Alembic autogenerate does **not** handle enum changes, so adding a
  value is always a hand-written `ALTER TYPE ... ADD VALUE` migration, and
  `create_type=False` plus an explicit `.create()` is required so the type is
  made exactly once.
- **The import root is `src`** (`from src.parameters... import ...`), matching
  the spec's file paths and asima's `src/` layout.
- **Every read of `booking_status = 'PENDING'` also filters
  `expires_at > now()`.** Not applicable until Phase 4, listed here because it
  is a project-wide rule with no exceptions. (spec §4.3)

## Working rule: format before you commit

Run `uv run ruff format .` before every commit. The code blocks in this plan
show **content, not final formatting** — magic trailing commas and line
breaks will differ from ruff's output, and CI runs `ruff format --check`.
Formatting first means the gate never fails on whitespace, and the diff you
commit is the one ruff would produce.

## File Structure

```
pikol-backend/
├── pyproject.toml            deps, ruff, mypy, pytest config
├── .python-version           pins the interpreter — 3.12, matching mypy/ruff
├── docker-compose.yml        local Postgres only
├── alembic.ini
├── .env.example              every var, with comments
├── .gitignore
├── README.md
├── src/
│   ├── main.py               app factory, middleware, router registration
│   ├── config/settings.py    environment — read once at boot, fail fast
│   ├── database/
│   │   ├── base.py           DeclarativeBase
│   │   ├── session.py        async engine + get_session dependency
│   │   └── migrations/       alembic env.py + versions/
│   ├── health/controllers/health.py   /health (liveness) + /health/ready (DB)
│   ├── parameters/
│   │   ├── domain/parameters.py       typed model + cross-field validation
│   │   ├── persistence/models.py      SQLAlchemy model
│   │   ├── persistence/repository.py  data access
│   │   └── service.py                 TTL-cached accessor
│   └── utils/
│       ├── api.py            API_VERSION, API_PREFIX
│       ├── request_id.py     X-Request-ID middleware + ContextVar
│       ├── log.py            JSON lines carrying the request_id
│       ├── errors.py         error envelope + exception handlers
│       ├── pagination.py     Page[T] — the list envelope
│       └── time.py           Manila day boundaries
└── tests/
    ├── conftest.py                      client + rolling-back db_session
    ├── health/test_health.py
    ├── config/test_settings.py
    ├── database/test_session.py
    ├── parameters/test_parameters_domain.py
    ├── parameters/test_parameters_service.py
    ├── utils/test_request_id.py
    ├── utils/test_pagination.py
    └── utils/test_manila_time.py
```

---

### Task 1: Repo scaffold and health endpoint

**Files:**
- Create: `pikol-backend/pyproject.toml`, `.gitignore`, `.python-version`
- Create: `pikol-backend/src/main.py`, `src/utils/api.py`
- Create: `pikol-backend/src/health/controllers/health.py`
- Test: `pikol-backend/tests/conftest.py`, `tests/health/test_health.py`

**Interfaces:**
- Consumes: nothing — this is the first task.
- Produces: `src.main.create_app() -> FastAPI`, `src.main.app`,
  `src.utils.api.API_PREFIX: str` (value `"/api/v1"`).

- [ ] **Step 1: Create the repo and project files**

```bash
cd /Users/dalenacario/Desktop/projects/pikol-parent
mkdir -p pikol-backend/src/{config,database,health/controllers,parameters,utils}
mkdir -p pikol-backend/tests/{health,config,database,parameters,utils}
cd pikol-backend
git init -b main
```

Create `pyproject.toml`:

```toml
[project]
name = "pikol-backend"
version = "0.1.0"
description = "Pikol — pickleball court booking API"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "pydantic>=2.9",
    "pydantic-settings>=2.6",
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30",
    "alembic>=1.14",
]

[dependency-groups]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "httpx>=0.28",
    "ruff>=0.8",
    "mypy>=1.13",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC"]

[tool.ruff.lint.flake8-bugbear]
# `session: AsyncSession = Depends(get_session)` is THE FastAPI idiom, and
# B008 ("no function call in an argument default") flags every one of them.
# Without this the lint gate fails on the first router that takes a
# dependency — which is all of them, from Phase 2 onward.
extend-immutable-calls = [
    "fastapi.Depends",
    "fastapi.Query",
    "fastapi.Path",
    "fastapi.Body",
    "fastapi.Header",
    "fastapi.Security",
]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
```

Create `.gitignore`:

```
__pycache__/
*.py[cod]
.venv/
.env
.env.*
!.env.example
.pytest_cache/
.mypy_cache/
.ruff_cache/
tasks/
.DS_Store
```

Pin the interpreter, then install:

```bash
uv python pin 3.12
uv sync
```

**Pin before syncing.** `requires-python = ">=3.12"` is a floor, not a pin —
uv resolves the newest Python that satisfies it (3.14 at time of writing),
while mypy is configured `python_version = "3.12"` and ruff
`target-version = "py312"`. Left unpinned the type checker reasons about a
different Python than the one running the tests, and CI — which calls
`setup-uv` with no version — drifts on its own schedule as new releases
land. `.python-version` is committed, so local and CI agree and stay that
way.

- [ ] **Step 2: Write the failing test**

`tests/conftest.py` (Task 3 replaces this file with the database-aware
version — imports go in the header block, never appended at the bottom, or
ruff's `E402` fails the lint gate):

```python
from collections.abc import AsyncGenerator

import pytest
from httpx import ASGITransport, AsyncClient

from src.main import create_app


@pytest.fixture
async def client() -> AsyncGenerator[AsyncClient, None]:
    transport = ASGITransport(app=create_app())
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

`tests/health/test_health.py`:

```python
from httpx import AsyncClient


async def test_health_reports_ok(client: AsyncClient) -> None:
    response = await client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


async def test_health_is_not_under_the_api_prefix(client: AsyncClient) -> None:
    """Health is infrastructure, not API — load balancers should not need to
    know the API version to probe liveness."""
    response = await client.get("/api/v1/health")

    assert response.status_code == 404
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `uv run pytest tests/health/ -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.main'`

- [ ] **Step 4: Write the minimal implementation**

`src/utils/api.py`:

```python
API_VERSION = "v1"
API_PREFIX = f"/api/{API_VERSION}"
```

`src/health/controllers/health.py`:

```python
from fastapi import APIRouter

router = APIRouter(tags=["health"])


@router.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

`src/main.py`:

```python
from fastapi import FastAPI

from src.health.controllers.health import router as health_router


def create_app() -> FastAPI:
    app = FastAPI(title="Pikol API", version="0.1.0")
    app.include_router(health_router)
    return app


app = create_app()
```

Add empty `__init__.py` to every package directory under `src/` and `tests/`.
The `__pycache__` exclusion matters: Step 3 already ran pytest, so
`tests/__pycache__/` exists by now and a bare `-type d` plants a stray
`__init__.py` inside it.

```bash
find src tests -type d -not -path "*__pycache__*" -exec touch {}/__init__.py \;
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `uv run pytest tests/health/ -v`
Expected: PASS — 2 passed

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "Add FastAPI scaffold with health endpoint

Health sits at /health rather than under /api/v1 because it is
infrastructure: a load balancer probing liveness should not have to track
the API version. A test pins that so the route cannot drift under the
prefix later."
```

---

### Task 2: Typed configuration that fails at boot

**Files:**
- Create: `pikol-backend/src/config/settings.py`, `.env.example`
- Test: `pikol-backend/tests/config/test_settings.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `src.config.settings.Settings` (Pydantic BaseSettings),
  `src.config.settings.get_settings() -> Settings` (cached).
  Fields: `app_name: str`, `debug: bool`, `database_url: str`,
  `cors_allowed_origins: list[str]`.

- [ ] **Step 1: Write the failing test**

`tests/config/test_settings.py`:

```python
import pytest
from pydantic import ValidationError

from src.config.settings import Settings


def test_settings_load_from_environment(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@localhost:5432/pikol")
    monkeypatch.setenv("CORS_ALLOWED_ORIGINS", '["http://localhost:3000"]')

    settings = Settings(_env_file=None)

    assert settings.database_url == "postgresql+asyncpg://u:p@localhost:5432/pikol"
    assert settings.cors_allowed_origins == ["http://localhost:3000"]
    assert settings.debug is False


def test_missing_database_url_fails_loudly(monkeypatch: pytest.MonkeyPatch) -> None:
    """A missing connection string must stop the process at boot, not surface
    as a confusing connection error on the first request."""
    monkeypatch.delenv("DATABASE_URL", raising=False)

    with pytest.raises(ValidationError) as exc:
        Settings(_env_file=None)

    assert "database_url" in str(exc.value)


def test_cors_origins_reject_trailing_slashes(monkeypatch: pytest.MonkeyPatch) -> None:
    """A trailing slash silently breaks CORS matching in production — the
    exact trap asima hit. Reject it at boot instead."""
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@localhost:5432/pikol")
    monkeypatch.setenv("CORS_ALLOWED_ORIGINS", '["http://localhost:3000/"]')

    with pytest.raises(ValidationError) as exc:
        Settings(_env_file=None)

    assert "trailing slash" in str(exc.value)
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `uv run pytest tests/config/ -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.config.settings'`

- [ ] **Step 3: Write the implementation**

`src/config/settings.py`:

```python
from functools import lru_cache

from pydantic import field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Environment configuration, read once at boot.

    Distinct from `parameters` (src/parameters/), which holds *business*
    rules — prices, windows, caps — in Postgres so they are tunable without
    a deploy. Anything here requires a restart to change.
    """

    # `extra="ignore"` because the process environment always carries
    # unrelated variables (PATH, HOME). Unknown keys are not an error.
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_name: str = "pikol"
    debug: bool = False
    database_url: str
    cors_allowed_origins: list[str] = []

    @field_validator("cors_allowed_origins")
    @classmethod
    def reject_trailing_slash(cls, origins: list[str]) -> list[str]:
        for origin in origins:
            if origin.endswith("/"):
                raise ValueError(
                    f"CORS origin {origin!r} has a trailing slash; browsers send "
                    f"the origin without one, so it would never match"
                )
        return origins


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Create `.env.example`:

```bash
# --- App ---
APP_NAME=pikol
DEBUG=true

# --- Database ---
# LOCAL: the docker-compose Postgres.
# SUPABASE: use the SESSION pooler on port 5432, never the transaction
# pooler on 6543 — asyncpg uses prepared statements and 6543 does not
# support them, which fails intermittently under load only in production.
DATABASE_URL=postgresql+asyncpg://pikol:pikol@localhost:5432/pikol

# --- CORS ---
# Exact origins, JSON array, NO trailing slash (boot fails if you add one).
CORS_ALLOWED_ORIGINS=["http://localhost:3000"]
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `uv run pytest tests/config/ -v`
Expected: PASS — 3 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add typed settings that fail at boot

Rejects CORS origins with a trailing slash. Browsers send the origin
without one, so a trailing slash produces a silent production-only CORS
failure — the trap asima hit. Failing at boot beats debugging it live.

Documents in .env.example why Supabase's session pooler (5432) is required
over the transaction pooler (6543): asyncpg uses prepared statements, which
6543 does not support."
```

---

### Task 3: Async session, readiness probe, and the rollback fixture

**Files:**
- Create: `pikol-backend/docker-compose.yml`
- Create: `pikol-backend/src/database/base.py`, `src/database/session.py`
- Test: `pikol-backend/tests/database/test_session.py`

**Interfaces:**
- Consumes: `src.config.settings.get_settings`.
- Produces: `src.database.base.Base` (DeclarativeBase),
  `src.database.session.engine`, `src.database.session.SessionLocal`,
  `src.database.session.get_session() -> AsyncGenerator[AsyncSession, None]`.

- [ ] **Step 1: Start local Postgres**

Create `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17
    container_name: pikol-postgres
    environment:
      POSTGRES_USER: pikol
      POSTGRES_PASSWORD: pikol
      POSTGRES_DB: pikol
    ports:
      - "5432:5432"
    volumes:
      - pikol-pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pikol"]
      interval: 5s
      retries: 10

volumes:
  pikol-pgdata:
```

Run:

```bash
cp .env.example .env
docker compose up -d
docker compose ps          # expect postgres healthy
```

- [ ] **Step 2: Write the failing test**

`tests/database/test_session.py`:

```python
from sqlalchemy import text

from src.database.session import get_session


async def test_session_connects_to_postgres() -> None:
    agen = get_session()
    session = await anext(agen)
    try:
        result = await session.execute(text("SELECT 1"))
        assert result.scalar_one() == 1
    finally:
        await agen.aclose()
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `uv run pytest tests/database/ -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.database.session'`

- [ ] **Step 4: Write the implementation**

`src/database/base.py`:

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """Declarative base for every Pikol model."""
```

`src/database/session.py`:

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from src.config.settings import get_settings

settings = get_settings()

# pool_pre_ping guards against connections killed by an idle pooler — which
# Supabase does. Modest pool size: Supabase's session pooler has a limited
# connection budget, and this process is long-lived (see spec §9.1).
engine = create_async_engine(
    settings.database_url,
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=5,
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI dependency yielding one session per request."""
    async with SessionLocal() as session:
        yield session
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `uv run pytest tests/database/ -v`
Expected: PASS — 1 passed

- [ ] **Step 6: Write the failing readiness test**

Add to `tests/health/test_health.py` — the test goes at the end, but any new
import belongs in the header block. Imports appended below existing code trip
ruff's `E402`, and `ruff check .` covers `tests/`:

```python
async def test_readiness_touches_the_database(client: AsyncClient) -> None:
    """Liveness and readiness are different questions, and here the
    difference is load-bearing.

    Spec §9.3 says the UptimeRobot ping stops Supabase pausing after 7 idle
    days. Supabase pauses on *database* inactivity, so a probe that returns a
    dict literal issues no query and does not keep it alive: Render stays
    awake, Postgres pauses anyway, and the first real request after a quiet
    week fails. The keep-alive must point HERE."""
    response = await client.get("/health/ready")

    assert response.status_code == 200
    assert response.json() == {"status": "ready", "database": "ok"}
```

- [ ] **Step 7: Run it to verify it fails**

Run: `uv run pytest tests/health/ -v`
Expected: FAIL — 404, the route does not exist

- [ ] **Step 8: Implement readiness**

Replace `src/health/controllers/health.py`:

```python
from fastapi import APIRouter, Depends
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from src.database.session import get_session

router = APIRouter(tags=["health"])


@router.get("/health")
async def health() -> dict[str, str]:
    """Liveness. Deliberately touches nothing.

    A database outage must not make the orchestrator kill a container that is
    otherwise healthy and serving cached reads.
    """
    return {"status": "ok"}


@router.get("/health/ready")
async def ready(session: AsyncSession = Depends(get_session)) -> dict[str, str]:
    """Readiness. Issues a real query.

    This is the endpoint the keep-alive monitor must hit: it is what keeps
    Supabase from pausing the project after 7 idle days (spec §9.3).
    """
    await session.execute(text("SELECT 1"))
    return {"status": "ready", "database": "ok"}
```

- [ ] **Step 9: Replace `tests/conftest.py` with the database-aware version**

**Replace the whole file** — do not append. Appending imports to the bottom
trips ruff's `E402` and `I001`, and `ruff check .` covers `tests/`, so the
lint gate would fail.

Two things here are load-bearing and neither is obvious:

```python
from collections.abc import AsyncGenerator

import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from src.database.session import engine, get_session
from src.main import create_app


@pytest.fixture(autouse=True)
async def _dispose_engine_between_tests() -> AsyncGenerator[None, None]:
    """pytest-asyncio gives every test its own event loop.

    The module-level engine pools connections, so one checked out under test
    A's loop is handed back and reused under test B's — and asyncpg raises
    `RuntimeError: Future attached to a different loop`, or `Event loop is
    closed`. It looks like flakiness and it is not: it is deterministic once
    two tests touch the database. Disposing after each test means no
    connection outlives the loop that created it.
    """
    yield
    await engine.dispose()


@pytest.fixture
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    """A session inside a transaction that is always rolled back.

    The test sees its own writes; the database never does.
    """
    async with engine.connect() as connection:
        transaction = await connection.begin()
        session = AsyncSession(bind=connection, expire_on_commit=False)
        try:
            yield session
        finally:
            await session.close()
            await transaction.rollback()


@pytest.fixture
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """The app, with its session dependency pointed at the rolling-back one.

    Without this override the app opens its OWN session on the real engine
    and commits for real — so the rollback fixture would guarantee nothing
    about anything a request writes, which is most of what Phase 2 onward
    tests. The fixture would be decorative.
    """
    app = create_app()
    app.dependency_overrides[get_session] = lambda: db_session
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

- [ ] **Step 10: Run the full suite**

Run: `uv run ruff format . && uv run pytest -v`
Expected: PASS — 7 passed

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "Add async session, readiness probe, and rollback test fixture

pool_pre_ping is on because Supabase's pooler drops idle connections, and
the pool is deliberately small: the session pooler has a limited connection
budget and this is a long-lived process, not a serverless function.

Splits liveness from readiness because the difference is load-bearing here.
The spec relies on a keep-alive ping to stop Supabase pausing after 7 idle
days, but Supabase pauses on database inactivity — a probe returning a dict
literal issues no query and would not have kept it alive. /health/ready runs
SELECT 1 and is what the monitor must target.

The db_session fixture rolls back every test transaction. Added now rather
than in Phase 2 because retrofitting it means rewriting every test that
came before."
```

---

### Task 4: Alembic and the seeded `parameters` table

**Files:**
- Create: `pikol-backend/alembic.ini`
- Create: `pikol-backend/src/database/migrations/env.py`, `script.py.mako`
- Create: `pikol-backend/src/database/migrations/versions/0001_parameters.py`
- Create: `pikol-backend/src/parameters/persistence/models.py`

**Schema source:** [`../adr/pikol.dbml`](../adr/pikol.dbml) — the `parameters`
table and its ten seeded keys (Appendix B). Every later phase's migration is
written from that file, and any constraint it marks ⚠️ must be copied from its
Appendix A rather than inferred from the diagram.

**Interfaces:**
- Consumes: `src.database.base.Base`, `src.config.settings.get_settings`.
- Produces: `src.parameters.persistence.models.ParameterModel` with columns
  `key: str` (PK), `value: str`, `value_type: str`, `description: str`,
  `updated_at: datetime`, `updated_by_user_id: int | None`.

- [ ] **Step 1: Write the model**

`src/parameters/persistence/models.py`:

```python
from datetime import datetime

from sqlalchemy import DateTime, String, Text, func
from sqlalchemy.dialects.postgresql import ENUM
from sqlalchemy.orm import Mapped, mapped_column

from src.database.base import Base


class ParameterModel(Base):
    """One tunable business rule (spec §4.5).

    Values are stored as text and coerced by the Pydantic model in
    src/parameters/domain/. `value_type` is not used for coercion — it exists
    so the admin UI can render the right input control.
    """

    __tablename__ = "parameters"

    key: Mapped[str] = mapped_column(String(64), primary_key=True)
    value: Mapped[str] = mapped_column(Text, nullable=False)
    # postgresql.ENUM, not the generic sqlalchemy.Enum: `create_type` is a
    # PG-dialect parameter, and the generic type SWALLOWS it silently — the
    # object ends up with no create_type at all and keeps default
    # create-with-table behaviour. Any metadata.create_all() or autogenerate
    # then emits CREATE TYPE a second time and fails with "type already
    # exists", which is the exact failure this is meant to prevent.
    value_type: Mapped[str] = mapped_column(
        ENUM(
            "int",
            "decimal",
            "bool",
            "string",
            name="parameter_value_type",
            create_type=False,  # the migration owns creation
        ),
        nullable=False,
    )
    description: Mapped[str] = mapped_column(Text, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
    # No FK yet — the users table arrives in Phase 2, which adds the
    # constraint in its own migration.
    updated_by_user_id: Mapped[int | None] = mapped_column(nullable=True)
```

- [ ] **Step 2: Initialise Alembic**

```bash
uv run alembic init -t async src/database/migrations
```

Edit `alembic.ini` **in place**. Change only these two keys inside the
existing `[alembic]` section, and delete the `sqlalchemy.url` line —
`env.py` supplies the URL:

```ini
script_location = src/database/migrations
prepend_sys_path = .
```

> **Do not replace the file with the block above.** `alembic init` also
> generates `[loggers]`, `[handlers]`, and `[formatters]` sections, and
> `env.py` calls `fileConfig()` unconditionally. Dropping them makes every
> `alembic` command — including CI's migrate step — die with
> `KeyError: 'formatters'`.

Replace `src/database/migrations/env.py`:

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

# Import every module defining a model so Base.metadata is complete.
# Each new feature adds its import here. It sits ABOVE the src.* from-imports
# because isort orders a plain `import x` before `from x import y` within the
# same section — placed below them, `ruff check .` fails on I001, and CI runs
# that gate.
import src.parameters.persistence.models  # noqa: F401
from src.config.settings import get_settings
from src.database.base import Base

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

# The URL is passed straight to the engine, NEVER through
# config.set_main_option(). That writes into ConfigParser, which performs
# `%` interpolation — so a Supabase password containing a percent-encoded
# character (`%40` for @, `%2F` for /) raises
# `ValueError: invalid interpolation syntax`. Local creds like `pikol:pikol`
# never trigger it, so this would pass in dev and in CI and then break the
# first production migration — the exact laptop-against-.env.supabase.prod
# flow spec §9.4 prescribes.
DATABASE_URL = get_settings().database_url


def run_migrations_offline() -> None:
    context.configure(
        url=DATABASE_URL,
        target_metadata=target_metadata,
        literal_binds=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
        url=DATABASE_URL,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_async_migrations())
```

- [ ] **Step 3: Write the migration with seeded defaults**

`src/database/migrations/versions/0001_parameters.py`:

```python
"""Create parameters table and seed defaults

Revision ID: 0001_parameters
Revises:
"""

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision = "0001_parameters"
down_revision = None
branch_labels = None
depends_on = None

SEED = [
    ("default_slot_price_centavos", "50000", "int",
     "Fallback slot price when no price rule matches (PHP 500.00)."),
    ("convenience_fee_percent", "3.00", "decimal",
     "Flat fee added to online bookings. Walk-ins pay none."),
    ("hold_duration_minutes", "15", "int",
     "How long a PENDING hold blocks a slot before it may be expired."),
    ("booking_horizon_days", "30", "int",
     "How far ahead a player may book."),
    ("max_active_holds_per_player", "3", "int",
     "Cap on unexpired holds per player. Denial-of-inventory guard."),
    ("staff_backdate_days", "7", "int",
     "How far back staff may record a walk-in booking."),
    ("cancellation_cutoff_minutes", "0", "int",
     "How close to start time a player may still cancel."),
    ("min_lead_minutes", "20", "int",
     "Earliest a player may book before a slot starts. Must be >= "
     "hold_duration_minutes, or a hold can outlive the slot it holds."),
    ("invitation_expiry_days", "7", "int",
     "Lifetime of a staff invitation token."),
    ("webhook_payload_retention_days", "90", "int",
     "Age at which raw provider payloads are purged (RA 10173)."),
]


# create_type=False so the type is made exactly once, by the explicit
# .create() below. Left at the default, SQLAlchemy also tries to emit it
# while building the table and the migration fails with "type already
# exists". Every enum in this project follows this shape.
parameter_value_type = postgresql.ENUM(
    "int",
    "decimal",
    "bool",
    "string",
    name="parameter_value_type",
    create_type=False,
)


def upgrade() -> None:
    parameter_value_type.create(op.get_bind(), checkfirst=True)

    parameters = op.create_table(
        "parameters",
        sa.Column("key", sa.String(64), primary_key=True),
        sa.Column("value", sa.Text(), nullable=False),
        sa.Column("value_type", parameter_value_type, nullable=False),
        sa.Column("description", sa.Text(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True),
                  server_default=sa.func.now(), nullable=False),
        sa.Column("updated_by_user_id", sa.Integer(), nullable=True),
    )
    op.bulk_insert(
        parameters,
        [
            {"key": k, "value": v, "value_type": t, "description": d}
            for k, v, t, d in SEED
        ],
    )


def downgrade() -> None:
    op.drop_table("parameters")
    parameter_value_type.drop(op.get_bind(), checkfirst=True)
```

- [ ] **Step 4: Run the migration**

```bash
uv run alembic upgrade head
```

Expected: `Running upgrade  -> 0001_parameters`

Verify the seed landed:

```bash
docker compose exec postgres psql -U pikol -d pikol -c "SELECT key, value FROM parameters ORDER BY key;"
```

Expected: 10 rows, `min_lead_minutes = 20`, `hold_duration_minutes = 15`.

- [ ] **Step 5: Verify the migration is reversible**

```bash
uv run alembic downgrade base && uv run alembic upgrade head
```

Expected: both succeed. A migration that cannot roll back is a migration you
cannot deploy safely.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "Add Alembic and seed the parameters table

Seeds all ten business rules from spec section 4.5 in the migration rather
than a separate seed script, so a fresh database is immediately usable and
the defaults are version-controlled alongside the schema.

Note min_lead_minutes (20) exceeds hold_duration_minutes (15) by design:
were it lower, a hold could outlive the slot it holds and the webhook would
confirm a session that had already started."
```

---

### Task 5: Typed parameter accessor with cross-field validation

**Files:**
- Create: `pikol-backend/src/parameters/domain/parameters.py`
- Create: `pikol-backend/src/parameters/persistence/repository.py`
- Create: `pikol-backend/src/parameters/service.py`
- Test: `pikol-backend/tests/parameters/test_parameters_domain.py`
- Test: `pikol-backend/tests/parameters/test_parameters_service.py`

**Interfaces:**
- Consumes: `ParameterModel`, `get_session`.
- Produces:
  - `src.parameters.domain.parameters.Parameters` — Pydantic model with
    fields `default_slot_price_centavos: int`, `convenience_fee_percent:
    Decimal`, `hold_duration_minutes: int`, `booking_horizon_days: int`,
    `max_active_holds_per_player: int`, `staff_backdate_days: int`,
    `cancellation_cutoff_minutes: int`, `min_lead_minutes: int`,
    `invitation_expiry_days: int`, `webhook_payload_retention_days: int`.
  - `src.parameters.persistence.repository.ParameterRepository.load_all(session) -> dict[str, str]`
  - `src.parameters.service.ParameterService.get(session) -> Parameters`

- [ ] **Step 1: Write the failing domain test**

`tests/parameters/test_parameters_domain.py`:

```python
from decimal import Decimal

import pytest
from pydantic import ValidationError

from src.parameters.domain.parameters import Parameters

VALID: dict[str, str] = {
    "default_slot_price_centavos": "50000",
    "convenience_fee_percent": "3.00",
    "hold_duration_minutes": "15",
    "booking_horizon_days": "30",
    "max_active_holds_per_player": "3",
    "staff_backdate_days": "7",
    "cancellation_cutoff_minutes": "0",
    "min_lead_minutes": "20",
    "invitation_expiry_days": "7",
    "webhook_payload_retention_days": "90",
}


def test_coerces_stored_strings_to_types() -> None:
    params = Parameters.model_validate(VALID)

    assert params.default_slot_price_centavos == 50000
    assert params.convenience_fee_percent == Decimal("3.00")
    assert params.hold_duration_minutes == 15


def test_non_numeric_value_fails_loudly() -> None:
    """A typo in a money-moving parameter must raise, never silently become
    zero and charge every player nothing."""
    broken = VALID | {"convenience_fee_percent": "abc"}

    with pytest.raises(ValidationError) as exc:
        Parameters.model_validate(broken)

    assert "convenience_fee_percent" in str(exc.value)


def test_min_lead_below_hold_duration_is_rejected() -> None:
    """Spec section 4.5: were min_lead_minutes lower than hold_duration,
    a player could hold a slot starting in 5 minutes, pay at minute 12, and
    have the webhook confirm a session that began 7 minutes earlier."""
    broken = VALID | {"min_lead_minutes": "10", "hold_duration_minutes": "15"}

    with pytest.raises(ValidationError) as exc:
        Parameters.model_validate(broken)

    assert "min_lead_minutes" in str(exc.value)


def test_missing_parameter_is_rejected() -> None:
    incomplete = {k: v for k, v in VALID.items() if k != "hold_duration_minutes"}

    with pytest.raises(ValidationError):
        Parameters.model_validate(incomplete)


def test_negative_price_is_rejected() -> None:
    with pytest.raises(ValidationError):
        Parameters.model_validate(VALID | {"default_slot_price_centavos": "-1"})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `uv run pytest tests/parameters/test_parameters_domain.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.parameters.domain.parameters'`

- [ ] **Step 3: Write the domain model**

`src/parameters/domain/parameters.py`:

```python
from decimal import Decimal
from typing import Self

from pydantic import BaseModel, Field, model_validator


class Parameters(BaseModel):
    """Every tunable business rule, typed (spec §4.5).

    Values arrive from the database as text; Pydantic does the coercion, so a
    typo raises here rather than silently becoming zero somewhere that moves
    money.
    """

    # "ignore", NOT "forbid". The Global Constraint mandating extra="forbid"
    # is about REQUEST schemas — untrusted input, where an unexpected field
    # is an attack surface. Here the input is our own table, and forbidding
    # extras inverts the compatibility it is meant to buy: the normal
    # migrate-then-deploy order means a new parameter row exists while the
    # previous release is still serving, and "forbid" would make every
    # booking 500 until the deploy lands. Missing keys still raise, because
    # every field below is required — which is the check that actually
    # matters.
    model_config = {"extra": "ignore"}

    default_slot_price_centavos: int = Field(ge=0)
    convenience_fee_percent: Decimal = Field(ge=0, le=100)
    hold_duration_minutes: int = Field(gt=0)
    booking_horizon_days: int = Field(gt=0)
    max_active_holds_per_player: int = Field(gt=0)
    staff_backdate_days: int = Field(ge=0)
    cancellation_cutoff_minutes: int = Field(ge=0)
    min_lead_minutes: int = Field(ge=0)
    invitation_expiry_days: int = Field(gt=0)
    webhook_payload_retention_days: int = Field(gt=0)

    @model_validator(mode="after")
    def lead_time_must_cover_the_hold(self) -> Self:
        if self.min_lead_minutes < self.hold_duration_minutes:
            raise ValueError(
                f"min_lead_minutes ({self.min_lead_minutes}) must be >= "
                f"hold_duration_minutes ({self.hold_duration_minutes}); "
                f"otherwise a hold can outlive the slot it holds"
            )
        return self
```

- [ ] **Step 4: Run the domain test to verify it passes**

Run: `uv run pytest tests/parameters/test_parameters_domain.py -v`
Expected: PASS — 5 passed

- [ ] **Step 5: Write the failing service test**

`tests/parameters/test_parameters_service.py`:

```python
from decimal import Decimal

from sqlalchemy.ext.asyncio import AsyncSession

from src.database.session import SessionLocal
from src.parameters.persistence.repository import ParameterRepository
from src.parameters.service import ParameterService


async def test_loads_seeded_parameters_from_the_database() -> None:
    service = ParameterService()

    async with SessionLocal() as session:
        params = await service.get(session)

    assert params.default_slot_price_centavos == 50000
    assert params.convenience_fee_percent == Decimal("3.00")
    assert params.min_lead_minutes >= params.hold_duration_minutes


class CountingRepository(ParameterRepository):
    """Injected rather than monkeypatched — the service takes its repository
    as a constructor argument, so the test needs no patching and no
    type: ignore."""

    def __init__(self) -> None:
        self.calls = 0

    async def load_all(self, session: AsyncSession) -> dict[str, str]:
        self.calls += 1
        return await super().load_all(session)


async def test_second_call_within_ttl_does_not_query_again() -> None:
    """The accessor is read on every booking; hitting Postgres each time is
    waste. The TTL is short so an admin edit still lands quickly."""
    repository = CountingRepository()
    service = ParameterService(repository, ttl_seconds=60)

    async with SessionLocal() as session:
        await service.get(session)
        await service.get(session)

    assert repository.calls == 1


async def test_expired_cache_reloads() -> None:
    repository = CountingRepository()
    service = ParameterService(repository, ttl_seconds=0)

    async with SessionLocal() as session:
        first = await service.get(session)
        second = await service.get(session)

    assert repository.calls == 2
    assert first == second
```

- [ ] **Step 6: Run the test to verify it fails**

Run: `uv run pytest tests/parameters/test_parameters_service.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.parameters.service'`

- [ ] **Step 7: Write the repository and service**

`src/parameters/persistence/repository.py`:

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.parameters.persistence.models import ParameterModel


class ParameterRepository:
    async def load_all(self, session: AsyncSession) -> dict[str, str]:
        result = await session.execute(select(ParameterModel))
        return {row.key: row.value for row in result.scalars().all()}
```

`src/parameters/service.py`:

```python
import time

from sqlalchemy.ext.asyncio import AsyncSession

from src.parameters.domain.parameters import Parameters
from src.parameters.persistence.repository import ParameterRepository

DEFAULT_TTL_SECONDS = 30


class ParameterService:
    """TTL-cached access to the parameters table.

    Read on nearly every booking, so it is cached; the TTL is short so an
    admin changing the fee sees it take effect promptly without a deploy.
    """

    def __init__(
        self,
        repository: ParameterRepository | None = None,
        ttl_seconds: int = DEFAULT_TTL_SECONDS,
    ) -> None:
        self._repository = repository or ParameterRepository()
        self._ttl = ttl_seconds
        self._cached: Parameters | None = None
        self._loaded_at = 0.0

    async def get(self, session: AsyncSession) -> Parameters:
        now = time.monotonic()
        if self._cached is not None and (now - self._loaded_at) < self._ttl:
            return self._cached

        self._cached = Parameters.model_validate(await self._repository.load_all(session))
        self._loaded_at = now
        return self._cached


parameter_service = ParameterService()
```

- [ ] **Step 8: Run the full suite to verify it passes**

Run: `uv run ruff format . && uv run pytest -v`
Expected: PASS — 15 passed

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "Add typed, TTL-cached parameter accessor

Values are stored as text and coerced by Pydantic, so a typo in a
money-moving parameter raises at load instead of silently becoming zero.

The cross-field validator enforces min_lead_minutes >= hold_duration_minutes
in code rather than leaving it as prose in the spec: below that threshold a
hold can outlive the slot it holds, and the webhook confirms a session that
already started."
```

---

### Task 6: Shared utilities — request ID, logging, errors, pagination, Manila time

These are the cross-cutting pieces every later feature imports. They are one
task because they share a home (`src/utils/`) and one reviewer gate; splitting
them would mean five commits that individually do nothing.

**Files:**
- Create: `src/utils/request_id.py`, `src/utils/log.py`, `src/utils/errors.py`,
  `src/utils/pagination.py`, `src/utils/time.py`
- Modify: `src/main.py`
- Test: `tests/utils/test_request_id.py`, `tests/utils/test_pagination.py`,
  `tests/utils/test_manila_time.py`

**Interfaces:**
- Consumes: `src.config.settings.Settings`, `get_settings`.
- Produces:
  - `src.utils.request_id.RequestIDMiddleware`, `request_id_var: ContextVar[str]`
  - `src.utils.log.configure_logging(debug: bool) -> None`
  - `src.utils.errors.ErrorResponse` (`detail: str`, `request_id: str`),
    `register_exception_handlers(app: FastAPI) -> None`
  - `src.utils.pagination.Page[T]` with `data, total, page, limit, has_more`
    and `Page.of(data, total, page, limit) -> Page[T]`
  - `src.utils.time.MANILA`, `manila_day_bounds(day: date) -> tuple[datetime, datetime]`
  - `src.main.create_app(settings: Settings | None = None) -> FastAPI`

- [ ] **Step 1: Write the failing tests**

`tests/utils/test_request_id.py`:

```python
from httpx import ASGITransport, AsyncClient

from src.config.settings import Settings
from src.main import create_app


async def test_response_carries_a_request_id(client: AsyncClient) -> None:
    response = await client.get("/health")

    assert response.headers.get("x-request-id")


async def test_incoming_request_id_is_echoed(client: AsyncClient) -> None:
    """Propagating the caller's ID is what makes a frontend log line and a
    backend log line joinable."""
    response = await client.get("/health", headers={"X-Request-ID": "abc-123"})

    assert response.headers["x-request-id"] == "abc-123"


async def test_ids_differ_between_requests(client: AsyncClient) -> None:
    first = await client.get("/health")
    second = await client.get("/health")

    assert first.headers["x-request-id"] != second.headers["x-request-id"]


async def test_docs_are_hidden_when_not_debugging() -> None:
    """asima 404s /docs in production. The OpenAPI schema lists every admin
    route and its request shape — free reconnaissance."""
    production = Settings(
        _env_file=None,
        database_url="postgresql+asyncpg://u:p@localhost:5432/pikol",
        debug=False,
    )
    transport = ASGITransport(app=create_app(production))

    async with AsyncClient(transport=transport, base_url="http://test") as c:
        assert (await c.get("/docs")).status_code == 404
        assert (await c.get("/openapi.json")).status_code == 404
```

`tests/utils/test_manila_time.py`:

```python
from datetime import UTC, date, datetime, timedelta

from src.utils.time import MANILA, manila_day_bounds


def test_manila_day_begins_at_1600_utc_the_previous_day() -> None:
    start, end = manila_day_bounds(date(2026, 9, 15))

    assert start == datetime(2026, 9, 14, 16, 0, tzinfo=UTC)
    assert end == datetime(2026, 9, 15, 16, 0, tzinfo=UTC)


def test_bounds_span_exactly_one_day() -> None:
    start, end = manila_day_bounds(date(2026, 9, 15))

    assert end - start == timedelta(days=1)


def test_the_midnight_slot_belongs_to_its_own_manila_day() -> None:
    """This is the test that matters. The 12am-1am slot has its own price
    rule, so misplacing it onto the previous day changes what a player is
    charged. Slicing on `starts_at::date` in UTC puts it on 14 September."""
    midnight_slot = datetime(2026, 9, 15, 0, 0, tzinfo=MANILA).astimezone(UTC)
    start, end = manila_day_bounds(date(2026, 9, 15))

    assert start <= midnight_slot < end


def test_the_last_slot_of_the_day_is_inside_the_bounds() -> None:
    last_slot = datetime(2026, 9, 15, 23, 0, tzinfo=MANILA).astimezone(UTC)
    start, end = manila_day_bounds(date(2026, 9, 15))

    assert start <= last_slot < end
```

`tests/utils/test_pagination.py`:

```python
from src.utils.pagination import Page


def test_has_more_is_true_when_further_pages_exist() -> None:
    page = Page.of(data=[1, 2, 3], total=10, page=1, limit=3)

    assert page.has_more is True


def test_has_more_is_false_on_the_last_partial_page() -> None:
    page = Page.of(data=[10], total=10, page=4, limit=3)

    assert page.has_more is False


def test_has_more_is_false_when_the_page_exactly_ends_the_set() -> None:
    """The off-by-one that makes a UI render an empty final page."""
    page = Page.of(data=[1, 2, 3], total=3, page=1, limit=3)

    assert page.has_more is False


def test_empty_result_set() -> None:
    page = Page.of(data=[], total=0, page=1, limit=20)

    assert page.has_more is False
    assert page.total == 0
```

- [ ] **Step 2: Run them to verify they fail**

Run: `uv run pytest tests/utils/ -v`
Expected: FAIL — `ModuleNotFoundError` for `src.utils.time` and
`src.utils.pagination`, and `KeyError: 'x-request-id'`

- [ ] **Step 3: Implement the request ID and logging**

`src/utils/request_id.py`:

```python
import logging
import time
from collections.abc import Awaitable, Callable
from contextvars import ContextVar
from uuid import uuid4

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

HEADER = "X-Request-ID"

# Read by the log formatter, so any log line anywhere in the request carries
# the ID without every call site having to thread it through.
request_id_var: ContextVar[str] = ContextVar("request_id", default="")

logger = logging.getLogger("pikol.request")


class RequestIDMiddleware(BaseHTTPMiddleware):
    """Tag every request, echo the caller's ID if it sent one (spec §3.5)."""

    async def dispatch(
        self, request: Request, call_next: Callable[[Request], Awaitable[Response]]
    ) -> Response:
        request_id = request.headers.get(HEADER) or str(uuid4())
        token = request_id_var.set(request_id)
        request.state.request_id = request_id
        started = time.perf_counter()
        try:
            response = await call_next(request)
            elapsed_ms = round((time.perf_counter() - started) * 1000, 1)
            logger.info(
                "%s %s %s %sms",
                request.method,
                request.url.path,
                response.status_code,
                elapsed_ms,
            )
            response.headers[HEADER] = request_id
            return response
        finally:
            request_id_var.reset(token)
```

`src/utils/log.py`:

```python
import json
import logging
import sys
from datetime import UTC, datetime
from typing import Any

from src.utils.request_id import request_id_var


class JsonFormatter(logging.Formatter):
    """JSON lines, because Render's log viewer is grep and nothing else.

    Every record carries the current request_id, so one failed booking can be
    traced across every line it produced.
    """

    def format(self, record: logging.LogRecord) -> str:
        payload: dict[str, Any] = {
            # Without this, a line grepped out of Render's viewer and pasted
            # into a ticket loses all timing — you cannot tell whether the 409
            # preceded the webhook, or line anything up against PayMongo's
            # timestamps during a payment dispute.
            "timestamp": datetime.fromtimestamp(record.created, UTC).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "request_id": request_id_var.get(),
        }
        if record.exc_info:
            payload["exception"] = self.formatException(record.exc_info)
        return json.dumps(payload)


def configure_logging(debug: bool) -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(logging.DEBUG if debug else logging.INFO)
```

> **Never log a provider payload.** `webhook_events.payload` and
> `payments.raw_payload` hold payer name, email, and mobile. Log the
> `provider_event_id` and the outcome; the payload stays in the table and is
> purged on schedule (spec §7.5).

- [ ] **Step 4: Implement the error envelope**

`src/utils/errors.py`:

```python
import logging

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from sqlalchemy.exc import IntegrityError

logger = logging.getLogger("pikol.error")

SLOT_CONSTRAINT = "uq_court_slot_active"


class ErrorResponse(BaseModel):
    detail: str
    request_id: str


def _envelope(request: Request, status: int, detail: str) -> JSONResponse:
    request_id = getattr(request.state, "request_id", "")
    return JSONResponse(
        status_code=status,
        content=ErrorResponse(detail=detail, request_id=request_id).model_dump(),
        headers={"X-Request-ID": request_id},
    )


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(IntegrityError)
    async def handle_integrity_error(request: Request, exc: IntegrityError) -> JSONResponse:
        """ONLY the slot-uniqueness violation is a 409.

        Two players racing for one slot is the expected outcome of §4.1 and
        deserves a 409. Every OTHER integrity error is a bug: a missing NOT
        NULL column, an FK violation, a duplicate email on registration.
        Dressing those as "That slot is no longer available" lies to the
        caller on a request that has nothing to do with slots, tells them to
        retry something that can never succeed, and hides a genuine fault
        from 5xx alerting.
        """
        cause = getattr(exc.orig, "__cause__", None)
        if getattr(cause, "constraint_name", None) == SLOT_CONSTRAINT:
            return _envelope(request, 409, "That slot is no longer available.")

        logger.exception("unhandled integrity error")
        return _envelope(request, 500, "Internal server error.")

    @app.exception_handler(Exception)
    async def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
        """Without this, an unhandled exception escapes to Starlette's
        ServerErrorMiddleware, which emits a bare 500 carrying NO
        X-Request-ID — so the one class of response where a support engineer
        most needs the correlation ID is the only class that lacks it.
        """
        logger.exception("unhandled exception")
        return _envelope(request, 500, "Internal server error.")
```

- [ ] **Step 5: Implement pagination**

`src/utils/pagination.py`:

```python
from typing import Generic, TypeVar

from pydantic import BaseModel

T = TypeVar("T")


class Page(BaseModel, Generic[T]):
    """The list envelope every collection endpoint returns (spec §3.5).

    One shape, defined once, so the frontend's pagination components target it
    directly instead of learning a new response per endpoint.
    """

    data: list[T]
    total: int
    page: int
    limit: int
    has_more: bool

    @classmethod
    def of(cls, data: list[T], total: int, page: int, limit: int) -> "Page[T]":
        return cls(
            data=data,
            total=total,
            page=page,
            limit=limit,
            has_more=(page * limit) < total,
        )
```

- [ ] **Step 6: Implement Manila day boundaries**

`src/utils/time.py`:

```python
from datetime import UTC, date, datetime, time, timedelta
from zoneinfo import ZoneInfo

MANILA = ZoneInfo("Asia/Manila")


def manila_day_bounds(day: date) -> tuple[datetime, datetime]:
    """Half-open UTC bounds [start, end) of one Manila calendar day.

    EVERY day-bounded query goes through this. Writing
    `WHERE starts_at::date = :day` instead slices on UTC days, which returns
    the wrong 24 hours and puts the 12am-1am slot — the one with its own
    price rule — on the previous day's grid.

    PH has no DST, so the offset is a fixed +08 and this needs no special
    cases.
    """
    start = datetime.combine(day, time.min, tzinfo=MANILA)
    return start.astimezone(UTC), (start + timedelta(days=1)).astimezone(UTC)
```

- [ ] **Step 7: Wire it all into the app**

Replace `src/main.py`:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from src.config.settings import Settings, get_settings
from src.health.controllers.health import router as health_router
from src.utils.errors import register_exception_handlers
from src.utils.log import configure_logging
from src.utils.request_id import RequestIDMiddleware


# Configured once at import, NOT inside create_app. configure_logging
# replaces the root handler list, and the `client` fixture builds an app per
# test — reconfiguring mid-session destroys pytest's caplog handler and drops
# any handler an operator attached.
configure_logging(get_settings().debug)


def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or get_settings()

    # OpenAPI lists every admin route and its request shape, so it is off
    # outside development — the same call asima makes.
    app = FastAPI(
        title="Pikol API",
        version="0.1.0",
        docs_url="/docs" if settings.debug else None,
        redoc_url=None,
        openapi_url="/openapi.json" if settings.debug else None,
    )

    # add_middleware prepends, so CORS ends up OUTERMOST — which is what a
    # preflight request needs.
    app.add_middleware(RequestIDMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_allowed_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
        expose_headers=["X-Request-ID"],
    )

    register_exception_handlers(app)
    app.include_router(health_router)
    return app


app = create_app()
```

> **Run uvicorn with `--no-access-log`.** Its access logger sets
> `propagate = False` and keeps its own plain-text formatter, so leaving it on
> means every request is logged twice — once as JSON by the middleware, once
> as plain text by uvicorn — and the JSON-lines format stops being greppable.

- [ ] **Step 8: Run the full suite**

Run: `uv run ruff format . && uv run pytest -v`
Expected: PASS — 27 passed

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "Add shared utilities: request ID, logging, pagination, Manila time

The Manila helper is the important one. The day-boundary rule is the single
most dangerous thing in the spec — 'starts_at::date' returns the wrong 24
hours and moves the 12am-1am slot, which has its own price rule, onto the
previous day. Making it a tested function turns 'everyone must remember
this' into 'you cannot get this wrong', and the test that pins it asserts
exactly that slot lands inside its own day.

Logging is JSON lines carrying the current request_id from a ContextVar, so
one failed booking is traceable across every line it produced without
threading an ID through every call site. Provider payloads are never
logged — they hold payer PII and live in their tables until purged.

Page is defined once so the frontend targets one shape; its test pins the
off-by-one where a page exactly ending the set reports has_more.

/docs and /openapi.json are off outside development: the schema lists every
admin route and its request shape."
```

---

### Task 7: CI pipeline

**Files:**
- Create: `pikol-backend/.github/workflows/ci.yml`
- Create: `pikol-backend/README.md`

**Interfaces:**
- Consumes: everything above.
- Produces: a CI run that gates every push.

- [ ] **Step 1: Verify all three gates pass locally**

```bash
uv run ruff check .
uv run ruff format --check .
uv run mypy src
uv run pytest -v
```

Fix anything red before continuing. CI must be green on its first run, or it
teaches everyone to ignore it.

- [ ] **Step 2: Write the workflow**

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: pikol
          POSTGRES_PASSWORD: pikol
          POSTGRES_DB: pikol
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U pikol"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

    env:
      DATABASE_URL: postgresql+asyncpg://pikol:pikol@localhost:5432/pikol
      CORS_ALLOWED_ORIGINS: '["http://localhost:3000"]'

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - run: uv sync --frozen

      - name: Lint
        run: uv run ruff check .

      - name: Format check
        run: uv run ruff format --check .

      - name: Type check
        run: uv run mypy src

      - name: Migrate
        run: uv run alembic upgrade head

      - name: Test
        run: uv run pytest -v
```

- [ ] **Step 3: Write the README**

`README.md`:

````markdown
# pikol-backend

Pikol court booking API — FastAPI, SQLAlchemy 2.0 async, Postgres.

Design: `../docs/plans/2026-09-09-court-booking-system-design.md`

## Local setup

```bash
cp .env.example .env
docker compose up -d
uv sync
uv run alembic upgrade head
uv run uvicorn src.main:app --reload --no-access-log
```

API docs at http://localhost:8000/docs, health at
http://localhost:8000/health.

## Checks

```bash
uv run ruff check . && uv run mypy src && uv run pytest
```

## Migrations

```bash
uv run alembic revision --autogenerate -m "description"
uv run alembic upgrade head
uv run alembic downgrade -1
```

**Against Supabase:** load `.env.supabase.prod` (gitignored) and use the
**session pooler on port 5432**, never the transaction pooler on 6543 —
asyncpg uses prepared statements, which 6543 does not support, and the
failure appears only under production load.
````

- [ ] **Step 4: Write `CLAUDE.md` so the conventions auto-load**

This is the asima pattern: a repo-level brief that loads automatically for
any session working under this directory. Without it the Global Constraints
live only in a plan nobody re-reads, and the first convention to be broken is
usually snake_case at the wire boundary.

`CLAUDE.md` (four-backtick fence — the block contains a ```` ```bash ````
fence of its own, and a three-backtick outer fence would be closed by it,
swallowing the next step):

````markdown
# pikol-backend

FastAPI + SQLAlchemy 2.0 (async) + Postgres. The court booking API for Pikol.

System-level brief: `../CLAUDE.md`.
Design: `../docs/plans/2026-09-09-court-booking-system-design.md`.

## Non-negotiables

- **snake_case end-to-end** — DB column, domain object, and JSON payload use
  the same name. No camelCase translation at the boundary.
- **All routes under `/api/v1/`** via `src/utils/api.py:API_PREFIX`.
  `/health` is deliberately outside it — infrastructure, not API.
- **Money is integer centavos.** Never float. Never a `Decimal` column.
- **Times are `timestamptz` in UTC. Day boundaries are Manila-local** —
  never `starts_at::date`, which returns the wrong 24 hours and misplaces
  the 12am-1am slot onto the previous day.
- **Request schemas set `extra="forbid"`.**
- **No `if (env)` in application code.** Environment differences are config
  and adapter selection only.
- **Every read of `booking_status = 'PENDING'` filters `expires_at > now()`.**
  Cleanup is lazy, so a lapsed hold keeps its row. `PENDING` alone never
  means "currently holding". No exceptions — this one has bitten already.

## Layering

Each feature is `src/<feature>/{controllers,domain,dto,persistence}/`,
mirroring asima's NestJS modules. The domain layer is **rich only where
invariants live** — `bookings`, `pricing`, `payments` get aggregates and
value objects with no SQLAlchemy imports. CRUD features (`venues`, `courts`,
`users`) go router → service → model, with no mapper.

`config/` is environment (secrets, read at boot). `parameters/` is business
rules (prices, windows, caps — in Postgres, tunable without a deploy). Do
not conflate them.

## Commands

```bash
docker compose up -d                       # local Postgres
uv run alembic upgrade head
uv run uvicorn src.main:app --reload --no-access-log
uv run ruff check . && uv run mypy src && uv run pytest
```
````

- [ ] **Step 5: Commit and confirm CI is green**

```bash
git add -A
git commit -m "Add CI pipeline with lint, types, migrations, and tests

Runs migrations against a real Postgres service rather than mocking the
database, so a migration that does not apply cleanly fails here rather than
on a deploy."
git push -u origin main    # after creating the remote
```

Then check the Actions tab. **Expected: green.** If it is red, fix it before
starting Phase 2 — a red main branch at the foundation stage compounds.

---

## Definition of done

- [ ] `uv run uvicorn src.main:app` serves `/health` → `{"status": "ok"}`
- [ ] `/docs` renders the OpenAPI UI
- [ ] `alembic upgrade head` from an empty database creates `parameters` with
      10 seeded rows
- [ ] `alembic downgrade base` succeeds
- [ ] `ParameterService.get()` returns typed values and rejects a bad one
- [ ] `/health/ready` returns `{"status": "ready", "database": "ok"}` and is
      what the keep-alive monitor targets
- [ ] Every response carries `X-Request-ID`, and log lines carry it too
- [ ] `/docs` is 404 when `debug` is false
- [ ] `manila_day_bounds(date(2026, 9, 15))` starts at `2026-09-14T16:00Z`
- [ ] The `db_session` fixture rolls back, so tests cannot leak state
- [ ] `ruff`, `mypy --strict`, and `pytest` all pass
- [ ] CI is green on `main`
- [ ] `CLAUDE.md` exists and states the non-negotiables

## What Phase 2 consumes from this

- `create_app()` — auth routers register here
- `get_session` — the session dependency for every repository
- `Base` — auth models extend it
- `parameter_service` — reads `invitation_expiry_days`
- `register_exception_handlers` — auth error types are added here
- `Page[T]` — every list endpoint returns it
- `manila_day_bounds` — every day-bounded query
- `db_session` — the fixture every write test uses
- The native-enum migration shape — `create_type=False` plus an explicit
  `.create()`, copied for `booking_status`, `payment_status`, and the rest
- `ParameterModel.updated_by_user_id` — gains its FK to `users` in Phase 2's
  migration
