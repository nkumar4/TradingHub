---
title: TradingAgents Architecture Reference
status: reference
created: 2026-06-20
source: tradingagents/ repo
---

# TradingAgents Architecture Reference

TradingAgents is a multi-agent LLM framework for equity and crypto research. Twelve specialized agents run inside a LangGraph state machine: four analysts gather data, two debate teams argue, and two synthesis agents make a final rated trade decision.

---

## Directory Tree

```
tradingagents/
├── tradingagents/             # Main Python package
│   ├── agents/                # All agent node implementations
│   │   ├── analysts/          # Market, Sentiment, News, Fundamentals
│   │   ├── researchers/       # Bull & Bear researchers (debate)
│   │   ├── managers/          # Research Manager & Portfolio Manager
│   │   ├── risk_mgmt/         # Aggressive, Conservative, Neutral debators
│   │   ├── trader/            # Trader (plan → transaction)
│   │   ├── utils/             # Shared tools, state types, memory log
│   │   └── schemas.py         # Pydantic output types + render functions
│   ├── dataflows/             # Data vendor abstraction layer
│   ├── graph/                 # LangGraph orchestration
│   ├── llm_clients/           # Multi-provider LLM factory
│   └── default_config.py      # Single source of truth for configuration
├── cli/                       # Typer CLI + Rich TUI
└── tests/                     # 50+ test files (flat directory)
```

---

## Agent Topology

```
                        ┌─────────────────────────────────────────┐
                        │           ANALYST TEAM (1–4 agents)     │
                        │                                         │
   START ──► ─────────► │  Market Analyst   (technical indicators)│
                        │  Sentiment Analyst(news + social media) │
                        │  News Analyst     (macro + events)      │
                        │  Fundamentals     (balance sheet, P&E)  │
                        └─────────────────┬───────────────────────┘
                                          │ all reports ready
                                          ▼
                        ┌─────────────────────────────────────────┐
                        │          RESEARCH DEBATE LOOP            │
                        │                                         │
                        │   Bull Researcher ◄──► Bear Researcher  │
                        │   (max_debate_rounds iterations)        │
                        └─────────────────┬───────────────────────┘
                                          │ debate exhausted
                                          ▼
                              ┌───────────────────────┐
                              │   Research Manager    │
                              │   → ResearchPlan      │
                              │     (5-tier rating)   │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │       Trader          │
                              │   → TraderProposal    │
                              │  (entry/stop/sizing)  │
                              └───────────┬───────────┘
                                          │
                                          ▼
                        ┌─────────────────────────────────────────┐
                        │           RISK DEBATE LOOP               │
                        │                                         │
                        │  Aggressive ◄──► Conservative ◄──► Neutral
                        │  (max_risk_discuss_rounds iterations)   │
                        └─────────────────┬───────────────────────┘
                                          │ debate exhausted
                                          ▼
                              ┌───────────────────────┐
                              │   Portfolio Manager   │
                              │ → PortfolioDecision   │
                              │   (final 5-tier +     │
                              │    thesis + target)   │
                              └───────────┬───────────┘
                                          │
                                         END
```

**5-tier rating vocabulary:** Buy · Overweight · Hold · Underweight · Sell

---

## Data Flow

```
CLI prompt (ticker, date)
        │
        ▼
TradingAgentsGraph.propagate()
        │
        ├─► Phase B: Resolve pending memory entries
        │     Fetch returns → calc alpha → LLM reflection → atomic write to memory log
        │
        ├─► Inject context into initial state
        │     past_context  = memory_log.get_past_context(ticker, n_same=5, n_cross=3)
        │     instrument_context = resolve_instrument_context(ticker)  [cached]
        │
        ├─► LangGraph.invoke(initial_state)
        │     Analyst nodes → ToolNode → LLM (multi-turn until no tool_calls)
        │     Debate nodes  → LLM prose (no tools)
        │     Synthesis nodes → LLM + Pydantic structured output
        │
        ├─► Log full state → JSON   (~/.tradingagents/logs/TICKER/…/DATE.json)
        │
        ├─► Append pending entry → memory log  (~/.tradingagents/memory/trading_memory.md)
        │
        └─► Return (final_state, signal)   signal = 5-tier string
```

### Inside each analyst node

```
Agent node invoked
    │  llm.bind_tools([tool_fn, …]).invoke(messages)
    ▼
LLM returns tool_calls?
    Yes → ToolNode executes (get_stock_data / get_indicators / …)
          appends (tool_call, result) to messages
          re-invokes same agent node
    No  → prose content stored as market_report / sentiment_report / …
```

---

## Key Classes

| Class | File | Role |
|-------|------|------|
| `TradingAgentsGraph` | `graph/trading_graph.py` | Main orchestrator; holds LLM clients, memory log, graph |
| `GraphSetup` | `graph/setup.py` | Builds LangGraph StateGraph: nodes + conditional edges |
| `Propagator` | `graph/propagation.py` | Creates initial `AgentState` dict |
| `ConditionalLogic` | `graph/conditional_logic.py` | Routing: debate-loop exit conditions |
| `Reflector` | `graph/reflection.py` | LLM-generated reflection on resolved past decisions |
| `SignalProcessor` | `graph/signal_processing.py` | Deterministic regex extraction of 5-tier rating |
| `AgentState` | `agents/utils/agent_states.py` | TypedDict: all LangGraph state fields |
| `TradingMemoryLog` | `agents/utils/memory.py` | Append-only markdown log; pending → resolved lifecycle |
| `InvestDebateState` | `agents/utils/agent_states.py` | Bull/bear debate tracking (history, count) |
| `RiskDebateState` | `agents/utils/agent_states.py` | Risk debate tracking (history, count) |
| `ResearchPlan` | `agents/schemas.py` | Pydantic: rating + rationale + actions |
| `TraderProposal` | `agents/schemas.py` | Pydantic: action + entry + stop + sizing |
| `PortfolioDecision` | `agents/schemas.py` | Pydantic: rating + thesis + target + horizon |
| `SentimentReport` | `agents/schemas.py` | Pydantic: band + score + confidence + narrative |

### AgentState fields

```python
class AgentState(TypedDict):
    messages: list                     # LangChain message list (tool-use turns)
    company_of_interest: str           # Ticker symbol
    asset_type: str                    # "stock" | "crypto"
    instrument_context: str            # Resolved identity (cached yfinance lookup)
    trade_date: str                    # YYYY-MM-DD
    sender: str                        # Node identifier
    market_report: str                 # Market Analyst output
    sentiment_report: str              # Sentiment Analyst output
    news_report: str                   # News Analyst output
    fundamentals_report: str           # Fundamentals Analyst output
    investment_debate_state: dict      # Bull/bear debate history + count
    investment_plan: str               # ResearchPlan rendered to markdown
    trader_investment_plan: str        # TraderProposal rendered to markdown
    risk_debate_state: dict            # Risk debate history + count
    final_trade_decision: str          # PortfolioDecision rendered to markdown
    past_context: str                  # Memory log lessons injected at run start
```

---

## Data Vendors (Dataflows Layer)

Vendor selection is two-level: category default overridden per-tool.

```python
config["data_vendors"] = {
    "core_stock_apis":      "yfinance",      # or "alpha_vantage"
    "technical_indicators": "yfinance",
    "fundamental_data":     "yfinance",
    "news_data":            "yfinance",
    "macro_data":           "fred",
    "prediction_markets":   "polymarket",
}
config["tool_vendors"] = {
    "get_stock_data": "alpha_vantage",       # override one tool
}
```

| Tool function | Default vendor | Auth required |
|---|---|---|
| `get_stock_data` | yfinance | No |
| `get_indicators` | yfinance | No |
| `get_fundamentals` / `get_balance_sheet` / `get_cashflow` / `get_income_statement` | yfinance | No |
| `get_news` / `get_global_news` | yfinance | No |
| `get_insider_transactions` | yfinance | No |
| `get_macro_indicators` | FRED | `FRED_API_KEY` |
| `get_prediction_markets` | Polymarket | No |
| `get_verified_market_snapshot` | yfinance | No |

Error hierarchy: `NoMarketDataError` → `VendorRateLimitError` → `VendorNotConfiguredError`

---

## LLM Providers

The factory (`llm_clients/factory.py`) lazily imports provider modules.

| Provider | Config key | Env var | Thinking mode param |
|---|---|---|---|
| OpenAI | `openai` | `OPENAI_API_KEY` | `openai_reasoning_effort` ∈ {low, medium, high} |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` | `anthropic_effort` ∈ {low, medium, high} |
| Google Gemini | `google` | `GOOGLE_API_KEY` | `google_thinking_level` ∈ {minimal, high} |
| Azure OpenAI | `azure` | `AZURE_OPENAI_ENDPOINT` + `AZURE_OPENAI_API_KEY` | — |
| AWS Bedrock | `bedrock` | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | — |
| xAI / Grok | `xai` | `XAI_API_KEY` | via `backend_url` |
| Deepseek | `deepseek` | `DEEPSEEK_API_KEY` | via `backend_url` |
| Ollama (local) | `ollama` | — | `backend_url=http://localhost:11434` |
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` | via `backend_url` |

**Structured output** is provider-native:
- OpenAI/xAI: `json_schema`
- Google: `response_schema`
- Anthropic: `tool-use` wrapper

Two LLM roles per run:
- `deep_think_llm` — Research Manager, Portfolio Manager (synthesis)
- `quick_think_llm` — all analysts, researchers, debaters (analytics)

---

## Configuration System

**Single source:** `tradingagents/default_config.py` → `DEFAULT_CONFIG` dict.

```python
DEFAULT_CONFIG = {
    # LLM
    "llm_provider": "openai",
    "deep_think_llm": "gpt-5.5",
    "quick_think_llm": "gpt-5.4-mini",
    "backend_url": None,
    "temperature": None,
    "openai_reasoning_effort": None,
    "google_thinking_level": None,
    "anthropic_effort": None,

    # Debate depth
    "max_debate_rounds": 1,
    "max_risk_discuss_rounds": 1,
    "analyst_concurrency_limit": 1,
    "max_recur_limit": 100,

    # Vendors
    "data_vendors": {...},
    "tool_vendors": {},

    # News / macro
    "news_article_limit": 20,
    "global_news_article_limit": 10,
    "global_news_lookback_days": 7,

    # Benchmark
    "benchmark_ticker": None,           # explicit override
    "benchmark_map": {                  # regional defaults
        ".NS": "^NSEI",   # India
        ".T":  "^N225",   # Japan
        ".HK": "^HSI",    # Hong Kong
        "":    "SPY",     # US default
    },

    # I18n
    "output_language": "English",

    # Persistence
    "checkpoint_enabled": False,
    "results_dir":        "~/.tradingagents/logs",
    "data_cache_dir":     "~/.tradingagents/cache",
    "memory_log_path":    "~/.tradingagents/memory/trading_memory.md",
    "memory_log_max_entries": None,
}
```

**Override precedence (lowest → highest):**
1. `DEFAULT_CONFIG` hardcoded values
2. `TRADINGAGENTS_*` environment variables (9 keys, auto-coerced to correct type)
3. CLI wizard selections
4. Programmatic `TradingAgentsGraph(config={…})`

---

## Persistence / Storage

| What | Where | Format | Written by |
|---|---|---|---|
| Full analysis state | `~/.tradingagents/logs/TICKER/TradingAgentsStrategy_logs/full_states_log_DATE.json` | JSON | `TradingAgentsGraph._run_graph()` |
| Decision memory log | `~/.tradingagents/memory/trading_memory.md` | Markdown | `TradingMemoryLog` |
| Checkpoint (optional) | `~/.tradingagents/cache/checkpoints/TICKER.sqlite` | SQLite | LangGraph `SqliteSaver` |

### Memory log format

```markdown
[2025-06-20 | AAPL | Buy | pending]

DECISION:
**Rating**: Buy
**Executive Summary**: …

<!-- ENTRY_END -->

[2025-06-10 | AAPL | Overweight | +3.2% | +1.5% alpha | 5d]

DECISION:
**Rating**: Overweight
…

REFLECTION:
Azure growth thesis held. Monitor FX headwinds next time.

<!-- ENTRY_END -->
```

Pending entries are resolved in Phase B of the next `propagate()` call for the same ticker: return data is fetched, alpha vs. benchmark computed, and an LLM reflection appended atomically (`os.replace()`).

---

## CLI

Entry point: `tradingagents` (Typer app in `cli/main.py`)

**Interactive wizard flow:**
1. Ticker input (or stdin pipe)
2. Asset type detection (stock / crypto)
3. LLM provider selection
4. Deep-think model selection
5. Quick-think model selection
6. Analyst subset selection (market / sentiment / news / fundamentals)
7. Output language selection
8. Provider-specific thinking mode (effort / thinking_level)
9. API key validation

**Rich TUI during execution:**
- Agent progress table (pending → in_progress → completed)
- Live tool-call feed
- Current report panel
- Footer stats (LLM calls, tool calls, tokens, elapsed)

**Output artifacts:**
- Real-time streaming report to terminal
- JSON state log
- Memory log update

---

## Key Dependencies

```toml
langchain-core>=0.3.81
langgraph>=0.4.8
langgraph-checkpoint-sqlite>=2.0.0
langchain-anthropic>=0.3.15
langchain-google-genai>=4.0.0
langchain-openai>=0.3.23
pandas>=2.3.0
yfinance>=1.4.1
stockstats>=0.6.5
typer>=0.21.0
rich>=14.0.0
questionary>=2.1.0
```

---

## Test Conventions

- **Flat `tests/` directory** — no subdirectories
- Markers: `@pytest.mark.unit` / `.integration` / `.smoke`
- `conftest.py` fixtures: `_dummy_api_keys` (mocks env vars), `_isolate_config` (resets global config)
- Unit tests never hit real APIs — provider clients are monkeypatched

Key test files:
- `test_memory_log.py` — pending/resolved lifecycle, atomic writes, rotation
- `test_checkpoint_resume.py` — SqliteSaver per-ticker, resume-on-crash
- `test_env_overrides.py` — TRADINGAGENTS_* coercion
- `test_structured_agents.py` — Pydantic schema round-trips
- `test_signal_processing.py` — regex rating extraction
- `test_market_data_validator.py` — staleness checks
