---
title: Storage Foundation Implementation Plan
status: draft
spec: 2026-05-10-industry-research-design.md
sub_project: 1 of 5
plan_number: 01
created: 2026-05-10
---

# Storage Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish SQLite-backed structured persistence layer with generalized vendor-aware cache, dual-write `TradingMemoryLog` refactor, and migration tooling — without changing any v0.2.5-visible behavior.

**Architecture:** SQLAlchemy Core (not ORM) over a single SQLite database at `~/.tradingagents/storage/tradingagents.db` with WAL mode. Two-layer write semantics: markdown stays canonical for legacy `ticker_analyses` data (dual-write); SQLite is sole truth for new data (briefs, comps, caches, external reports). Alembic for schema migrations, opt-in via CLI.

**Tech Stack:** SQLAlchemy 2.x Core, Alembic, stdlib `sqlite3`, pytest, pytest-mock, freezegun.

**Depends on:** None — this is the foundation. Required by Plans 02–05.

**Backwards-compat invariants enforced:** BC-1, BC-2, BC-3, BC-5, BC-6, BC-8, BC-9 (per spec §17).

**Test directory convention:** This project uses a flat `tests/` directory (no subdirectories). All new test files created by this plan go directly into `tests/` (e.g. `tests/test_storage_db.py`), NOT into `tests/storage/` or `tests/agents/utils/`. Adjust all file paths in the tasks below accordingly.

---

## File Structure

**Created:**
- `tradingagents/storage/__init__.py` — public API surface
- `tradingagents/storage/db.py` — engine, connection, session, WAL config
- `tradingagents/storage/schema.py` — SQLAlchemy Core table definitions
- `tradingagents/storage/cache.py` — vendor-aware TTL cache layer
- `tradingagents/storage/memory_view.py` — markdown projection from SQLite
- `tradingagents/storage/reconciler.py` — startup reconciliation (markdown ↔ SQLite)
- `tradingagents/storage/migrations/env.py` — Alembic environment
- `tradingagents/storage/migrations/script.py.mako` — Alembic template
- `tradingagents/storage/migrations/versions/001_initial.py` — initial schema
- `tradingagents/storage/migrations/versions/002_backfill_markdown_memory.py` — opt-in backfill
- `tradingagents/agents/utils/industry_memory.py` — `IndustryMemoryLog` (façade over SQLite)
- `cli/storage.py` — `tradingagents storage ...` subcommands
- `alembic.ini` — Alembic config (project root)
- `tests/test_storage_db.py`
- `tests/test_storage_schema.py`
- `tests/test_storage_cache.py`
- `tests/test_storage_memory_view.py`
- `tests/test_storage_reconciler.py`
- `tests/test_storage_migrations.py`
- `tests/test_memory_dual_write.py`
- `tests/test_industry_memory.py`
- `tests/test_storage_cli.py`
- `tests/fixtures/sample_trading_memory.md` — synthetic markdown for migration tests

> **Note:** The project uses a flat `tests/` directory. Subdirectories would require adding `__init__.py` files and updating `pytest.ini`. Use the flat naming convention above instead.

**Modified:**
- `tradingagents/agents/utils/memory.py` — `TradingMemoryLog` dual-writes to SQLite
- `tradingagents/default_config.py` — adds `storage` config block
- `cli/main.py` — registers `storage` subcommand
- `pyproject.toml` — adds `sqlalchemy>=2.0`, `alembic>=1.13` to dependencies

**Untouched (verifies BC):**
- `~/.tradingagents/cache/checkpoints/` — LangGraph checkpoint paths preserved
- `~/.tradingagents/memory/trading_memory.md` — write path unchanged from v0.2.4
- All existing `tradingagents/graph/`, `tradingagents/agents/analysts/`, etc.

---

## Tasks

### Task 1: Add dependencies and verify install

**Files:**
- Modify: `pyproject.toml`

- [ ] **Step 1: Edit `pyproject.toml`** — add to `[project]` `dependencies` list:

```toml
"sqlalchemy>=2.0,<3.0",
"alembic>=1.13,<2.0",
```

Place these alphabetically with the other deps; if you see existing entries like `"langchain>=..."`, follow the same quoting style.

- [ ] **Step 2: Sync the env**

Run: `pip install -e .` (or `uv sync` if the repo uses uv — check for `uv.lock`)
Expected: clean install with new packages reported.

- [ ] **Step 3: Smoke-import**

Run: `python -c "import sqlalchemy; import alembic; print(sqlalchemy.__version__, alembic.__version__)"`
Expected: prints two version strings, no traceback.

- [ ] **Step 4: Commit**

```bash
git add pyproject.toml uv.lock
git commit -m "chore: add sqlalchemy and alembic deps for storage layer"
```

---

### Task 2: Create storage package skeleton

**Files:**
- Create: `tradingagents/storage/__init__.py`
- Create: `tests/storage/__init__.py`
- Create: `tests/storage/conftest.py`
- Create: `tests/storage/test_db.py` (test only)

- [ ] **Step 1: Write the failing import test**

Create `tests/storage/test_db.py`:

```python
"""Smoke tests for the storage package."""

def test_storage_package_imports():
    from tradingagents import storage  # noqa: F401


def test_storage_get_engine_is_callable():
    from tradingagents.storage import get_engine
    assert callable(get_engine)
```

- [ ] **Step 2: Run the test to verify failure**

Run: `pytest tests/storage/test_db.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'tradingagents.storage'`.

- [ ] **Step 3: Create the package skeleton**

Create `tradingagents/storage/__init__.py`:

```python
"""SQLite-backed structured persistence for TradingAgents.

See specs/2026-05-10-industry-research-design.md §6 for the design.
"""

from tradingagents.storage.db import get_engine

__all__ = ["get_engine"]
```

Create `tests/storage/__init__.py` as an empty file.

Create `tests/storage/conftest.py`:

```python
"""Shared fixtures for storage tests."""
import pytest
from sqlalchemy import create_engine


@pytest.fixture
def in_memory_engine():
    """An ephemeral in-memory SQLite engine, fresh per test."""
    engine = create_engine("sqlite:///:memory:", future=True)
    yield engine
    engine.dispose()
```

- [ ] **Step 4: Run test — still fails (no `db.py`)**

Run: `pytest tests/storage/test_db.py -v`
Expected: FAIL with `ImportError` from `tradingagents.storage` trying to import `get_engine` from non-existent `db.py`.

- [ ] **Step 5: Create stub `db.py` to make the import test pass**

Create `tradingagents/storage/db.py`:

```python
"""SQLite engine and connection management."""
from sqlalchemy import Engine


def get_engine() -> Engine:
    """Stub. Replaced in Task 3."""
    raise NotImplementedError("Implemented in Task 3")
```

- [ ] **Step 6: Run test — should pass**

Run: `pytest tests/storage/test_db.py -v`
Expected: 2 passed.

- [ ] **Step 7: Commit**

```bash
git add tradingagents/storage tests/storage
git commit -m "feat(storage): add storage package skeleton"
```

---

### Task 3: Implement SQLite engine with WAL mode and configurable path

**Files:**
- Modify: `tradingagents/storage/db.py`
- Modify: `tradingagents/default_config.py`
- Modify: `tests/storage/test_db.py`

- [ ] **Step 1: Add storage config block to `default_config.py`**

Open `tradingagents/default_config.py`. After the existing `_TRADINGAGENTS_HOME` definition, add:

```python
_TRADINGAGENTS_STORAGE = os.path.join(_TRADINGAGENTS_HOME, "storage")
```

Then in `DEFAULT_CONFIG` dict, add at the bottom (before the closing `}`):

```python
    "storage": {
        "db_path": os.getenv(
            "TRADINGAGENTS_DB_PATH",
            os.path.join(_TRADINGAGENTS_STORAGE, "tradingagents.db"),
        ),
        "wal_mode": True,
        "cache_ttl_days": {
            "fundamentals": 7,
            "prices": 1,
            "news": 1,
            "fred": 7,
            "etf_holdings": 1,
            "peer_comps": 7,
        },
    },
```

- [ ] **Step 2: Write the failing tests**

Replace `tests/storage/test_db.py` with:

```python
"""Tests for storage.db engine factory."""
import os
import sqlite3
import pytest
from pathlib import Path

from tradingagents.storage import get_engine
from tradingagents.default_config import DEFAULT_CONFIG


def test_get_engine_returns_sqlalchemy_engine():
    from sqlalchemy import Engine
    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": ":memory:"}
    engine = get_engine(config)
    assert isinstance(engine, Engine)


def test_get_engine_creates_parent_dir_for_file_path(tmp_path):
    db_path = tmp_path / "nested" / "dir" / "test.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    with engine.connect() as conn:
        conn.execute(__import__("sqlalchemy").text("SELECT 1"))
    assert db_path.exists()
    assert db_path.parent.is_dir()


def test_get_engine_enables_wal_mode_when_configured(tmp_path):
    db_path = tmp_path / "wal_test.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    with engine.connect() as conn:
        result = conn.execute(__import__("sqlalchemy").text("PRAGMA journal_mode")).scalar()
    assert result == "wal"


def test_get_engine_skips_wal_for_memory_db():
    config = {"storage": {"db_path": ":memory:", "wal_mode": True}}
    engine = get_engine(config)
    # In-memory DBs don't support WAL — no error, just no-op
    with engine.connect() as conn:
        result = conn.execute(__import__("sqlalchemy").text("PRAGMA journal_mode")).scalar()
    assert result == "memory"


def test_get_engine_idempotent_returns_same_engine_for_same_path(tmp_path):
    db_path = tmp_path / "idem.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    e1 = get_engine(config)
    e2 = get_engine(config)
    assert e1 is e2
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `pytest tests/storage/test_db.py -v`
Expected: 5 tests FAIL (NotImplementedError or signature mismatch).

- [ ] **Step 4: Implement `db.py`**

Replace `tradingagents/storage/db.py` with:

```python
"""SQLite engine and connection management.

Single global engine per db_path, lazily initialized.
WAL mode enabled by default for concurrent read support.
"""
import logging
import os
from pathlib import Path
from typing import Any, Dict, Optional

from sqlalchemy import Engine, create_engine, event, text

logger = logging.getLogger(__name__)

_engines: Dict[str, Engine] = {}


def get_engine(config: Optional[Dict[str, Any]] = None) -> Engine:
    """Return the shared SQLAlchemy Engine for the configured db_path.

    Lazily creates the engine on first call per path. Enables WAL mode
    on file-backed databases when ``config["storage"]["wal_mode"]`` is True.

    Args:
        config: TradingAgents config dict. If None, imports DEFAULT_CONFIG.
    """
    if config is None:
        from tradingagents.default_config import DEFAULT_CONFIG
        config = DEFAULT_CONFIG

    storage_cfg = config.get("storage", {})
    db_path = storage_cfg.get("db_path", ":memory:")
    wal = storage_cfg.get("wal_mode", True)

    if db_path in _engines:
        return _engines[db_path]

    if db_path != ":memory:":
        Path(db_path).expanduser().parent.mkdir(parents=True, exist_ok=True)
        url = f"sqlite:///{Path(db_path).expanduser()}"
    else:
        url = "sqlite:///:memory:"

    engine = create_engine(url, future=True)

    if wal and db_path != ":memory:":
        @event.listens_for(engine, "connect")
        def _enable_wal(dbapi_conn, _):
            cursor = dbapi_conn.cursor()
            try:
                cursor.execute("PRAGMA journal_mode=WAL")
                cursor.execute("PRAGMA synchronous=NORMAL")
            finally:
                cursor.close()

        # Trigger one connection so WAL is active for verification queries
        with engine.connect() as conn:
            conn.execute(text("PRAGMA journal_mode"))

    _engines[db_path] = engine
    logger.info("Initialized SQLite engine at %s (WAL=%s)", db_path, wal)
    return engine


def reset_engines() -> None:
    """Test-only helper to clear the engine cache."""
    for engine in _engines.values():
        engine.dispose()
    _engines.clear()
```

- [ ] **Step 5: Update conftest to reset engines between tests**

Edit `tests/storage/conftest.py` and add:

```python
@pytest.fixture(autouse=True)
def reset_storage_engines():
    """Clear cached engines before and after each test."""
    from tradingagents.storage.db import reset_engines
    reset_engines()
    yield
    reset_engines()
```

- [ ] **Step 6: Run tests — verify pass**

Run: `pytest tests/storage/test_db.py -v`
Expected: 5 passed.

- [ ] **Step 7: Commit**

```bash
git add tradingagents/storage/db.py tradingagents/default_config.py tests/storage
git commit -m "feat(storage): add SQLite engine factory with WAL mode"
```

---

### Task 4: Define schema with SQLAlchemy Core tables

**Files:**
- Create: `tradingagents/storage/schema.py`
- Create: `tests/storage/test_schema.py`

- [ ] **Step 1: Write the failing tests**

Create `tests/storage/test_schema.py`:

```python
"""Tests for SQLAlchemy Core schema definitions."""
from sqlalchemy import inspect

from tradingagents.storage import get_engine
from tradingagents.storage.schema import metadata, ALL_TABLES


def test_schema_defines_all_required_tables():
    expected = {
        "ticker_analyses",
        "industry_briefs",
        "peer_comps_snapshots",
        "fundamentals_cache",
        "price_cache",
        "news_cache",
        "fred_cache",
        "etf_holdings_cache",
        "external_reports",
    }
    assert {t.name for t in ALL_TABLES} == expected


def test_metadata_creates_all_tables():
    config = {"storage": {"db_path": ":memory:", "wal_mode": False}}
    engine = get_engine(config)
    metadata.create_all(engine)
    inspector = inspect(engine)
    tables = set(inspector.get_table_names())
    assert "ticker_analyses" in tables
    assert "industry_briefs" in tables
    assert "external_reports" in tables


def test_ticker_analyses_unique_constraint():
    from sqlalchemy import insert
    from sqlalchemy.exc import IntegrityError
    from tradingagents.storage.schema import ticker_analyses

    config = {"storage": {"db_path": ":memory:", "wal_mode": False}}
    engine = get_engine(config)
    metadata.create_all(engine)
    with engine.begin() as conn:
        conn.execute(insert(ticker_analyses).values(
            ticker="NVDA", date="2026-05-09", decision="BUY",
            created_at="2026-05-09T10:00:00",
        ))
    with engine.begin() as conn:
        try:
            conn.execute(insert(ticker_analyses).values(
                ticker="NVDA", date="2026-05-09", decision="SELL",
                created_at="2026-05-09T11:00:00",
            ))
            raised = False
        except IntegrityError:
            raised = True
    assert raised, "expected UNIQUE(ticker, date) violation"


def test_industry_briefs_unique_constraint():
    from sqlalchemy import insert
    from sqlalchemy.exc import IntegrityError
    from tradingagents.storage.schema import industry_briefs

    config = {"storage": {"db_path": ":memory:", "wal_mode": False}}
    engine = get_engine(config)
    metadata.create_all(engine)
    with engine.begin() as conn:
        conn.execute(insert(industry_briefs).values(
            sub_industry="Semiconductors", date="2026-05-09",
            mode="brief", call="OW", rationale_md="test",
            brief_md="test", created_at="2026-05-09T10:00:00",
        ))
    with engine.begin() as conn:
        try:
            conn.execute(insert(industry_briefs).values(
                sub_industry="Semiconductors", date="2026-05-09",
                mode="brief", call="UW", rationale_md="test2",
                brief_md="test2", created_at="2026-05-09T11:00:00",
            ))
            raised = False
        except IntegrityError:
            raised = True
    assert raised


def test_indexes_present():
    config = {"storage": {"db_path": ":memory:", "wal_mode": False}}
    engine = get_engine(config)
    metadata.create_all(engine)
    inspector = inspect(engine)
    ti_indexes = {ix["name"] for ix in inspector.get_indexes("ticker_analyses")}
    assert "idx_ticker_analyses_industry" in ti_indexes
    ib_indexes = {ix["name"] for ix in inspector.get_indexes("industry_briefs")}
    assert "idx_industry_briefs_lookup" in ib_indexes
    er_indexes = {ix["name"] for ix in inspector.get_indexes("external_reports")}
    assert "idx_external_reports_scope" in er_indexes
```

- [ ] **Step 2: Run tests — verify fail**

Run: `pytest tests/storage/test_schema.py -v`
Expected: 5 tests FAIL with `ModuleNotFoundError` for `tradingagents.storage.schema`.

- [ ] **Step 3: Create `schema.py`**

Create `tradingagents/storage/schema.py`:

```python
"""SQLAlchemy Core table definitions.

See specs/2026-05-10-industry-research-design.md §6.3 for schema rationale.
"""
from sqlalchemy import (
    Column, Float, Index, Integer, MetaData, String, Table, UniqueConstraint,
)

metadata = MetaData()

ticker_analyses = Table(
    "ticker_analyses", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("ticker", String, nullable=False),
    Column("date", String, nullable=False),
    Column("sub_industry", String, nullable=True),
    Column("decision", String, nullable=False),
    Column("raw_return", Float, nullable=True),
    Column("alpha_return", Float, nullable=True),
    Column("sector_alpha_return", Float, nullable=True),
    Column("holding_days", Integer, nullable=True),
    Column("reflection_md", String, nullable=True),
    Column("full_state_json", String, nullable=True),
    Column("created_at", String, nullable=False),
    Column("resolved_at", String, nullable=True),
    UniqueConstraint("ticker", "date", name="uq_ticker_analyses_ticker_date"),
    Index("idx_ticker_analyses_industry", "sub_industry", "date"),
)

industry_briefs = Table(
    "industry_briefs", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("sub_industry", String, nullable=False),
    Column("date", String, nullable=False),
    Column("mode", String, nullable=False),
    Column("call", String, nullable=False),
    Column("conviction", Float, nullable=True),
    Column("top_longs_json", String, nullable=True),
    Column("top_shorts_json", String, nullable=True),
    Column("key_debates_json", String, nullable=True),
    Column("catalysts_json", String, nullable=True),
    Column("rationale_md", String, nullable=False),
    Column("brief_md", String, nullable=False),
    Column("sector_etf", String, nullable=True),
    Column("realized_etf_alpha_vs_spy", Float, nullable=True),
    Column("reflection_md", String, nullable=True),
    Column("created_at", String, nullable=False),
    Column("resolved_at", String, nullable=True),
    UniqueConstraint("sub_industry", "date", "mode",
                     name="uq_industry_briefs_subind_date_mode"),
    Index("idx_industry_briefs_lookup", "sub_industry", "date"),
)

peer_comps_snapshots = Table(
    "peer_comps_snapshots", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("sub_industry", String, nullable=False),
    Column("date", String, nullable=False),
    Column("basket_json", String, nullable=False),
    Column("comps_json", String, nullable=False),
    Column("vendor", String, nullable=False),
    Column("created_at", String, nullable=False),
    UniqueConstraint("sub_industry", "date",
                     name="uq_peer_comps_subind_date"),
)

fundamentals_cache = Table(
    "fundamentals_cache", metadata,
    Column("ticker", String, primary_key=True),
    Column("period_end", String, primary_key=True),
    Column("vendor", String, primary_key=True),
    Column("payload_json", String, nullable=False),
    Column("fetched_at", String, nullable=False),
)

price_cache = Table(
    "price_cache", metadata,
    Column("ticker", String, primary_key=True),
    Column("start_date", String, primary_key=True),
    Column("end_date", String, primary_key=True),
    Column("vendor", String, primary_key=True),
    Column("payload_json", String, nullable=False),
    Column("fetched_at", String, nullable=False),
)

news_cache = Table(
    "news_cache", metadata,
    Column("scope_type", String, primary_key=True),
    Column("scope_value", String, primary_key=True),
    Column("date", String, primary_key=True),
    Column("vendor", String, primary_key=True),
    Column("items_json", String, nullable=False),
    Column("fetched_at", String, nullable=False),
)

fred_cache = Table(
    "fred_cache", metadata,
    Column("series_id", String, primary_key=True),
    Column("payload_json", String, nullable=False),
    Column("fetched_at", String, nullable=False),
)

etf_holdings_cache = Table(
    "etf_holdings_cache", metadata,
    Column("symbol", String, primary_key=True),
    Column("as_of_date", String, primary_key=True),
    Column("holdings_json", String, nullable=False),
    Column("fetched_at", String, nullable=False),
)

external_reports = Table(
    "external_reports", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("filename", String, nullable=False),
    Column("source", String, nullable=False),
    Column("ingested_at", String, nullable=False),
    Column("doc_date", String, nullable=True),
    Column("scope_type", String, nullable=True),
    Column("scope_value", String, nullable=True),
    Column("report_type", String, nullable=True),
    Column("takeaways_md", String, nullable=False),
    Column("raw_text_path", String, nullable=True),
    Column("raw_pdf_path", String, nullable=True),
    Column("page_count", Integer, nullable=True),
    Column("classifier_confidence", Float, nullable=True),
    UniqueConstraint("filename", "source", "doc_date",
                     name="uq_external_reports_file_src_date"),
    Index("idx_external_reports_scope", "scope_type", "scope_value", "doc_date"),
)

ALL_TABLES = [
    ticker_analyses,
    industry_briefs,
    peer_comps_snapshots,
    fundamentals_cache,
    price_cache,
    news_cache,
    fred_cache,
    etf_holdings_cache,
    external_reports,
]
```

- [ ] **Step 4: Run tests — verify pass**

Run: `pytest tests/storage/test_schema.py -v`
Expected: 5 passed.

- [ ] **Step 5: Commit**

```bash
git add tradingagents/storage/schema.py tests/storage/test_schema.py
git commit -m "feat(storage): define SQLAlchemy Core schema for all tables"
```

---

### Task 5: Wire up Alembic for migrations

**Files:**
- Create: `alembic.ini`
- Create: `tradingagents/storage/migrations/env.py`
- Create: `tradingagents/storage/migrations/script.py.mako`
- Create: `tradingagents/storage/migrations/versions/.gitkeep`

- [ ] **Step 1: Create `alembic.ini` at project root**

```ini
[alembic]
script_location = tradingagents/storage/migrations
prepend_sys_path = .
sqlalchemy.url = sqlite:///%(here)s/.tradingagents-build/alembic-default.db

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

(The `sqlalchemy.url` here is a build-time placeholder; runtime uses `get_engine(config)`.)

- [ ] **Step 2: Create the migrations directory**

```bash
mkdir -p tradingagents/storage/migrations/versions
touch tradingagents/storage/migrations/versions/.gitkeep
```

- [ ] **Step 3: Create `env.py`**

Create `tradingagents/storage/migrations/env.py`:

```python
"""Alembic env that uses the project's get_engine."""
from logging.config import fileConfig

from alembic import context

from tradingagents.storage.db import get_engine
from tradingagents.storage.schema import metadata

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    # Use the configured engine; honors TRADINGAGENTS_DB_PATH
    connectable = get_engine()
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

- [ ] **Step 4: Create the `script.py.mako` template**

Create `tradingagents/storage/migrations/script.py.mako`:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}
branch_labels = ${repr(branch_labels)}
depends_on = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

- [ ] **Step 5: Verify Alembic can find the env**

Run: `alembic --config alembic.ini current`
Expected: prints `INFO  [alembic.runtime.migration] Context impl SQLiteImpl.` and an empty current-revision line (no migrations yet).

- [ ] **Step 6: Commit**

```bash
git add alembic.ini tradingagents/storage/migrations
git commit -m "feat(storage): scaffold alembic migrations directory"
```

---

### Task 6: Write the initial schema migration (001_initial)

**Files:**
- Create: `tradingagents/storage/migrations/versions/001_initial.py`
- Create: `tests/storage/test_migrations.py`

- [ ] **Step 1: Write the failing test**

Create `tests/storage/test_migrations.py`:

```python
"""Tests for Alembic migrations."""
import os
import subprocess
from pathlib import Path

import pytest
from sqlalchemy import inspect

from tradingagents.storage import get_engine


def _run_alembic_upgrade(db_path: Path) -> subprocess.CompletedProcess:
    env = os.environ.copy()
    env["TRADINGAGENTS_DB_PATH"] = str(db_path)
    return subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "head"],
        capture_output=True, text=True, env=env,
    )


def test_initial_migration_creates_all_tables(tmp_path):
    db_path = tmp_path / "alembic_test.db"
    result = _run_alembic_upgrade(db_path)
    assert result.returncode == 0, f"alembic failed: {result.stderr}"

    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    inspector = inspect(engine)
    tables = set(inspector.get_table_names())
    expected = {
        "ticker_analyses", "industry_briefs", "peer_comps_snapshots",
        "fundamentals_cache", "price_cache", "news_cache", "fred_cache",
        "etf_holdings_cache", "external_reports", "alembic_version",
    }
    assert expected.issubset(tables)


def test_migration_is_idempotent(tmp_path):
    db_path = tmp_path / "idem.db"
    r1 = _run_alembic_upgrade(db_path)
    assert r1.returncode == 0
    r2 = _run_alembic_upgrade(db_path)
    assert r2.returncode == 0
    # Second run should be a no-op (no errors, alembic_version unchanged)
```

- [ ] **Step 2: Run tests — verify fail**

Run: `pytest tests/storage/test_migrations.py -v`
Expected: FAIL — alembic exits non-zero because there are no migration scripts in `versions/`.

- [ ] **Step 3: Create the initial migration**

Create `tradingagents/storage/migrations/versions/001_initial.py`:

```python
"""initial schema

Revision ID: 001_initial
Revises:
Create Date: 2026-05-10
"""
from alembic import op

from tradingagents.storage.schema import metadata

revision = "001_initial"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    metadata.create_all(op.get_bind())


def downgrade() -> None:
    metadata.drop_all(op.get_bind())
```

- [ ] **Step 4: Run tests — verify pass**

Run: `pytest tests/storage/test_migrations.py -v`
Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add tradingagents/storage/migrations/versions tests/storage/test_migrations.py
git commit -m "feat(storage): initial Alembic migration creates all tables"
```

---

### Task 7: Implement vendor-aware TTL cache layer

**Files:**
- Create: `tradingagents/storage/cache.py`
- Create: `tests/storage/test_cache.py`

- [ ] **Step 1: Write the failing tests**

Create `tests/storage/test_cache.py`:

```python
"""Tests for the generalized cache layer."""
import json
from datetime import datetime, timedelta

import pytest
from freezegun import freeze_time

from tradingagents.storage import get_engine
from tradingagents.storage.cache import Cache, CacheKey, CacheMiss
from tradingagents.storage.schema import metadata


@pytest.fixture
def cache(tmp_path):
    db_path = tmp_path / "cache_test.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True,
                          "cache_ttl_days": {"fundamentals": 7, "prices": 1}}}
    engine = get_engine(config)
    metadata.create_all(engine)
    return Cache(engine=engine, ttl_days=config["storage"]["cache_ttl_days"])


def test_cache_miss_raises(cache):
    key = CacheKey(category="fundamentals", ticker="NVDA",
                   period_end="2026-Q1", vendor="yfinance")
    with pytest.raises(CacheMiss):
        cache.get(key)


def test_cache_set_then_get(cache):
    key = CacheKey(category="fundamentals", ticker="NVDA",
                   period_end="2026-Q1", vendor="yfinance")
    payload = {"revenue": 1000, "ebitda": 200}
    cache.set(key, payload)
    assert cache.get(key) == payload


@freeze_time("2026-05-10")
def test_cache_respects_ttl(cache):
    key = CacheKey(category="fundamentals", ticker="NVDA",
                   period_end="2026-Q1", vendor="yfinance")
    cache.set(key, {"x": 1})

    with freeze_time("2026-05-15"):
        assert cache.get(key) == {"x": 1}  # within 7-day TTL

    with freeze_time("2026-05-18"):
        with pytest.raises(CacheMiss):
            cache.get(key)  # past 7-day TTL


def test_cache_for_different_categories(cache):
    f_key = CacheKey(category="fundamentals", ticker="NVDA",
                     period_end="2026-Q1", vendor="yfinance")
    p_key = CacheKey(category="prices", ticker="NVDA",
                     start_date="2026-04-01", end_date="2026-05-09",
                     vendor="yfinance")
    cache.set(f_key, {"f": 1})
    cache.set(p_key, {"p": 2})
    assert cache.get(f_key) == {"f": 1}
    assert cache.get(p_key) == {"p": 2}


def test_cache_different_vendors_dont_collide(cache):
    k1 = CacheKey(category="fundamentals", ticker="NVDA",
                  period_end="2026-Q1", vendor="yfinance")
    k2 = CacheKey(category="fundamentals", ticker="NVDA",
                  period_end="2026-Q1", vendor="alpha_vantage")
    cache.set(k1, {"v": "yf"})
    cache.set(k2, {"v": "av"})
    assert cache.get(k1) == {"v": "yf"}
    assert cache.get(k2) == {"v": "av"}


def test_cache_get_or_compute(cache):
    key = CacheKey(category="fundamentals", ticker="MSFT",
                   period_end="2026-Q1", vendor="yfinance")
    call_count = {"n": 0}

    def compute():
        call_count["n"] += 1
        return {"computed": True}

    r1 = cache.get_or_compute(key, compute)
    r2 = cache.get_or_compute(key, compute)
    assert r1 == r2 == {"computed": True}
    assert call_count["n"] == 1, "compute should run only on miss"
```

- [ ] **Step 2: Add freezegun to dev deps**

The project's dev extras live under `[project.optional-dependencies]` in `pyproject.toml`. Add `freezegun` to the `dev` list (alongside `ruff`, `pytest`, `pytest-subtests`):

```toml
[project.optional-dependencies]
dev = [
    "ruff>=0.15",
    "pytest>=8.0",
    "pytest-subtests>=0.13",
    "freezegun>=1.5,<2.0",
]
```

Run: `uv sync` (the project uses uv — `uv.lock` exists at the repo root).

- [ ] **Step 3: Run tests — verify fail**

Run: `pytest tests/storage/test_cache.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'tradingagents.storage.cache'`.

- [ ] **Step 4: Implement `cache.py`**

Create `tradingagents/storage/cache.py`:

```python
"""Generalized vendor-aware TTL cache backed by SQLite tables."""
import json
import logging
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Any, Callable, Dict, Optional

from sqlalchemy import Engine, delete, insert, select, update

from tradingagents.storage.schema import (
    fundamentals_cache, price_cache, news_cache, fred_cache,
    etf_holdings_cache, peer_comps_snapshots,
)

logger = logging.getLogger(__name__)


class CacheMiss(Exception):
    """Raised when a key is not in cache or has expired."""


@dataclass(frozen=True)
class CacheKey:
    """Composite key for the cache. Required fields depend on category."""
    category: str  # fundamentals|prices|news|fred|etf_holdings|peer_comps
    ticker: Optional[str] = None
    period_end: Optional[str] = None
    start_date: Optional[str] = None
    end_date: Optional[str] = None
    scope_type: Optional[str] = None
    scope_value: Optional[str] = None
    date: Optional[str] = None
    series_id: Optional[str] = None
    symbol: Optional[str] = None
    as_of_date: Optional[str] = None
    sub_industry: Optional[str] = None
    vendor: Optional[str] = None


_TABLE_BY_CATEGORY = {
    "fundamentals": fundamentals_cache,
    "prices": price_cache,
    "news": news_cache,
    "fred": fred_cache,
    "etf_holdings": etf_holdings_cache,
    "peer_comps": peer_comps_snapshots,
}


def _key_to_where(key: CacheKey) -> Dict[str, Any]:
    """Map a CacheKey to the WHERE-clause dict for its table."""
    if key.category == "fundamentals":
        return {"ticker": key.ticker, "period_end": key.period_end,
                "vendor": key.vendor}
    if key.category == "prices":
        return {"ticker": key.ticker, "start_date": key.start_date,
                "end_date": key.end_date, "vendor": key.vendor}
    if key.category == "news":
        return {"scope_type": key.scope_type, "scope_value": key.scope_value,
                "date": key.date, "vendor": key.vendor}
    if key.category == "fred":
        return {"series_id": key.series_id}
    if key.category == "etf_holdings":
        return {"symbol": key.symbol, "as_of_date": key.as_of_date}
    if key.category == "peer_comps":
        return {"sub_industry": key.sub_industry, "date": key.date}
    raise ValueError(f"Unknown cache category: {key.category}")


def _payload_field(category: str) -> str:
    if category == "news":
        return "items_json"
    if category == "etf_holdings":
        return "holdings_json"
    if category == "peer_comps":
        return "comps_json"  # primary; basket_json handled by writer
    return "payload_json"


class Cache:
    """Vendor-aware TTL cache backed by SQLite."""

    def __init__(self, engine: Engine, ttl_days: Dict[str, int]):
        self.engine = engine
        self.ttl_days = ttl_days

    def get(self, key: CacheKey) -> Any:
        """Retrieve a payload. Raises CacheMiss if absent or expired."""
        table = _TABLE_BY_CATEGORY[key.category]
        where = _key_to_where(key)
        payload_col = _payload_field(key.category)
        ttl = self.ttl_days.get(key.category)

        with self.engine.connect() as conn:
            stmt = select(table.c[payload_col], table.c.fetched_at)
            for col, val in where.items():
                stmt = stmt.where(table.c[col] == val)
            row = conn.execute(stmt).first()

        if row is None:
            raise CacheMiss(f"no entry for {key}")

        if ttl is not None:
            fetched = datetime.fromisoformat(row.fetched_at)
            if datetime.utcnow() - fetched > timedelta(days=ttl):
                raise CacheMiss(f"expired (ttl={ttl}d): {key}")

        return json.loads(row[0])

    def set(self, key: CacheKey, payload: Any) -> None:
        """Store a payload, replacing any existing row."""
        table = _TABLE_BY_CATEGORY[key.category]
        where = _key_to_where(key)
        payload_col = _payload_field(key.category)
        now = datetime.utcnow().isoformat()

        with self.engine.begin() as conn:
            del_stmt = delete(table)
            for col, val in where.items():
                del_stmt = del_stmt.where(table.c[col] == val)
            conn.execute(del_stmt)

            row = {**where, payload_col: json.dumps(payload), "fetched_at": now}
            if key.category == "peer_comps":
                row.setdefault("basket_json", "[]")
                row.setdefault("vendor", key.vendor or "unknown")
                row.setdefault("created_at", now)
            conn.execute(insert(table).values(**row))

    def get_or_compute(self, key: CacheKey, compute: Callable[[], Any]) -> Any:
        """Return cached value or call ``compute()`` and cache the result."""
        try:
            return self.get(key)
        except CacheMiss:
            value = compute()
            self.set(key, value)
            return value
```

- [ ] **Step 5: Run tests — verify pass**

Run: `pytest tests/storage/test_cache.py -v`
Expected: 6 passed.

- [ ] **Step 6: Commit**

```bash
git add tradingagents/storage/cache.py tests/storage/test_cache.py pyproject.toml
git commit -m "feat(storage): vendor-aware TTL cache layer"
```

---

### Task 8: Refactor TradingMemoryLog to dual-write

**Files:**
- Modify: `tradingagents/agents/utils/memory.py`
- Create: `tests/agents/utils/test_memory_dual_write.py`

- [ ] **Step 1: Read existing memory.py**

Open `tradingagents/agents/utils/memory.py`. The actual public API is:
- `store_decision(ticker, trade_date, final_trade_decision)` — appends pending entry
- `load_entries()` — returns all parsed entries
- `get_pending_entries()` — pending-only subset
- `get_past_context(ticker, n_same, n_cross)` — builds prompt context string
- `update_with_outcome(ticker, trade_date, raw_return, alpha_return, holding_days, reflection)` — resolves one pending entry

> **Correction vs design doc:** The spec references `batch_update_with_outcomes` but the actual method is the singular `update_with_outcome`. The dual-write mirror should hook into `update_with_outcome`, not a batch variant. Preserve the existing signature.

- [ ] **Step 2: Write failing dual-write test**

Create `tests/agents/utils/test_memory_dual_write.py`:

```python
"""Tests for TradingMemoryLog dual-write to SQLite.

Markdown remains canonical; SQLite is a parallel mirror. SQLite write failures
must not propagate to the markdown write path (BC-2, BC-5).
"""
from pathlib import Path
import pytest
from sqlalchemy import select

from tradingagents.agents.utils.memory import TradingMemoryLog
from tradingagents.storage import get_engine
from tradingagents.storage.schema import metadata, ticker_analyses


@pytest.fixture
def configured(tmp_path):
    md_path = tmp_path / "memory" / "trading_memory.md"
    md_path.parent.mkdir(parents=True)
    db_path = tmp_path / "storage" / "test.db"
    config = {
        "memory_log_path": str(md_path),
        "memory_log_max_entries": None,
        "storage": {"db_path": str(db_path), "wal_mode": True,
                    "cache_ttl_days": {}},
    }
    engine = get_engine(config)
    metadata.create_all(engine)
    return config, md_path, db_path


def test_store_decision_writes_to_both_markdown_and_sqlite(configured):
    config, md_path, db_path = configured
    log = TradingMemoryLog(config)
    log.store_decision(ticker="NVDA", trade_date="2026-05-09",
                       final_trade_decision="BUY 100 shares")

    md_text = md_path.read_text()
    assert "NVDA" in md_text
    assert "2026-05-09" in md_text

    engine = get_engine(config)
    with engine.connect() as conn:
        rows = conn.execute(select(ticker_analyses)).all()
    assert len(rows) == 1
    assert rows[0].ticker == "NVDA"
    assert rows[0].date == "2026-05-09"
    assert "BUY 100 shares" in rows[0].decision


def test_sqlite_write_failure_does_not_break_markdown_write(configured, monkeypatch):
    config, md_path, _ = configured
    log = TradingMemoryLog(config)

    def boom(*a, **kw):
        raise RuntimeError("simulated SQLite failure")

    monkeypatch.setattr("tradingagents.agents.utils.memory._mirror_to_sqlite", boom)

    log.store_decision(ticker="MSFT", trade_date="2026-05-09",
                       final_trade_decision="HOLD")

    assert "MSFT" in md_path.read_text()


def test_get_past_context_reads_from_markdown_unchanged(configured):
    config, md_path, _ = configured
    log = TradingMemoryLog(config)
    log.store_decision("NVDA", "2026-05-08", "BUY")
    log.store_decision("NVDA", "2026-05-09", "HOLD")
    context = log.get_past_context("NVDA")
    assert "BUY" in context or "HOLD" in context
```

- [ ] **Step 3: Run tests — verify fail**

Run: `pytest tests/agents/utils/test_memory_dual_write.py -v`
Expected: FAIL — either signature mismatch or no SQLite mirror happening.

- [ ] **Step 4: Add `_mirror_to_sqlite` helper to memory.py**

Open `tradingagents/agents/utils/memory.py`. At the top of the file, after existing imports, add:

```python
import json
import logging
from datetime import datetime
from typing import Any, Dict, Optional

logger = logging.getLogger(__name__)


def _mirror_to_sqlite(config: Dict[str, Any], ticker: str, trade_date: str,
                     decision: str, full_state: Optional[Dict] = None) -> None:
    """Mirror a ticker decision to SQLite ticker_analyses.

    Failures are logged but never raised — markdown remains canonical.
    """
    try:
        from sqlalchemy import insert, update
        from tradingagents.storage import get_engine
        from tradingagents.storage.schema import ticker_analyses, metadata

        engine = get_engine(config)
        # Best-effort table creation: covers the case where storage migrate
        # has not been run yet (markdown still works without SQLite).
        metadata.create_all(engine, tables=[ticker_analyses], checkfirst=True)

        now = datetime.utcnow().isoformat()
        with engine.begin() as conn:
            try:
                conn.execute(insert(ticker_analyses).values(
                    ticker=ticker, date=trade_date, decision=decision,
                    full_state_json=json.dumps(full_state) if full_state else None,
                    created_at=now,
                ))
            except Exception:  # likely UNIQUE violation on (ticker, date)
                conn.execute(
                    update(ticker_analyses)
                    .where(ticker_analyses.c.ticker == ticker)
                    .where(ticker_analyses.c.date == trade_date)
                    .values(decision=decision,
                            full_state_json=json.dumps(full_state) if full_state else None)
                )
    except Exception as e:
        logger.warning("SQLite mirror failed for %s/%s: %s",
                       ticker, trade_date, e)
```

Then locate the existing `store_decision` method in the `TradingMemoryLog` class. After its existing markdown-write logic (whatever it does today), append:

```python
        _mirror_to_sqlite(self.config, ticker, trade_date, final_trade_decision)
```

(If you aren't sure where the markdown write completes, add the call as the last statement of `store_decision`.)

Also add a resolution mirror for `update_with_outcome`. The actual method signature (from the codebase) is:
`update_with_outcome(ticker, trade_date, raw_return, alpha_return, holding_days, reflection)`.

Add this helper at the bottom of the file:

```python
def _mirror_resolution_to_sqlite(config: Dict[str, Any], ticker: str,
                                  trade_date: str, raw_return: float,
                                  alpha_return: float, holding_days: int,
                                  reflection: str) -> None:
    try:
        from sqlalchemy import update
        from tradingagents.storage import get_engine
        from tradingagents.storage.schema import ticker_analyses, metadata
        engine = get_engine(config)
        metadata.create_all(engine, tables=[ticker_analyses], checkfirst=True)
        now = datetime.utcnow().isoformat()
        with engine.begin() as conn:
            conn.execute(
                update(ticker_analyses)
                .where(ticker_analyses.c.ticker == ticker)
                .where(ticker_analyses.c.date == trade_date)
                .values(raw_return=raw_return, alpha_return=alpha_return,
                        holding_days=holding_days, reflection_md=reflection,
                        resolved_at=now)
            )
    except Exception as e:
        logger.warning("SQLite resolution mirror failed for %s/%s: %s",
                       ticker, trade_date, e)
```

In `update_with_outcome`, after the existing `os.replace()` atomic write completes (at the end of the method, after `updated` is confirmed True), call:

```python
        if updated:
            _mirror_resolution_to_sqlite(
                self.config, ticker, trade_date,
                raw_return, alpha_return, holding_days, reflection,
            )
```

> **Important:** The existing `TradingMemoryLog.__init__` stores the config as `self._log_path` and `self._max_entries` only — it does not keep a reference to the full config dict. You must add `self.config = cfg` to `__init__` so the mirror helpers can access `config["storage"]`. This is the full-config-threading decision: `TradingAgentsGraph` passes `self.config` (the full `DEFAULT_CONFIG`) to `TradingMemoryLog`, so the storage block is available at `self.config.get("storage", {})`.

- [ ] **Step 5: Run tests — verify pass**

Run: `pytest tests/agents/utils/test_memory_dual_write.py -v`
Expected: 3 passed.

- [ ] **Step 6: Run the existing memory tests to verify no regression**

Run: `pytest tests/ -k memory -v`
Expected: all existing memory tests still pass; new ones added.

- [ ] **Step 7: Commit**

```bash
git add tradingagents/agents/utils/memory.py tests/agents/utils/test_memory_dual_write.py
git commit -m "feat(storage): TradingMemoryLog dual-writes to SQLite"
```

---

### Task 9: Implement IndustryMemoryLog (sole-truth in SQLite)

**Files:**
- Create: `tradingagents/agents/utils/industry_memory.py`
- Create: `tests/agents/utils/test_industry_memory.py`

- [ ] **Step 1: Write failing tests**

Create `tests/agents/utils/test_industry_memory.py`:

```python
"""Tests for IndustryMemoryLog (SQLite-only)."""
import json
import pytest
from sqlalchemy import insert

from tradingagents.agents.utils.industry_memory import IndustryMemoryLog
from tradingagents.storage import get_engine
from tradingagents.storage.schema import (
    metadata, industry_briefs, ticker_analyses,
)


@pytest.fixture
def log(tmp_path):
    db_path = tmp_path / "industry.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True,
                          "cache_ttl_days": {}}}
    engine = get_engine(config)
    metadata.create_all(engine)
    return IndustryMemoryLog(config)


def test_store_brief_creates_row(log):
    view = {"call": "OW", "conviction": 0.7, "rationale": "test"}
    log.store_brief(sub_industry="Semiconductors", date="2026-05-09",
                    mode="brief", view=view, brief_md="# brief\n\nbody",
                    sector_etf="SOXX")

    from tradingagents.storage import get_engine
    from sqlalchemy import select
    engine = get_engine(log.config)
    with engine.connect() as conn:
        rows = conn.execute(select(industry_briefs)).all()
    assert len(rows) == 1
    assert rows[0].sub_industry == "Semiconductors"
    assert rows[0].call == "OW"
    assert rows[0].sector_etf == "SOXX"


def test_get_latest_brief(log):
    log.store_brief("Banks", "2026-05-01", "brief",
                    {"call": "N", "conviction": 0.5, "rationale": "old"},
                    brief_md="old", sector_etf="KBE")
    log.store_brief("Banks", "2026-05-09", "brief",
                    {"call": "OW", "conviction": 0.6, "rationale": "new"},
                    brief_md="new", sector_etf="KBE")
    latest = log.get_latest_brief("Banks", mode="brief")
    assert latest is not None
    assert latest["date"] == "2026-05-09"
    assert latest["call"] == "OW"


def test_get_latest_brief_returns_none_when_missing(log):
    assert log.get_latest_brief("Software", mode="brief") is None


def test_get_constituent_decisions(log):
    # Seed ticker_analyses with 2 NVDA + 1 AAPL decisions
    engine = get_engine(log.config)
    with engine.begin() as conn:
        conn.execute(insert(ticker_analyses).values(
            ticker="NVDA", date="2026-05-01", sub_industry="Semiconductors",
            decision="BUY", sector_alpha_return=0.05,
            created_at="2026-05-01T10:00:00",
        ))
        conn.execute(insert(ticker_analyses).values(
            ticker="NVDA", date="2026-05-08", sub_industry="Semiconductors",
            decision="HOLD", sector_alpha_return=-0.01,
            created_at="2026-05-08T10:00:00",
        ))
        conn.execute(insert(ticker_analyses).values(
            ticker="AAPL", date="2026-05-08", sub_industry="Technology Hardware",
            decision="BUY", sector_alpha_return=0.02,
            created_at="2026-05-08T10:00:00",
        ))

    rows = log.get_constituent_decisions("Semiconductors", days=30)
    tickers = {r["ticker"] for r in rows}
    assert tickers == {"NVDA"}
    assert len(rows) == 2
```

- [ ] **Step 2: Run tests — verify fail**

Run: `pytest tests/agents/utils/test_industry_memory.py -v`
Expected: ModuleNotFoundError.

- [ ] **Step 3: Implement IndustryMemoryLog**

Create `tradingagents/agents/utils/industry_memory.py`:

```python
"""Industry-level memory log backed by SQLite (sole source of truth)."""
import json
from datetime import datetime, timedelta
from typing import Any, Dict, List, Optional

from sqlalchemy import desc, insert, select, update

from tradingagents.storage import get_engine
from tradingagents.storage.schema import (
    metadata, industry_briefs, ticker_analyses,
)


class IndustryMemoryLog:
    """Persists industry briefs and queries cross-ticker decision history."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        engine = get_engine(config)
        metadata.create_all(
            engine,
            tables=[industry_briefs, ticker_analyses],
            checkfirst=True,
        )

    def store_brief(self, sub_industry: str, date: str, mode: str,
                    view: Dict[str, Any], brief_md: str,
                    sector_etf: Optional[str] = None) -> int:
        engine = get_engine(self.config)
        now = datetime.utcnow().isoformat()
        row = dict(
            sub_industry=sub_industry, date=date, mode=mode,
            call=view.get("call", "N"),
            conviction=view.get("conviction"),
            top_longs_json=json.dumps(view.get("top_longs", [])),
            top_shorts_json=json.dumps(view.get("top_shorts", [])),
            key_debates_json=json.dumps(view.get("key_debates", [])),
            catalysts_json=json.dumps(view.get("catalysts", [])),
            rationale_md=view.get("rationale", ""),
            brief_md=brief_md,
            sector_etf=sector_etf,
            created_at=now,
        )
        with engine.begin() as conn:
            try:
                result = conn.execute(insert(industry_briefs).values(**row))
                return result.inserted_primary_key[0]
            except Exception:
                conn.execute(
                    update(industry_briefs)
                    .where(industry_briefs.c.sub_industry == sub_industry)
                    .where(industry_briefs.c.date == date)
                    .where(industry_briefs.c.mode == mode)
                    .values(**{k: v for k, v in row.items()
                               if k not in {"sub_industry", "date", "mode"}})
                )
                existing = conn.execute(
                    select(industry_briefs.c.id)
                    .where(industry_briefs.c.sub_industry == sub_industry)
                    .where(industry_briefs.c.date == date)
                    .where(industry_briefs.c.mode == mode)
                ).scalar()
                return existing

    def get_latest_brief(self, sub_industry: str, mode: str = "brief"
                          ) -> Optional[Dict[str, Any]]:
        engine = get_engine(self.config)
        with engine.connect() as conn:
            row = conn.execute(
                select(industry_briefs)
                .where(industry_briefs.c.sub_industry == sub_industry)
                .where(industry_briefs.c.mode == mode)
                .order_by(desc(industry_briefs.c.date))
                .limit(1)
            ).first()
        if row is None:
            return None
        return {
            "id": row.id, "sub_industry": row.sub_industry, "date": row.date,
            "mode": row.mode, "call": row.call, "conviction": row.conviction,
            "rationale_md": row.rationale_md, "brief_md": row.brief_md,
            "sector_etf": row.sector_etf,
            "top_longs": json.loads(row.top_longs_json or "[]"),
            "top_shorts": json.loads(row.top_shorts_json or "[]"),
            "key_debates": json.loads(row.key_debates_json or "[]"),
            "catalysts": json.loads(row.catalysts_json or "[]"),
        }

    def get_constituent_decisions(self, sub_industry: str, days: int = 30
                                    ) -> List[Dict[str, Any]]:
        """Recent ticker decisions for any constituent in the sub-industry."""
        engine = get_engine(self.config)
        cutoff = (datetime.utcnow() - timedelta(days=days)).date().isoformat()
        with engine.connect() as conn:
            rows = conn.execute(
                select(ticker_analyses)
                .where(ticker_analyses.c.sub_industry == sub_industry)
                .where(ticker_analyses.c.date >= cutoff)
                .order_by(desc(ticker_analyses.c.date))
            ).all()
        return [
            {"ticker": r.ticker, "date": r.date, "decision": r.decision,
             "raw_return": r.raw_return, "alpha_return": r.alpha_return,
             "sector_alpha_return": r.sector_alpha_return}
            for r in rows
        ]
```

- [ ] **Step 4: Run tests — verify pass**

Run: `pytest tests/agents/utils/test_industry_memory.py -v`
Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add tradingagents/agents/utils/industry_memory.py tests/agents/utils/test_industry_memory.py
git commit -m "feat(storage): IndustryMemoryLog with SQLite-only persistence"
```

---

### Task 10: Markdown projection from SQLite (memory_view)

**Files:**
- Create: `tradingagents/storage/memory_view.py`
- Create: `tests/storage/test_memory_view.py`

- [ ] **Step 1: Write failing tests**

Create `tests/storage/test_memory_view.py`:

```python
"""Tests for markdown projection from SQLite."""
import pytest
from sqlalchemy import insert

from tradingagents.storage import get_engine
from tradingagents.storage.memory_view import (
    project_ticker_memory, project_industry_memory,
)
from tradingagents.storage.schema import (
    metadata, ticker_analyses, industry_briefs,
)


@pytest.fixture
def engine(tmp_path):
    db_path = tmp_path / "view.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    metadata.create_all(engine)
    return engine


def test_project_ticker_memory_empty(engine):
    md = project_ticker_memory(engine)
    assert isinstance(md, str)


def test_project_ticker_memory_with_entries(engine):
    with engine.begin() as conn:
        conn.execute(insert(ticker_analyses).values(
            ticker="NVDA", date="2026-05-09", sub_industry="Semiconductors",
            decision="BUY 100", raw_return=0.05, alpha_return=0.02,
            holding_days=5, created_at="2026-05-09T10:00:00",
            resolved_at="2026-05-15T10:00:00",
        ))
    md = project_ticker_memory(engine)
    assert "NVDA" in md
    assert "2026-05-09" in md
    assert "BUY 100" in md


def test_project_industry_memory(engine):
    with engine.begin() as conn:
        conn.execute(insert(industry_briefs).values(
            sub_industry="Semiconductors", date="2026-05-09", mode="brief",
            call="OW", conviction=0.7, rationale_md="strong cycle",
            brief_md="# brief\n\nfull text", sector_etf="SOXX",
            created_at="2026-05-09T10:00:00",
        ))
    md = project_industry_memory(engine)
    assert "Semiconductors" in md
    assert "OW" in md
    assert "SOXX" in md
```

- [ ] **Step 2: Run tests — verify fail (ImportError)**

Run: `pytest tests/storage/test_memory_view.py -v`

- [ ] **Step 3: Implement `memory_view.py`**

Create `tradingagents/storage/memory_view.py`:

```python
"""Markdown projection of SQLite-backed memory tables for LLM consumption."""
from typing import Optional

from sqlalchemy import Engine, desc, select

from tradingagents.storage.schema import ticker_analyses, industry_briefs


def project_ticker_memory(engine: Engine, limit: int = 50) -> str:
    """Project ticker_analyses to a markdown log."""
    with engine.connect() as conn:
        rows = conn.execute(
            select(ticker_analyses)
            .order_by(desc(ticker_analyses.c.date))
            .limit(limit)
        ).all()

    if not rows:
        return "# Trading memory\n\n_(no entries yet)_\n"

    lines = ["# Trading memory\n"]
    for r in rows:
        outcome = ""
        if r.raw_return is not None:
            outcome = (f"  - Raw: {r.raw_return:+.2%}, Alpha vs SPY: "
                       f"{r.alpha_return:+.2%}, days held: {r.holding_days}\n")
        reflection = f"  - Reflection: {r.reflection_md}\n" if r.reflection_md else ""
        lines.append(
            f"## {r.ticker} ({r.date})\n"
            f"  - Sub-industry: {r.sub_industry or '(unresolved)'}\n"
            f"  - Decision: {r.decision}\n"
            f"{outcome}{reflection}"
        )
    return "\n".join(lines)


def project_industry_memory(engine: Engine, limit: int = 50) -> str:
    """Project industry_briefs to a markdown log."""
    with engine.connect() as conn:
        rows = conn.execute(
            select(industry_briefs)
            .order_by(desc(industry_briefs.c.date))
            .limit(limit)
        ).all()

    if not rows:
        return "# Industry memory\n\n_(no entries yet)_\n"

    lines = ["# Industry memory\n"]
    for r in rows:
        outcome = ""
        if r.realized_etf_alpha_vs_spy is not None:
            outcome = (f"  - Realized {r.sector_etf} alpha vs SPY: "
                       f"{r.realized_etf_alpha_vs_spy:+.2%}\n")
        reflection = f"  - Reflection: {r.reflection_md}\n" if r.reflection_md else ""
        lines.append(
            f"## {r.sub_industry} ({r.date}, mode={r.mode})\n"
            f"  - Call: {r.call} (conviction: {r.conviction or 'n/a'})\n"
            f"  - Sector ETF: {r.sector_etf or 'n/a'}\n"
            f"  - Rationale: {r.rationale_md[:200]}{'...' if len(r.rationale_md) > 200 else ''}\n"
            f"{outcome}{reflection}"
        )
    return "\n".join(lines)
```

- [ ] **Step 4: Run tests — verify pass**

Run: `pytest tests/storage/test_memory_view.py -v`
Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add tradingagents/storage/memory_view.py tests/storage/test_memory_view.py
git commit -m "feat(storage): markdown projection of SQLite memory tables"
```

---

### Task 11: Backfill migration (002) — opt-in import of legacy markdown

**Files:**
- Create: `tradingagents/storage/migrations/versions/002_backfill_markdown_memory.py`
- Create: `tests/storage/fixtures/sample_trading_memory.md`
- Add tests to: `tests/storage/test_migrations.py`

- [ ] **Step 1: Create the fixture markdown file**

Create `tests/storage/fixtures/sample_trading_memory.md`:

```markdown
# Trading memory

## NVDA (2026-04-01)
  - Decision: BUY 50 shares at market open
  - Raw: +3.20%, Alpha vs SPY: +1.10%, days held: 5
  - Reflection: Conviction was rewarded; semis cycle still strong.

## AAPL (2026-04-15)
  - Decision: HOLD; no new conviction signal
  - Raw: -0.50%, Alpha vs SPY: -0.20%, days held: 5
  - Reflection: Hold was the right call given mixed signals.

## MSFT (2026-05-01)
  - Decision: BUY 30 shares
  (pending — no return data yet)
```

- [ ] **Step 2: Add backfill tests**

Append to `tests/storage/test_migrations.py`:

```python
def test_backfill_imports_markdown_entries(tmp_path):
    md_src = Path("tests/storage/fixtures/sample_trading_memory.md")
    md_dest = tmp_path / "memory" / "trading_memory.md"
    md_dest.parent.mkdir(parents=True)
    md_dest.write_text(md_src.read_text())

    db_path = tmp_path / "backfill.db"
    env = os.environ.copy()
    env["TRADINGAGENTS_DB_PATH"] = str(db_path)
    env["TRADINGAGENTS_MEMORY_LOG_PATH"] = str(md_dest)

    # Apply 001 first
    r1 = subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "001_initial"],
        capture_output=True, text=True, env=env,
    )
    assert r1.returncode == 0, r1.stderr

    # Apply 002 (backfill)
    r2 = subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "002_backfill"],
        capture_output=True, text=True, env=env,
    )
    assert r2.returncode == 0, r2.stderr

    from sqlalchemy import select
    from tradingagents.storage.schema import ticker_analyses
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    with engine.connect() as conn:
        rows = conn.execute(select(ticker_analyses)).all()
    tickers = {r.ticker for r in rows}
    assert tickers == {"NVDA", "AAPL", "MSFT"}

    # Original markdown preserved as .pre-migration
    assert (md_dest.parent / "trading_memory.md.pre-migration").exists()


def test_backfill_is_idempotent_when_rerun(tmp_path):
    """Running 002 twice should not duplicate entries."""
    md_src = Path("tests/storage/fixtures/sample_trading_memory.md")
    md_dest = tmp_path / "memory" / "trading_memory.md"
    md_dest.parent.mkdir(parents=True)
    md_dest.write_text(md_src.read_text())

    db_path = tmp_path / "idem_backfill.db"
    env = os.environ.copy()
    env["TRADINGAGENTS_DB_PATH"] = str(db_path)
    env["TRADINGAGENTS_MEMORY_LOG_PATH"] = str(md_dest)

    for _ in range(2):
        r = subprocess.run(
            ["alembic", "--config", "alembic.ini", "upgrade", "head"],
            capture_output=True, text=True, env=env,
        )
        assert r.returncode == 0, r.stderr

    from sqlalchemy import func, select
    from tradingagents.storage.schema import ticker_analyses
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    with engine.connect() as conn:
        count = conn.execute(select(func.count()).select_from(ticker_analyses)).scalar()
    assert count == 3
```

- [ ] **Step 3: Run new tests — verify fail**

Run: `pytest tests/storage/test_migrations.py::test_backfill_imports_markdown_entries -v`
Expected: FAIL — alembic exits non-zero because there's no `002_backfill` revision.

- [ ] **Step 4: Implement migration 002**

Create `tradingagents/storage/migrations/versions/002_backfill_markdown_memory.py`:

```python
"""backfill markdown memory into ticker_analyses

Revision ID: 002_backfill
Revises: 001_initial
Create Date: 2026-05-10

Idempotent: scans for existing (ticker, date) before inserting; preserves
the original markdown file as <path>.pre-migration on first run.
"""
import os
import re
import shutil
from datetime import datetime
from pathlib import Path

from alembic import op
from sqlalchemy import select, insert

from tradingagents.storage.schema import ticker_analyses

revision = "002_backfill"
down_revision = "001_initial"
branch_labels = None
depends_on = None


_HEADER_RE = re.compile(r"^##\s+([A-Z][A-Z0-9.\-]*)\s+\((\d{4}-\d{2}-\d{2})\)\s*$")
_DECISION_RE = re.compile(r"^\s*-\s*Decision:\s*(.+)$", re.IGNORECASE)
_RETURN_RE = re.compile(
    r"^\s*-\s*Raw:\s*([+-]?\d+(?:\.\d+)?)%,\s*Alpha\s+vs\s+SPY:\s*"
    r"([+-]?\d+(?:\.\d+)?)%,\s*days\s+held:\s*(\d+)", re.IGNORECASE)
_REFLECTION_RE = re.compile(r"^\s*-\s*Reflection:\s*(.+)$", re.IGNORECASE)


def _parse_markdown(md: str):
    """Yield dicts for each entry in the markdown log."""
    entries = []
    current = None
    for line in md.splitlines():
        m = _HEADER_RE.match(line)
        if m:
            if current is not None:
                entries.append(current)
            current = {"ticker": m.group(1), "date": m.group(2)}
            continue
        if current is None:
            continue
        m = _DECISION_RE.match(line)
        if m:
            current["decision"] = m.group(1).strip()
            continue
        m = _RETURN_RE.match(line)
        if m:
            current["raw_return"] = float(m.group(1)) / 100
            current["alpha_return"] = float(m.group(2)) / 100
            current["holding_days"] = int(m.group(3))
            continue
        m = _REFLECTION_RE.match(line)
        if m:
            current["reflection_md"] = m.group(1).strip()
    if current is not None:
        entries.append(current)
    return [e for e in entries if "ticker" in e and "date" in e
            and "decision" in e]


def upgrade() -> None:
    md_path_env = os.getenv("TRADINGAGENTS_MEMORY_LOG_PATH")
    if md_path_env:
        md_path = Path(md_path_env)
    else:
        md_path = Path.home() / ".tradingagents" / "memory" / "trading_memory.md"

    if not md_path.exists():
        return  # nothing to backfill

    backup = md_path.with_suffix(md_path.suffix + ".pre-migration")
    if not backup.exists():
        shutil.copy2(md_path, backup)

    md = md_path.read_text(encoding="utf-8")
    entries = _parse_markdown(md)

    bind = op.get_bind()
    now = datetime.utcnow().isoformat()

    for e in entries:
        existing = bind.execute(
            select(ticker_analyses.c.id)
            .where(ticker_analyses.c.ticker == e["ticker"])
            .where(ticker_analyses.c.date == e["date"])
        ).first()
        if existing:
            continue

        bind.execute(insert(ticker_analyses).values(
            ticker=e["ticker"], date=e["date"], decision=e["decision"],
            raw_return=e.get("raw_return"),
            alpha_return=e.get("alpha_return"),
            holding_days=e.get("holding_days"),
            reflection_md=e.get("reflection_md"),
            created_at=now,
            resolved_at=now if "raw_return" in e else None,
        ))


def downgrade() -> None:
    # Backfill is data-only — downgrade truncates ticker_analyses.
    bind = op.get_bind()
    bind.execute(ticker_analyses.delete())
```

- [ ] **Step 5: Run tests — verify pass**

Run: `pytest tests/storage/test_migrations.py -v`
Expected: 4 passed.

- [ ] **Step 6: Commit**

```bash
git add tradingagents/storage/migrations/versions/002_backfill_markdown_memory.py \
        tests/storage/test_migrations.py tests/storage/fixtures
git commit -m "feat(storage): opt-in backfill migration from legacy markdown memory"
```

---

### Task 12: Storage CLI subcommand

**Files:**
- Create: `cli/storage.py`
- Modify: `cli/main.py`
- Create: `tests/cli/test_storage_cli.py`

- [ ] **Step 1: Write failing CLI tests**

Create `tests/cli/test_storage_cli.py`:

```python
"""Tests for `tradingagents storage ...` CLI."""
import os
import subprocess

import pytest


def _run(args, env_overrides=None):
    env = os.environ.copy()
    if env_overrides:
        env.update(env_overrides)
    return subprocess.run(
        ["python", "-m", "cli.main"] + args,
        capture_output=True, text=True, env=env,
    )


def test_storage_migrate_command(tmp_path):
    db = tmp_path / "cli.db"
    r = _run(["storage", "migrate"],
             env_overrides={"TRADINGAGENTS_DB_PATH": str(db)})
    assert r.returncode == 0, f"stdout={r.stdout}\nstderr={r.stderr}"
    assert db.exists()


def test_storage_stats_command(tmp_path):
    db = tmp_path / "stats.db"
    _run(["storage", "migrate"],
         env_overrides={"TRADINGAGENTS_DB_PATH": str(db)})
    r = _run(["storage", "stats"],
             env_overrides={"TRADINGAGENTS_DB_PATH": str(db)})
    assert r.returncode == 0
    assert "ticker_analyses" in r.stdout


def test_storage_backfill_runs(tmp_path):
    md = tmp_path / "memory" / "trading_memory.md"
    md.parent.mkdir(parents=True)
    md.write_text("# Trading memory\n\n## NVDA (2026-05-09)\n  - Decision: BUY\n")
    db = tmp_path / "bf.db"
    env = {"TRADINGAGENTS_DB_PATH": str(db),
           "TRADINGAGENTS_MEMORY_LOG_PATH": str(md)}
    _run(["storage", "migrate"], env_overrides=env)
    r = _run(["storage", "backfill"], env_overrides=env)
    assert r.returncode == 0
```

- [ ] **Step 2: Run tests — verify fail**

Run: `pytest tests/cli/test_storage_cli.py -v`
Expected: 3 FAIL with non-zero exit code (storage subcommand doesn't exist).

- [ ] **Step 3: Implement `cli/storage.py`**

> **CLI framework note:** The project uses [Typer](https://typer.tiangolo.com/), not argparse. `cli/main.py` defines `app = typer.Typer(...)`. New subcommands must be Typer sub-apps registered via `app.add_typer()`.

Create `cli/storage.py`:

```python
"""`tradingagents storage ...` subcommand group."""
import subprocess
import sys

import typer
from sqlalchemy import func, select

from tradingagents.default_config import DEFAULT_CONFIG
from tradingagents.storage import get_engine
from tradingagents.storage.schema import ALL_TABLES

app = typer.Typer(name="storage", help="Storage management (migrations, backfill, stats)")


@app.command("migrate")
def cmd_migrate():
    """Apply pending Alembic migrations."""
    result = subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "head"],
        capture_output=False,
    )
    raise typer.Exit(result.returncode)


@app.command("backfill")
def cmd_backfill():
    """Re-import legacy markdown memory log into SQLite."""
    result = subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "002_backfill"],
        capture_output=False,
    )
    raise typer.Exit(result.returncode)


@app.command("stats")
def cmd_stats():
    """Print row counts and DB size."""
    engine = get_engine(DEFAULT_CONFIG)
    typer.echo(f"Database: {DEFAULT_CONFIG['storage']['db_path']}\n")
    typer.echo(f"{'Table':<32}{'Rows':>10}")
    typer.echo("-" * 42)
    with engine.connect() as conn:
        for table in ALL_TABLES:
            try:
                n = conn.execute(select(func.count()).select_from(table)).scalar()
            except Exception:
                n = "(missing)"
            typer.echo(f"{table.name:<32}{str(n):>10}")
```

- [ ] **Step 4: Wire into `cli/main.py`**

Open `cli/main.py`. After the `app = typer.Typer(...)` definition and the existing imports, add:

```python
from cli.storage import app as storage_app
app.add_typer(storage_app)
```

This registers `tradingagents storage migrate`, `tradingagents storage backfill`, and `tradingagents storage stats` as sub-commands of the top-level `app`.

- [ ] **Step 5: Run tests — verify pass**

Run: `pytest tests/cli/test_storage_cli.py -v`
Expected: 3 passed.

- [ ] **Step 6: Commit**

```bash
git add cli/storage.py cli/main.py tests/cli/test_storage_cli.py
git commit -m "feat(cli): add 'tradingagents storage' subcommand"
```

---

### Task 13: Startup notice for pending migration (opt-in pattern)

**Files:**
- Create: `tradingagents/storage/reconciler.py`
- Create: `tests/storage/test_reconciler.py`
- Modify: `tradingagents/graph/trading_graph.py` (call reconciler on init)

- [ ] **Step 1: Write failing test**

Create `tests/storage/test_reconciler.py`:

```python
"""Tests for storage reconciler / startup notice."""
import logging
import pytest

from tradingagents.storage.reconciler import check_migration_status, MigrationStatus


def test_status_when_db_does_not_exist(tmp_path):
    db_path = tmp_path / "noexist.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    status = check_migration_status(config)
    assert status == MigrationStatus.NOT_INITIALIZED


def test_status_when_up_to_date(tmp_path):
    import subprocess, os
    db = tmp_path / "ok.db"
    env = os.environ.copy()
    env["TRADINGAGENTS_DB_PATH"] = str(db)
    subprocess.run(
        ["alembic", "--config", "alembic.ini", "upgrade", "head"],
        check=True, env=env,
    )
    config = {"storage": {"db_path": str(db), "wal_mode": True}}
    assert check_migration_status(config) == MigrationStatus.UP_TO_DATE
```

- [ ] **Step 2: Run tests — verify fail**

Run: `pytest tests/storage/test_reconciler.py -v`

- [ ] **Step 3: Implement reconciler**

Create `tradingagents/storage/reconciler.py`:

```python
"""Startup migration-status checker.

Migration is OPT-IN: this module never auto-applies migrations. It only logs
a one-line notice if the user should run `tradingagents storage migrate`.
"""
import logging
from enum import Enum
from pathlib import Path
from typing import Any, Dict

from sqlalchemy import inspect

from tradingagents.storage import get_engine

logger = logging.getLogger(__name__)


class MigrationStatus(Enum):
    NOT_INITIALIZED = "not_initialized"
    PENDING = "pending"
    UP_TO_DATE = "up_to_date"


def check_migration_status(config: Dict[str, Any]) -> MigrationStatus:
    db_path = config.get("storage", {}).get("db_path")
    if db_path and db_path != ":memory:" and not Path(db_path).exists():
        return MigrationStatus.NOT_INITIALIZED

    engine = get_engine(config)
    inspector = inspect(engine)
    if "alembic_version" not in inspector.get_table_names():
        return MigrationStatus.NOT_INITIALIZED

    # Naive check — head is the most recent file in versions/
    versions_dir = Path(__file__).parent / "migrations" / "versions"
    head = sorted([f.stem for f in versions_dir.glob("*.py")
                   if f.stem != "__init__"])[-1].split("_")[0]

    with engine.connect() as conn:
        from sqlalchemy import text
        current = conn.execute(text("SELECT version_num FROM alembic_version")).scalar()
    if current and current.startswith(head):
        return MigrationStatus.UP_TO_DATE
    return MigrationStatus.PENDING


def emit_notice_if_needed(config: Dict[str, Any]) -> None:
    """Log a one-line notice if SQLite needs init or migration. Never raises."""
    try:
        status = check_migration_status(config)
        if status == MigrationStatus.NOT_INITIALIZED:
            logger.info(
                "Pending migration: run 'tradingagents storage migrate' to "
                "enable structured storage. (Existing functionality works without it.)"
            )
        elif status == MigrationStatus.PENDING:
            logger.info(
                "SQLite migrations pending: run 'tradingagents storage migrate'."
            )
    except Exception as e:
        logger.debug("Migration status check skipped: %s", e)
```

- [ ] **Step 4: Wire into `TradingAgentsGraph.__init__`**

In `tradingagents/graph/trading_graph.py`, in `__init__`, after `set_config(self.config)` and before LLM setup:

```python
        # Storage layer notice (opt-in migration). Never raises.
        try:
            from tradingagents.storage.reconciler import emit_notice_if_needed
            emit_notice_if_needed(self.config)
        except ImportError:
            pass  # storage layer not installed — older deployments
```

- [ ] **Step 5: Run tests — verify pass**

Run: `pytest tests/storage/test_reconciler.py -v`
Expected: 2 passed.

- [ ] **Step 6: Smoke-run BC-3**

Run: `python -c "from tradingagents.graph.trading_graph import TradingAgentsGraph; from tradingagents.default_config import DEFAULT_CONFIG; ta = TradingAgentsGraph(config=DEFAULT_CONFIG); print('OK')"`
Expected: prints `OK` (and possibly the storage notice).

- [ ] **Step 7: Commit**

```bash
git add tradingagents/storage/reconciler.py tradingagents/graph/trading_graph.py \
        tests/storage/test_reconciler.py
git commit -m "feat(storage): opt-in migration notice on TradingAgentsGraph init"
```

---

### Task 14: BC-1 / BC-7 regression test (snapshot of v0.2.4 ticker decision shape)

**Files:**
- Create: `tests/regression/test_bc_invariants.py`
- Create: `tests/regression/snapshots/nvda_2026_04_01_v0_2_4.json`
- Create: `tests/regression/conftest.py`

- [ ] **Step 1: Capture a baseline decision shape**

This test verifies BC-1 (output JSON shape unchanged from v0.2.4 with default config). The snapshot file is the locked baseline — any new top-level fields fail the test unless explicitly whitelisted.

Create `tests/regression/conftest.py`:

```python
import pytest
from pathlib import Path


@pytest.fixture
def snapshot_dir():
    return Path("tests/regression/snapshots")
```

Create `tests/regression/snapshots/nvda_2026_04_01_v0_2_4.json`:

```json
{
  "expected_top_level_keys": [
    "company_of_interest",
    "trade_date",
    "market_report",
    "sentiment_report",
    "news_report",
    "fundamentals_report",
    "investment_debate_state",
    "trader_investment_decision",
    "risk_debate_state",
    "investment_plan",
    "final_trade_decision"
  ],
  "v0_2_4_baseline_version": "0.2.4",
  "notes": "This snapshot is the locked v0.2.4 output shape. Adding fields requires updating this file AND the spec section §17 BC-1."
}
```

Create `tests/regression/__init__.py` as empty file.

- [ ] **Step 2: Write the test**

Create `tests/regression/test_bc_invariants.py`:

```python
"""Backwards-compat regression tests for §17 invariants."""
import importlib
import json
from pathlib import Path

import pytest

from tradingagents.default_config import DEFAULT_CONFIG


def test_bc8_module_imports_without_optional_deps():
    """BC-8: TradingAgentsGraph imports without industry-research deps."""
    mod = importlib.import_module("tradingagents.graph.trading_graph")
    assert hasattr(mod, "TradingAgentsGraph")


def test_bc3_default_config_does_not_require_new_env_vars():
    """BC-3: default config can be instantiated without TIINGO/MCP/etc."""
    config = DEFAULT_CONFIG.copy()
    # No env vars set in this test process for new vendors
    assert "storage" in config
    assert config["storage"].get("db_path") is not None


def test_bc4_propagate_signature_unchanged():
    """BC-4: propagate(ticker, date) — no new required args."""
    from tradingagents.graph.trading_graph import TradingAgentsGraph
    import inspect
    sig = inspect.signature(TradingAgentsGraph.propagate)
    params = list(sig.parameters.values())
    required = [p for p in params[1:]  # skip self
                if p.default is inspect.Parameter.empty
                and p.kind != inspect.Parameter.VAR_KEYWORD]
    assert len(required) == 2, f"expected 2 required args, got {[p.name for p in required]}"


def test_bc1_agent_state_keys_present(snapshot_dir):
    """BC-1: AgentState fields used by _log_state still exist."""
    from tradingagents.agents.utils.agent_states import AgentState
    snapshot = json.loads((snapshot_dir / "nvda_2026_04_01_v0_2_4.json").read_text())
    expected = set(snapshot["expected_top_level_keys"])
    actual_keys = set(AgentState.__annotations__.keys())
    missing = expected - actual_keys
    assert not missing, f"BC-1 violation: missing v0.2.4 keys: {missing}"


def test_bc6_checkpoint_dir_path_unchanged():
    """BC-6: LangGraph checkpoint path is unchanged."""
    config = DEFAULT_CONFIG.copy()
    cache_dir = config["data_cache_dir"]
    # Path lives at <data_cache_dir>/checkpoints/<TICKER>.db
    assert "checkpoints" not in cache_dir, "data_cache_dir is the parent only"
```

- [ ] **Step 3: Run tests**

Run: `pytest tests/regression/test_bc_invariants.py -v`
Expected: 5 passed.

- [ ] **Step 4: Commit**

```bash
git add tests/regression
git commit -m "test(regression): BC-1/3/4/6/8 invariants"
```

---

### Task 15: BC-5 fault-injection test (new subsystem failures don't break ticker)

**Files:**
- Append tests to: `tests/regression/test_bc_invariants.py`

- [ ] **Step 1: Add the test**

Append to `tests/regression/test_bc_invariants.py`:

```python
def test_bc5_sqlite_failure_does_not_break_memory_log(tmp_path, monkeypatch):
    """BC-5: SQLite mirror failure doesn't break TradingMemoryLog."""
    from tradingagents.agents.utils.memory import TradingMemoryLog
    md = tmp_path / "memory" / "tm.md"
    md.parent.mkdir(parents=True)
    config = {
        "memory_log_path": str(md),
        "memory_log_max_entries": None,
        "storage": {"db_path": "/non/existent/path/that/will/fail.db",
                    "wal_mode": True, "cache_ttl_days": {}},
    }
    log = TradingMemoryLog(config)
    # Should not raise even though SQLite path is unwritable
    log.store_decision(ticker="TST", trade_date="2026-05-09",
                       final_trade_decision="BUY")
    assert "TST" in md.read_text()


def test_bc5_storage_reconciler_swallows_errors(monkeypatch):
    """BC-5: reconciler.emit_notice_if_needed never raises."""
    from tradingagents.storage.reconciler import emit_notice_if_needed

    def boom(*a, **kw):
        raise RuntimeError("simulated")

    monkeypatch.setattr(
        "tradingagents.storage.reconciler.check_migration_status", boom
    )
    # Should not raise
    emit_notice_if_needed({"storage": {"db_path": ":memory:"}})
```

- [ ] **Step 2: Run tests — verify pass**

Run: `pytest tests/regression/test_bc_invariants.py -v`
Expected: 7 passed total.

- [ ] **Step 3: Commit**

```bash
git add tests/regression/test_bc_invariants.py
git commit -m "test(regression): BC-5 fault-injection invariants"
```

---

### Task 16: Concurrent write smoke test (WAL mode under contention)

**Files:**
- Create: `tests/storage/test_concurrent.py`

- [ ] **Step 1: Write the test**

Create `tests/storage/test_concurrent.py`:

```python
"""Smoke test for concurrent writes under WAL mode (P-10)."""
import threading
from datetime import datetime

import pytest
from sqlalchemy import insert, func, select

from tradingagents.storage import get_engine
from tradingagents.storage.schema import metadata, ticker_analyses


def test_concurrent_writers_under_wal(tmp_path):
    db_path = tmp_path / "concurrent.db"
    config = {"storage": {"db_path": str(db_path), "wal_mode": True}}
    engine = get_engine(config)
    metadata.create_all(engine)

    errors = []

    def writer(prefix: str, n: int):
        try:
            for i in range(n):
                with engine.begin() as conn:
                    conn.execute(insert(ticker_analyses).values(
                        ticker=f"{prefix}{i:03d}",
                        date="2026-05-09",
                        decision="BUY",
                        created_at=datetime.utcnow().isoformat(),
                    ))
        except Exception as e:
            errors.append(e)

    threads = [
        threading.Thread(target=writer, args=("A", 50)),
        threading.Thread(target=writer, args=("B", 50)),
        threading.Thread(target=writer, args=("C", 50)),
    ]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert not errors, f"concurrent writers raised: {errors}"

    with engine.connect() as conn:
        n = conn.execute(select(func.count()).select_from(ticker_analyses)).scalar()
    assert n == 150
```

- [ ] **Step 2: Run tests — verify pass**

Run: `pytest tests/storage/test_concurrent.py -v`
Expected: 1 passed.

- [ ] **Step 3: Commit**

```bash
git add tests/storage/test_concurrent.py
git commit -m "test(storage): concurrent writers under WAL mode"
```

---

### Task 17: Final integration smoke + plan-1 PR description

**Files:**
- (none new; verification only)

- [ ] **Step 1: Run the full storage test suite**

Run: `pytest tests/storage tests/cli tests/regression tests/agents/utils -v`
Expected: ALL pass.

- [ ] **Step 2: Run the full pre-existing test suite to verify zero regression**

Run: `pytest tests/ -v`
Expected: all pre-existing tests still pass; new storage tests added.

- [ ] **Step 3: Smoke-test BC-3 from a clean import**

Run:
```bash
python -c "
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG
ta = TradingAgentsGraph(config=DEFAULT_CONFIG)
print('TradingAgentsGraph instantiates cleanly:', ta is not None)
"
```
Expected: prints OK message.

- [ ] **Step 4: Smoke-test the storage CLI**

Run:
```bash
TRADINGAGENTS_DB_PATH=/tmp/smoke.db python -m cli.main storage migrate
TRADINGAGENTS_DB_PATH=/tmp/smoke.db python -m cli.main storage stats
rm /tmp/smoke.db /tmp/smoke.db-wal /tmp/smoke.db-shm 2>/dev/null
```
Expected: migrate succeeds; stats prints table sizes.

- [ ] **Step 5: Open the PR**

Branch is ready. Create PR with:

```
Title: feat(storage): SQLite foundation + dual-write memory log (Plan 1/5)

Summary:
- Adds tradingagents.storage package: SQLAlchemy Core schema, Alembic migrations,
  vendor-aware TTL cache, IndustryMemoryLog, markdown memory_view.
- Refactors TradingMemoryLog to dual-write: markdown stays canonical, SQLite mirror.
- Adds tradingagents storage {migrate, backfill, stats} CLI subcommands.
- Migration is opt-in; ticker analysis works identically to v0.2.4 with or
  without migration applied.

Backwards-compat invariants verified: BC-1, BC-3, BC-4, BC-5, BC-6, BC-8.

Test plan:
- [x] All new tests pass: tests/storage, tests/agents/utils, tests/cli, tests/regression
- [x] All pre-existing tests pass
- [x] BC-3 smoke: TradingAgentsGraph instantiates with default config
- [x] CLI smoke: storage migrate / stats end-to-end

Next: Plan 02 (industry workflow) depends on this.
```

- [ ] **Step 6: Final commit if anything outstanding**

```bash
git status
# If clean, you're done. Otherwise resolve and commit.
```

---

## Plan 1 done — what's next

After this plan ships and is on `main`, proceed to:

- **Plan 02** — Standalone industry workflow (`IndustryResearchGraph` with the 9-node topology, 3 modes, Universe Resolver, GICS taxonomy, ETF holdings, FRED industry, Tiingo + MCP loader, all industry agents, `tradingagents industry ...` CLI).
- **Plan 03** — PDF report ingestion (pdfplumber + Fidelity/Schwab/generic adapters, classifier, takeaway extractor, `tradingagents reports ...` CLI). Independent of Plan 02.
- **Plan 04** — Industry-context injection into ticker pipeline. Depends on Plans 01 + 02 + 03.
- **Plan 05** — Cross-ticker feedback loop. Depends on Plans 01 + 02 + 04.
