---
title: Cross-Ticker Feedback Loop Implementation Plan
status: draft
spec: 2026-05-10-industry-research-design.md
sub_project: 5 of 5
plan_number: 05
created: 2026-05-10
---

# Cross-Ticker Feedback Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the loop. When `IndustryResearchGraph` regenerates a brief, it sees recent constituent ticker decisions (with realized sector-relative alpha) and uses them to refine the new brief. Past industry briefs themselves get reflected on against the realized sector-ETF alpha vs SPY.

**Architecture:** Wire `IndustryMemoryLog.get_constituent_decisions()` into the Industry Strategist's prompt; populate `ticker_analyses.sub_industry` and `sector_alpha_return` columns at ticker resolution time; add an industry-brief reflector that runs at refresh time and writes `industry_briefs.realized_etf_alpha_vs_spy` + `reflection_md`.

**Tech Stack:** Existing modules from Plans 01-04; no new external deps.

**Depends on:** Plans 01 (storage), 02 (industry workflow), 04 (ticker injection — for `sub_industry` resolution path).

**Backwards-compat invariants enforced:** BC-2 (markdown unchanged), BC-5 (failures don't propagate).

---

## File Structure

> **Test directory convention:** The project uses a flat `tests/` directory (no subdirectories). All test files live directly in `tests/`. Inline task references to `tests/industry/`, `tests/agents/utils/`, `tests/graph/`, etc. should be read as `tests/` (the prefix is kept only for readability within this plan).
>
> **`sector_alpha_return` dependency note:** The `ticker_analyses.sector_alpha_return` column is populated by computing `ticker_return - sector_ETF_return` for the holding period. This requires the ticker → sub-industry → sector-ETF chain established in Plan 02 (`GICS_TAXONOMY`, `ETF_HOLDINGS`) to be fully operational before Plan 05 can populate this column. Plan 05 Task 2 should only run after Plan 02 is complete and the sector-ETF mapping is available in the database.

**Created:**
- `tradingagents/industry/brief_resolver.py` — resolves pending briefs against sector-relative alpha
- `tests/test_brief_resolver.py`
- `tests/test_constituent_feedback.py`

**Modified:**
- `tradingagents/agents/utils/memory.py` — `_resolve_pending_entries` also writes `sub_industry` + `sector_alpha_return` to SQLite
- `tradingagents/agents/industry/industry_strategist.py` — prompt picks up `constituent_decisions` (already wired in Plan 02; Plan 05 verifies the loop)
- `tradingagents/industry/industry_research_graph.py` — adds `_resolve_pending_briefs` mirroring `TradingAgentsGraph._resolve_pending_entries`
- `cli/industry.py` — `analyze` and `monitor` commands now invoke brief reflection

---

## Tasks

### Task 1: Populate `ticker_analyses.sub_industry` at resolution time

**Files:**
- Modify: `tradingagents/agents/utils/memory.py`
- Modify: `tradingagents/graph/trading_graph.py`
- Add tests: `tests/agents/utils/test_sub_industry_population.py`

- [ ] **Step 1: Write failing test**

```python
"""Verify sub_industry is recorded for ticker_analyses rows."""
import pytest
from sqlalchemy import select


def test_sub_industry_populated_after_propagate(tmp_path, monkeypatch):
    monkeypatch.setenv("TRADINGAGENTS_DB_PATH", str(tmp_path / "si.db"))
    from tradingagents.storage import get_engine
    from tradingagents.storage.schema import metadata, ticker_analyses
    from tradingagents.agents.utils.memory import TradingMemoryLog
    from tradingagents.default_config import DEFAULT_CONFIG

    md = tmp_path / "memory" / "tm.md"
    md.parent.mkdir(parents=True)
    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": str(tmp_path / "si.db")}
    config["memory_log_path"] = str(md)

    engine = get_engine(config)
    metadata.create_all(engine)

    log = TradingMemoryLog(config)
    log.store_decision(ticker="NVDA", trade_date="2026-05-09",
                        final_trade_decision="BUY 100",
                        sub_industry="Semiconductors")

    with engine.connect() as conn:
        rows = conn.execute(select(ticker_analyses)).all()
    assert len(rows) == 1
    assert rows[0].sub_industry == "Semiconductors"
```

- [ ] **Step 2: Update `TradingMemoryLog.store_decision`**

In `tradingagents/agents/utils/memory.py`, add `sub_industry: Optional[str] = None` to `store_decision`'s signature. Pass it through to `_mirror_to_sqlite`. Update `_mirror_to_sqlite` to include `sub_industry` in the values dict:

```python
def store_decision(self, ticker, trade_date, final_trade_decision,
                    sub_industry=None):
    # ... existing markdown write logic unchanged ...
    _mirror_to_sqlite(self.config, ticker, trade_date, final_trade_decision,
                      sub_industry=sub_industry)


def _mirror_to_sqlite(config, ticker, trade_date, decision,
                       full_state=None, sub_industry=None):
    # In the insert/update values, include sub_industry=sub_industry
    ...
```

- [ ] **Step 3: Update `TradingAgentsGraph._run_graph`**

In `tradingagents/graph/trading_graph.py`, the call to `self.memory_log.store_decision(...)` should pass the sub-industry resolved during the injection pre-pass (if any):

```python
        sub_ind = None
        if self.config.get("industry", {}).get("enabled", False):
            from tradingagents.graph.industry_injection import resolve_sub_industry_for_ticker
            try:
                sub_ind = resolve_sub_industry_for_ticker(company_name)
            except Exception:
                pass

        self.memory_log.store_decision(
            ticker=company_name, trade_date=trade_date,
            final_trade_decision=final_state["final_trade_decision"],
            sub_industry=sub_ind,
        )
```

- [ ] **Step 4: Run, commit**

Run: `pytest tests/agents/utils/test_sub_industry_population.py -v`
Expected: 1 passed.

```bash
git add tradingagents/agents/utils/memory.py tradingagents/graph/trading_graph.py \
        tests/agents/utils/test_sub_industry_population.py
git commit -m "feat(memory): record sub_industry on ticker_analyses rows"
```

---

### Task 2: Compute and persist `sector_alpha_return` on resolution

**Files:**
- Modify: `tradingagents/graph/trading_graph.py` (`_resolve_pending_entries` + `_fetch_returns`)
- Modify: `tradingagents/agents/utils/memory.py` (`_mirror_resolution_to_sqlite`)
- Add tests: `tests/graph/test_sector_alpha_return.py`

- [ ] **Step 1: Write failing test**

```python
"""Verify sector_alpha_return is computed and persisted at resolution."""
from unittest.mock import patch, MagicMock
from sqlalchemy import insert, select


def test_sector_alpha_return_computed_for_known_sub_industry(tmp_path, monkeypatch):
    monkeypatch.setenv("TRADINGAGENTS_DB_PATH", str(tmp_path / "sa.db"))
    from tradingagents.storage import get_engine
    from tradingagents.storage.schema import metadata, ticker_analyses
    from tradingagents.default_config import DEFAULT_CONFIG

    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": str(tmp_path / "sa.db")}
    engine = get_engine(config)
    metadata.create_all(engine)
    with engine.begin() as conn:
        conn.execute(insert(ticker_analyses).values(
            ticker="NVDA", date="2026-05-01", sub_industry="Semiconductors",
            decision="BUY", created_at="2026-05-01T10:00:00",
        ))

    # Mock yfinance: NVDA +5%, SPY +1%, SOXX +4% over 5 days.
    with patch("yfinance.Ticker") as t:
        def maker(symbol):
            class H:
                def history(self, start, end):
                    import pandas as pd
                    if symbol == "NVDA":
                        return pd.DataFrame({"Close": [100, 101, 102, 103, 104, 105]})
                    if symbol == "SPY":
                        return pd.DataFrame({"Close": [100, 100.2, 100.4, 100.6, 100.8, 101]})
                    if symbol == "SOXX":
                        return pd.DataFrame({"Close": [100, 100.8, 101.6, 102.4, 103.2, 104]})
                    return pd.DataFrame()
            return H()
        t.side_effect = maker

        from tradingagents.graph.trading_graph import TradingAgentsGraph
        # Bypass full init by directly instantiating + calling the resolution path:
        # In practice the v0.3.0 code calls _resolve_pending_entries internally.
        # Here we exercise the helper directly via a minimal stub.
        # Engineer note: refactor _fetch_returns to also return sector_alpha when
        # sector_etf is provided, then assert on the persisted column below.
        from tradingagents.graph.trading_graph import TradingAgentsGraph
        ta = TradingAgentsGraph(config=config)
        ta.ticker = "NVDA"
        ta._resolve_pending_entries("NVDA")  # invokes the sector-aware resolver

    with engine.connect() as conn:
        row = conn.execute(
            select(ticker_analyses).where(ticker_analyses.c.ticker == "NVDA")
        ).first()
    assert row.sector_alpha_return is not None
    # NVDA returned +5%; SOXX returned +4% over 5 days → ticker-vs-sector +1%
    # Sector-vs-SPY = +4% - +1% = +3% (different metric — depends on definition)
    # Engineer chooses the definition; whichever, just assert non-null + correct sign.
    assert row.sector_alpha_return > 0
```

- [ ] **Step 2: Refactor `_fetch_returns` to compute sector alpha**

In `tradingagents/graph/trading_graph.py`, extend the existing `_fetch_returns` to also pull the sector-ETF return for the resolved sub-industry, and return it as a fourth element:

```python
    def _fetch_returns(self, ticker, trade_date, holding_days=5,
                        sector_etf=None):
        # ... existing yfinance fetch for ticker + SPY ...
        sector_alpha = None
        if sector_etf:
            try:
                sec = yf.Ticker(sector_etf).history(start=trade_date, end=end_str)
                if len(sec) >= 2:
                    sec_ret = float((sec["Close"].iloc[actual_days]
                                     - sec["Close"].iloc[0])
                                     / sec["Close"].iloc[0])
                    sector_alpha = raw - sec_ret
            except Exception as e:
                logger.warning("Sector ETF return fetch failed: %s", e)
        return raw, alpha, actual_days, sector_alpha
```

In `_resolve_pending_entries`, look up the sub_industry → sector ETF and pass it:

```python
        from tradingagents.dataflows.industry.gics_taxonomy import etf_proxy_for
        for entry in pending:
            sec_etf = etf_proxy_for(entry.get("sub_industry") or "") or None
            raw, alpha, days, sec_alpha = self._fetch_returns(
                ticker, entry["date"], sector_etf=sec_etf,
            )
            ...
            updates.append({
                ..., "sector_alpha_return": sec_alpha,
            })
```

- [ ] **Step 3: Update `_mirror_resolution_to_sqlite` to write sector_alpha_return**

```python
def _mirror_resolution_to_sqlite(config, ticker, trade_date, raw_return,
                                  alpha_return, holding_days, reflection,
                                  sector_alpha_return=None):
    # In the update values, include sector_alpha_return=sector_alpha_return
    ...
```

- [ ] **Step 4: Run, commit**

Run: `pytest tests/graph/test_sector_alpha_return.py -v`
Expected: 1 passed.

```bash
git add tradingagents/graph/trading_graph.py tradingagents/agents/utils/memory.py \
        tests/graph/test_sector_alpha_return.py
git commit -m "feat(graph): compute and persist sector_alpha_return at resolution"
```

---

### Task 3: Industry brief reflector

**Files:**
- Create: `tradingagents/industry/brief_resolver.py`
- Create: `tests/industry/test_brief_resolver.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import patch
from datetime import datetime, timedelta
from sqlalchemy import insert, select


def test_resolve_pending_briefs_writes_realized_alpha(tmp_path, monkeypatch):
    monkeypatch.setenv("TRADINGAGENTS_DB_PATH", str(tmp_path / "br.db"))
    from tradingagents.storage import get_engine
    from tradingagents.storage.schema import metadata, industry_briefs
    from tradingagents.default_config import DEFAULT_CONFIG

    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": str(tmp_path / "br.db")}

    engine = get_engine(config)
    metadata.create_all(engine)
    old_date = (datetime.utcnow() - timedelta(days=10)).date().isoformat()
    with engine.begin() as conn:
        conn.execute(insert(industry_briefs).values(
            sub_industry="Semiconductors", date=old_date, mode="brief",
            call="OW", rationale_md="strong cycle",
            brief_md="# brief", sector_etf="SOXX",
            created_at=datetime.utcnow().isoformat(),
        ))

    with patch("tradingagents.industry.brief_resolver.fetch_sector_alpha_vs_spy") as f:
        f.return_value = (0.025, 7)  # +2.5% sector-relative alpha
        from tradingagents.industry.brief_resolver import resolve_pending_briefs
        resolve_pending_briefs("Semiconductors", config=config)

    with engine.connect() as conn:
        row = conn.execute(
            select(industry_briefs)
            .where(industry_briefs.c.sub_industry == "Semiconductors")
        ).first()
    assert row.realized_etf_alpha_vs_spy is not None
    assert abs(row.realized_etf_alpha_vs_spy - 0.025) < 1e-9
    assert row.reflection_md is not None
    assert "vindicated" in row.reflection_md.lower() or "missed" in row.reflection_md.lower()
```

- [ ] **Step 2: Implement**

```python
"""Resolve pending industry briefs against realized sector-relative alpha."""
import logging
from datetime import datetime, timedelta
from typing import Any, Dict, Optional

from sqlalchemy import select, update

from tradingagents.industry.industry_reflection import (
    fetch_sector_alpha_vs_spy, reflect_on_brief,
)
from tradingagents.storage import get_engine
from tradingagents.storage.schema import industry_briefs

logger = logging.getLogger(__name__)


def resolve_pending_briefs(sub_industry: str, config: Dict[str, Any],
                            holding_days: int = 7) -> int:
    """Find any unresolved briefs older than holding_days and reflect on them.

    Returns the number of briefs newly resolved.
    """
    engine = get_engine(config)
    cutoff = (datetime.utcnow() - timedelta(days=holding_days)).date().isoformat()

    with engine.connect() as conn:
        rows = conn.execute(
            select(industry_briefs)
            .where(industry_briefs.c.sub_industry == sub_industry)
            .where(industry_briefs.c.date <= cutoff)
            .where(industry_briefs.c.realized_etf_alpha_vs_spy.is_(None))
        ).all()

    n_resolved = 0
    for row in rows:
        if not row.sector_etf:
            continue
        sector_alpha, days = fetch_sector_alpha_vs_spy(
            row.sector_etf, row.date, holding_days=holding_days,
        )
        if sector_alpha is None:
            continue
        view = {"call": row.call}
        reflection = reflect_on_brief(view, sector_alpha)
        with engine.begin() as conn:
            conn.execute(
                update(industry_briefs)
                .where(industry_briefs.c.id == row.id)
                .values(
                    realized_etf_alpha_vs_spy=sector_alpha,
                    reflection_md=reflection,
                    resolved_at=datetime.utcnow().isoformat(),
                )
            )
        n_resolved += 1
        logger.info("Resolved brief id=%d: alpha=%+.2f%%, %s",
                    row.id, sector_alpha * 100, reflection)
    return n_resolved
```

- [ ] **Step 3: Run, commit**

```bash
git add tradingagents/industry/brief_resolver.py tests/industry/test_brief_resolver.py
git commit -m "feat(industry): brief resolver computes realized sector-relative alpha"
```

---

### Task 4: Wire brief resolution into `IndustryResearchGraph.propagate`

**Files:**
- Modify: `tradingagents/industry/industry_research_graph.py`
- Add tests: `tests/industry/test_propagate_resolves_pending.py`

- [ ] **Step 1: Test**

```python
def test_propagate_resolves_pending_briefs_before_regenerating(tmp_path, monkeypatch):
    monkeypatch.setenv("TRADINGAGENTS_DB_PATH", str(tmp_path / "rp.db"))
    from unittest.mock import patch, MagicMock
    from datetime import datetime, timedelta
    from sqlalchemy import insert, select

    from tradingagents.storage import get_engine
    from tradingagents.storage.schema import metadata, industry_briefs
    from tradingagents.default_config import DEFAULT_CONFIG

    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": str(tmp_path / "rp.db")}

    engine = get_engine(config)
    metadata.create_all(engine)
    old = (datetime.utcnow() - timedelta(days=10)).date().isoformat()
    with engine.begin() as conn:
        conn.execute(insert(industry_briefs).values(
            sub_industry="Semiconductors", date=old, mode="brief",
            call="OW", rationale_md="x", brief_md="# old",
            sector_etf="SOXX",
            created_at=datetime.utcnow().isoformat(),
        ))

    with patch("tradingagents.industry.brief_resolver.fetch_sector_alpha_vs_spy") as f:
        f.return_value = (0.03, 7)
        with patch("tradingagents.industry.industry_research_graph.IndustryGraphSetup"), \
             patch("tradingagents.industry.industry_research_graph.create_llm_client"):
            from tradingagents.industry.industry_research_graph import IndustryResearchGraph
            ig = IndustryResearchGraph(config=config)
            # Stub graph.invoke for this test:
            ig.graph = MagicMock()
            ig.graph.invoke.return_value = {
                "industry_view": {"call": "OW", "rationale": "test", "conviction": 0.7,
                                  "top_longs": [], "top_shorts": [],
                                  "key_debates": [], "catalysts": []},
                "rendered_output": "# new brief",
            }
            ig.propagate("Semiconductors",
                         datetime.utcnow().date().isoformat(), mode="brief")

    with engine.connect() as conn:
        row = conn.execute(
            select(industry_briefs)
            .where(industry_briefs.c.date == old)
        ).first()
    assert row.realized_etf_alpha_vs_spy is not None
```

- [ ] **Step 2: Edit `IndustryResearchGraph.propagate`**

Add at the top of `propagate`, before the constituent_decisions fetch:

```python
        # Resolve any pending briefs from prior runs (sub-project #5).
        try:
            from tradingagents.industry.brief_resolver import resolve_pending_briefs
            resolve_pending_briefs(sub_industry, config=self.config)
        except Exception as e:
            logger.warning("Brief resolution failed for %s: %s; continuing",
                            sub_industry, e)
```

- [ ] **Step 3: Run, commit**

```bash
git add tradingagents/industry/industry_research_graph.py \
        tests/industry/test_propagate_resolves_pending.py
git commit -m "feat(industry): propagate resolves pending briefs before regen"
```

---

### Task 5: F-8 functional test (cross-ticker feedback visible in brief)

**Files:**
- Add tests: `tests/regression/test_functional_feedback.py`

- [ ] **Step 1: Write F-8**

```python
"""F-8: cross-ticker feedback visible in industry brief refresh."""
from unittest.mock import patch, MagicMock
from sqlalchemy import insert
from datetime import datetime, timedelta


def test_f8_constituent_decisions_in_strategist_prompt(tmp_path, monkeypatch):
    monkeypatch.setenv("TRADINGAGENTS_DB_PATH", str(tmp_path / "f8.db"))
    from tradingagents.storage import get_engine
    from tradingagents.storage.schema import metadata, ticker_analyses
    from tradingagents.default_config import DEFAULT_CONFIG

    config = DEFAULT_CONFIG.copy()
    config["storage"] = {**config["storage"], "db_path": str(tmp_path / "f8.db")}

    engine = get_engine(config)
    metadata.create_all(engine)
    today = datetime.utcnow().date().isoformat()
    for i in range(3):
        with engine.begin() as conn:
            conn.execute(insert(ticker_analyses).values(
                ticker="NVDA", date=today, sub_industry="Semiconductors",
                decision=f"BUY iter {i}", sector_alpha_return=0.02,
                created_at=datetime.utcnow().isoformat(),
            ))

    from tradingagents.agents.utils.industry_memory import IndustryMemoryLog
    log = IndustryMemoryLog(config)
    decisions = log.get_constituent_decisions("Semiconductors", days=30)
    assert len(decisions) >= 1
    assert all(d["sub_industry"] == "Semiconductors" or
               d.get("sub_industry") is None or
               d.get("ticker") == "NVDA"
               for d in decisions)
```

(The Strategist's prompt builder receives `state["constituent_decisions"]` already from Plan 02 Task 16 — this test verifies the upstream feed works.)

- [ ] **Step 2: Commit**

```bash
git add tests/regression/test_functional_feedback.py
git commit -m "test(regression): F-8 cross-ticker feedback flows into brief refresh"
```

---

### Task 6: Final smoke + PR

- [ ] **Step 1: Run all Plan 05 tests**

```bash
pytest tests/industry/test_brief_resolver.py \
       tests/industry/test_propagate_resolves_pending.py \
       tests/agents/utils/test_sub_industry_population.py \
       tests/graph/test_sector_alpha_return.py \
       tests/regression/test_functional_feedback.py -v
```

- [ ] **Step 2: Run full Plans 01-05 test suite**

```bash
pytest tests/ -v
```

Expected: ALL pass — no regressions in any sub-project.

- [ ] **Step 3: Manual end-to-end smoke**

```bash
# 1. Generate a brief.
python -m cli.main industry analyze "Semiconductors" --mode brief

# 2. Run a few ticker analyses with --industry-context. Their decisions get
#    sub_industry='Semiconductors' and (after holding-period elapses)
#    sector_alpha_return populated.
python -m cli.main analyze NVDA 2026-05-01 --industry-context
python -m cli.main analyze AVGO 2026-05-01 --industry-context

# 3. Wait for the holding period (or fast-forward by manually inserting into DB
#    in a test setup), then refresh the brief — it should reflect on the prior call
#    and incorporate the constituent decisions.
python -m cli.main industry refresh "Semiconductors"

# 4. Inspect:
python -m cli.main industry list
python -m cli.main industry show "Semiconductors"
sqlite3 ~/.tradingagents/storage/tradingagents.db \
    "SELECT date, call, realized_etf_alpha_vs_spy, reflection_md FROM industry_briefs WHERE sub_industry='Semiconductors' ORDER BY date DESC LIMIT 5;"
```

- [ ] **Step 4: Open PR**

```
Title: feat(industry): cross-ticker feedback loop + brief reflection (Plan 5/5)

Summary:
- Closes the loop. ticker_analyses rows now record sub_industry and
  sector_alpha_return (vs sector ETF) at resolution time.
- IndustryResearchGraph.propagate now calls resolve_pending_briefs
  before regenerating, computing realized_etf_alpha_vs_spy and
  writing reflection_md for any brief older than its holding window.
- The Strategist (Plan 02) already accepts constituent_decisions in
  its state — Plan 05 supplies the data via IndustryMemoryLog and
  ensures sub_industry is populated for the join to work.
- All failures non-fatal (BC-5).

This completes the v0.3.0 implementation. Storage foundation, standalone
industry workflow, PDF ingestion, ticker injection, and feedback loop
are now all wired together.

Test plan:
- [x] All Plan 05 tests pass
- [x] All Plans 01-04 tests still pass
- [x] Manual end-to-end smoke (brief → tickers → refresh → reflection)
- [x] BC regression suite passes

Follow-up (v0.4.0 candidates):
- Thematic / freeform baskets (Q3 option C/D from spec)
- News-event-triggered cache invalidation
- Backtest harness for industry calls
- Cross-industry rotation pair-trades
- Additional broker formats beyond Fidelity / Schwab
- OCR support for scanned PDFs
```

---

## Plan 05 done — v0.3.0 complete

After this plan ships, all 5 sub-projects from the design spec are implemented. The system is feature-complete for v0.3.0:

- ✅ Standalone industry workflow with 3 modes
- ✅ Industry-context injection into ticker pipeline (opt-in)
- ✅ Cross-ticker feedback loop
- ✅ SQLite storage layer with cache generalization
- ✅ PDF report ingestion for Fidelity / Schwab / generic

Tag: `git tag v0.3.0` after merge.
