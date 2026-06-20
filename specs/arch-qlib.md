---
title: Qlib Architecture Reference
status: reference
created: 2026-06-20
source: qlib/ repo (Microsoft)
---

# Qlib Architecture Reference

Qlib is Microsoft's open-source quantitative research framework. It provides an end-to-end ML pipeline: raw data ingestion → feature engineering → model training → prediction → portfolio backtesting, with MLflow-backed experiment tracking and an online serving layer.

---

## Directory Tree

```
qlib/
├── qlib/                          # Main Python package
│   ├── __init__.py                # qlib.init() entry point
│   ├── config.py                  # QlibConfig, DataPathManager, MODE_CONF
│   ├── constant.py                # REG_CN, REG_US, REG_TW
│   ├── log.py                     # Logging + TimeInspector
│   ├── typehint.py                # Shared type hints
│   ├── data/                      # Data layer (providers, cache, expressions)
│   │   ├── data.py                # D (global API), CalendarProvider, FeatureProvider
│   │   ├── base.py                # Expression, Feature base classes
│   │   ├── ops.py                 # 100+ expression operators
│   │   ├── cache.py               # ExpressionCache, DatasetCache
│   │   ├── filter.py              # Expression filters
│   │   ├── pit.py                 # Point-in-time data support
│   │   ├── dataset/               # Dataset, DataHandler, processors, loaders
│   │   └── storage/               # File storage backend
│   ├── model/                     # Base model classes + trainer
│   │   ├── base.py                # BaseModel, Model, ModelFT
│   │   ├── trainer.py             # Trainer, DelayTrainer, task_train()
│   │   └── ens/                   # Ensemble utilities
│   ├── backtest/                  # Backtesting engine
│   │   ├── backtest.py            # backtest_loop(), collect_data_loop()
│   │   ├── executor.py            # BaseExecutor, NestedExecutor
│   │   ├── decision.py            # Order, OrderDir, BaseTradeDecision
│   │   ├── exchange.py            # Exchange (market simulation)
│   │   ├── account.py             # Account, AccumulatedInfo
│   │   ├── position.py            # BasePosition, Position
│   │   └── report.py              # PortfolioMetrics, Indicator
│   ├── workflow/                  # Experiment management
│   │   ├── __init__.py            # QlibRecorder (global R)
│   │   ├── expm.py                # ExpManager, MLflowExpManager
│   │   ├── exp.py                 # Experiment
│   │   ├── recorder.py            # Recorder, MLflowRecorder
│   │   ├── record_temp.py         # RecordTemp (artifact generation templates)
│   │   ├── task/                  # TaskGen, RollingGen, TaskManager
│   │   └── online/                # Online serving + prediction updates
│   ├── strategy/                  # Strategy base class
│   ├── rl/                        # Reinforcement learning framework
│   ├── contrib/                   # 30+ contributed models, strategies, workflows
│   │   ├── model/                 # LSTM, GRU, ALSTM, TFT, LightGBM, XGBoost…
│   │   ├── strategy/              # Ensemble and portfolio strategies
│   │   ├── data/                  # Custom data handlers and processors
│   │   ├── workflow/              # Rolling backtest, online inference
│   │   ├── online/                # Online serving orchestration
│   │   └── meta/                  # Meta-learning approaches
│   ├── utils/                     # Time, parallelization, serialization
│   └── cli/                       # qrun entry point
├── examples/                      # YAML config examples + Jupyter notebooks
├── scripts/                       # Data downloaders (baostock, yahoo, crypto…)
└── tests/                         # Test suites per subsystem
```

---

## End-to-End ML Pipeline

```
Raw Data (CSV / Pickle / NFS)
        │
        ▼ [DataLoader]
Loaded DataFrame  (datetime × instruments × fields)
        │
        ▼ [Processors: Fillna → Normalize → Dropna → Zscore → …]
Processed DataFrame
        │
        ▼ [Expression Engine: Feature / Ops / Cache]
Calculated Features  (lazy, Cython-accelerated, cached)
        │
        ▼ [DataHandler → DatasetH]
Segmented Data  (train / valid / test splits)
        │
        ▼ [Model.fit(dataset)]
Trained Model  (params.pkl saved to Recorder)
        │
        ▼ [Model.predict(dataset, segment="test")]
Predictions  (pd.Series: datetime × instrument)
        │
        ├──► [RecordTemp → IC, Long-Short analysis, signal evaluation]
        │          Evaluation Metrics (IC, precision, recall, etc.)
        │
        └──► [backtest_loop(Exchange, Account, Strategy)]
                   Backtest Report  (returns, Sharpe, drawdown, trades)
                        │
                        ▼ [MLflowRecorder.save_objects()]
                   Artifacts in MLflow  (model, dataset, predictions, reports)
```

---

## Key Classes

### Data Layer

| Class | File | Role |
|-------|------|------|
| `D` | `data/data.py` | Global data API singleton (`D.features()`, `D.calendar()`) |
| `CalendarProvider` | `data/data.py` | Trading calendars; locate timestamps, validate date ranges |
| `InstrumentProvider` | `data/data.py` | Stock universe queries |
| `FeatureProvider` | `data/data.py` | Load raw OHLCV fields from storage |
| `ExpressionProvider` | `data/data.py` | Cache and serve calculated expressions |
| `Expression` / `Feature` | `data/base.py` | Lazy-load base; builds expression DAG |
| `ExpressionOps` | `data/ops.py` | 100+ operators: Abs, Sign, Ref, Rolling, Expanding, Mean, Std… |
| `DataHandler` / `DataHandlerLP` | `data/dataset/handler.py` | Load → process → cache DataFrame; defines train/valid/test |
| `DataLoader` | `data/dataset/loader.py` | Abstract source loading (CSV, Pickle, etc.) |
| `Processor` | `data/dataset/processor.py` | Transform chain: DropnaLabel, Fillna, Normalization, Zscore |
| `Dataset` / `DatasetH` | `data/dataset/__init__.py` | Segment data; hand off to model |
| `Reweighter` | `data/dataset/weight.py` | Sample reweighting during training |

### Model Layer

| Class | File | Role |
|-------|------|------|
| `BaseModel` | `model/base.py` | Abstract: `predict()` only |
| `Model` | `model/base.py` | Learnable: `fit(dataset)` + `predict(dataset, segment)` |
| `ModelFT` | `model/base.py` | Fine-tunable variant |
| `Trainer` | `model/trainer.py` | Coordinate model training; save to recorder |
| `DelayTrainer` | `model/trainer.py` | Defer training to second phase (online/parallel) |

### Workflow / Experiment Tracking

| Class | File | Role |
|-------|------|------|
| `QlibRecorder` | `workflow/__init__.py` | Global `R` singleton; context manager for experiments |
| `ExpManager` / `MLflowExpManager` | `workflow/expm.py` | MLflow backend integration |
| `Experiment` | `workflow/exp.py` | Experiment metadata and lifecycle |
| `Recorder` / `MLflowRecorder` | `workflow/recorder.py` | Log params, metrics; save/load Python objects |
| `RecordTemp` | `workflow/record_temp.py` | Template: generate IC, backtest, analysis artifacts |
| `TaskGen` / `RollingGen` | `workflow/task/gen.py` | Generate task variations (rolling periods, hyperparams) |
| `TaskManager` | `workflow/task/manage.py` | Manage task lifecycle and execution order |

### Backtest Engine

| Class | File | Role |
|-------|------|------|
| `Exchange` | `backtest/exchange.py` | Market simulator: order matching, costs, limit moves |
| `Account` | `backtest/account.py` | Cash, positions, accumulated metrics |
| `BasePosition` / `Position` | `backtest/position.py` | Stock holdings + portfolio value at each timestamp |
| `Order` | `backtest/decision.py` | Single trade: stock, amount, direction, time range |
| `BaseTradeDecision` | `backtest/decision.py` | Encapsulated list of Orders for one trading period |
| `BaseExecutor` / `NestedExecutor` | `backtest/executor.py` | Orchestrate strategy → exchange → account; time loop |
| `BaseStrategy` | `strategy/base.py` | Abstract: `generate_trade_decision()` |
| `PortfolioMetrics` / `Indicator` | `backtest/report.py` | Sharpe, drawdown, returns post-backtest |

---

## Workflow / Experiment System

```
with R.start(experiment_name="lgb_cn_2024", recorder_name="run_01"):
    │
    ▼
QlibRecorder.start_exp()
    │  MLflowExpManager creates/loads Experiment
    │  MLflowRecorder starts a run
    ▼
task_train(task_config)
    │  _log_task_info()   → log params, save task YAML
    │  _exe_task()        → model.fit(dataset); save params.pkl + dataset
    │
    │  for record_cfg in task["record"]:
    │      RecordTemp.generate()
    │          ├── ICAnalysisRecord   → IC, ICIR, Rank IC
    │          ├── BacktestRecord     → portfolio returns, trades
    │          └── SignalAnalysisRecord → long/short analysis
    │      recorder.save_objects(artifact_path=…, **outputs)
    ▼
recorder.end_run(status="FINISHED")
    All artifacts in MLflow URI
```

### Task config structure (YAML)

```yaml
qlib_init:
  provider_uri: "~/.qlib/qlib_data/cn_data"
  region: cn

task:
  model:
    class: LGBModel
    module_path: qlib.contrib.model
    kwargs:
      loss: mse
      num_leaves: 128

  dataset:
    class: DatasetH
    kwargs:
      handler:
        class: DataHandlerLP
        kwargs:
          instruments: csi300
          start_time: 2008-01-01
          end_time:   2020-08-01
          infer_processors: [RobustZScoreNorm, Fillna]
          learn_processors: [DropnaLabel, CSZScoreNorm]
      segments:
        train: [2008-01-01, 2014-12-31]
        valid: [2015-01-01, 2016-12-31]
        test:  [2017-01-01, 2020-08-01]

  record:
    - class: ICAnalysisRecord
    - class: BacktestRecord
      kwargs:
        strategy:
          class: TopkDropoutStrategy
          kwargs: {topk: 50, n_drop: 5}
```

---

## Data Infrastructure

### Caching layers (innermost → outermost)

1. **Cython expression cache** — memoizes rolling/expanding calculations in-process
2. **Memory cache `H`** — global dict for calendars, instruments, recent expressions
3. **DiskExpressionCache** — persistent per-expression cache on disk
4. **DiskDatasetCache / SimpleDatasetCache** — caches full processed DataFrame to skip re-run

### Multi-frequency support

```python
qlib.init(provider_uri={
    "day":  "~/.qlib/qlib_data/cn_data",
    "1min": "~/.qlib/qlib_data/cn_data_1min",
})

exchange = get_exchange(freq="1min", …)   # switches to intraday data
```

### Data path resolution

```
qlib.init(provider_uri=…, region=…)
        │
        ▼ QlibConfig.resolve_path()
DataPathManager.get_data_uri(freq)
        │
        ├── local path → used directly
        └── NFS path   → auto-mounted if mount_path set
```

---

## Model Zoo (contrib/model/)

### Deep Learning (PyTorch)

| Model | File | Architecture |
|-------|------|---|
| LSTM | `pytorch_lstm.py` | Standard LSTM |
| GRU | `pytorch_gru.py` | Gated Recurrent Unit |
| ALSTM | `pytorch_alstm.py` | Attention-LSTM |
| HIST | `pytorch_hist.py` | Heterogeneous stock graph |
| TFT | `pytorch_tft.py` | Temporal Fusion Transformer |
| LocalFormer | `pytorch_localformer.py` | Local self-attention |
| TCN | `pytorch_tcn.py` | Temporal Convolutional Network |
| IGMTF | `pytorch_igmtf.py` | Interaction-based graph model |
| KRNN | `pytorch_krnn.py` | Kernel RNN |
| TabNet | `pytorch_tabnet.py` | Attention-based tabular |
| SFM | `pytorch_sfm.py` | State Frequency Memory |
| TRA | `pytorch_tra.py` | Temporal Routing Adaptor |

### Tree-Based

| Model | File |
|-------|------|
| LightGBM | `gbdt.py` |
| XGBoost | `xgboost.py` |
| CatBoost | `catboost_model.py` |
| Linear / Ridge | `linear.py` |

### Ensemble

| Model | File |
|-------|------|
| DEnsembleModel | `double_ensemble.py` |
| General ensemble utils | `model/ens/` |

---

## Backtest Engine

### Core loop

```python
exchange = get_exchange(freq="day", start_time=…, end_time=…)
account  = create_account_instance(init_cash=1e9, benchmark="SH000300")
executor = BaseExecutor(time_per_step="day")
strategy = TopkDropoutStrategy(model=model, dataset=dataset, topk=50)

backtest_loop(start, end, exchange, account, executor, strategy)
```

### Per-step execution

```
strategy.generate_trade_decision(execute_result)
        │  → BaseTradeDecision (list of Orders)
        ▼
executor.execute(trade_decision)
        │  for each order:
        │      Exchange.get_order_handling(order, account)
        │          → deal_amount, factor set
        │      Account.update_position(order, cost)
        │          → current_position adjusted
        │      PortfolioMetrics.update() [optional]
        ▼
execute_result returned for next strategy call
```

### Nested execution (portfolio → stock level)

`NestedExecutor` runs a parent strategy that delegates to a child executor per instrument, sharing Exchange and Calendar but maintaining separate position state per level.

### Available strategies (contrib/strategy/)

- `TopkDropoutStrategy` — rank-based long portfolio, rotation with dropout
- `WeightStrategyBase` — allocation by predicted score weight
- `EnhancedIndexingStrategy` — index-tracking with alpha tilt
- `RLStrategy` / `RLIntStrategy` — reinforcement learning-based

---

## Configuration System

### `qlib.init()` signature

```python
qlib.init(
    default_conf="client",              # "client" or "server" template
    provider_uri="~/.qlib/qlib_data/cn_data",   # dict for multi-freq
    region="cn",                        # REG_CN, REG_US, REG_TW
    mount_path=None,                    # NFS mount
    exp_manager={                       # MLflow backend config
        "class": "MLflowExpManager",
        "kwargs": {"uri": "mlruns/"},
    },
    dataset_cache=None,                 # "DiskDatasetCache" | "SimpleDatasetCache"
    expression_cache=None,              # "DiskExpressionCache"
    kernels=NUM_USABLE_CPU,            # parallelism
    skip_if_reg=False,
)
```

### Override precedence (lowest → highest)

1. `_default_config` base
2. `MODE_CONF` (client / server template)
3. `QSettings` (pydantic-settings from `QLIB_*` env vars)
4. User `kwargs` to `qlib.init()`

### Region-specific defaults

| Region | `trade_unit` | `limit_threshold` | Default benchmark |
|--------|---|---|---|
| CN | 100 (round lots) | 10% daily limit | CSI 300 |
| US | 1 | None | SPY |
| TW | 1000 | 10% | TAIEX |

---

## CLI

### `qrun` (main entry point)

```bash
qrun workflow <config.yaml> \
     --experiment_name my_exp \
     --uri_folder mlruns/
```

Located: `qlib/cli/run.py` → `workflow()` function.

- Loads YAML with Jinja2 templating (env vars, `BASE_CONFIG_PATH` inheritance)
- Calls `qlib.init()` from YAML `qlib_init` section
- Calls `task_train(task_config)` which drives the full pipeline

### Data scripts

`scripts/data_collector/` contains downloaders for:
- **baostock** — Chinese A-share OHLCV
- **Yahoo Finance** — global equities
- **crypto** — cryptocurrency OHLCV
- **index constituents** — CSI 300/500, Russell, etc.

---

## Key External Dependencies

```toml
# Core
pandas>=1.1
numpy>=1.24
mlflow                  # experiment tracking backend
lightgbm               # GBDT model
redis                  # caching + task queue (server mode)
pymongo                # task/experiment storage
dill                   # serialization (objects with closures)
joblib                 # parallelization
ruamel.yaml>=0.17.38  # YAML parsing with comments
fire                   # CLI (scripts)
cvxpy                  # convex optimization (portfolio)

# Optional
[rl]       tianshou<=0.4.10  # RL training framework
[pytorch]  torch             # deep learning models
[analysis] plotly, statsmodels
[client]   python-socketio, tables
```

---

## Design Patterns

| Pattern | Where used |
|---------|---|
| **Singleton** | `QlibRecorder` global `R`; `QlibConfig` global config |
| **Factory** | `init_instance_by_config()` — dynamic class from dict |
| **Template Method** | `RecordTemp.generate()`, `Processor.__call__()` |
| **Strategy** | `BaseStrategy.generate_trade_decision()` |
| **Chain of Responsibility** | Processor chain in DataHandler |
| **Observer** | Account / PortfolioMetrics update on each trade |
| **Builder** | YAML config → nested object construction |
| **Lazy Evaluation** | Expression tree; computed only when fetched |

---

## Test Structure

```
tests/
├── backtest/               # executor, exchange, account
├── dataset_tests/          # DatasetH, handler, loader, processor
├── data_mid_layer_tests/   # expression engine, operators, caching
├── model/                  # model training and prediction
├── ops/                    # expression operations
├── storage_tests/          # storage backend
├── rl/                     # RL strategy and training
├── rolling_tests/          # rolling / online workflow
├── dependency_tests/       # integration tests
└── misc/                   # miscellaneous
```

Tests require `qlib.init()` to be called in fixtures. Integration tests hit real data files; unit tests mock providers.

---

## Key Design Constraints

1. **Data is immutable per timestamp** — point-in-time design prevents lookahead in features
2. **Expression tree is lazy** — expressions not evaluated until `load()` is called; enables large virtual feature spaces
3. **Multi-frequency is first-class** — `provider_uri` dict and `freq=` parameter allow day/minute/tick switching without code changes
4. **All objects serializable** — `dill` used instead of `pickle` so models with closures survive experiment checkpointing
5. **MLflow as sole truth for artifacts** — model weights, datasets, and reports are keyed by run UUID; reproducibility relies on the MLflow store
