# BTC Forecasting Initiative — Research & Engineering System (15m Horizon)

## 0) Purpose
This repository implements a reproducible research + engineering system for forecasting Bitcoin price action at a **15-minute (15m)** horizon.

The goal is not to build one “perfect model.”  
The goal is to build a **model factory** that:
1) ingests and versions data (price + optional sentiment),
2) builds features with strict no-leakage rules,
3) trains multiple model families,
4) evaluates models with walk-forward validation,
5) promotes winners via a registry gate,
6) (later) serves predictions with monitoring and retraining triggers.

This system must be:
- reproducible
- low-cost (local-first)
- portable (no lock-in)
- safe against leakage and overfitting


---

## 1) Core Research Strategy (SOTA Strategy)

### 1.1 Definition of “State of the Art”
“SOTA” is defined as:
- best out-of-sample performance under strict walk-forward validation
- robust across market regimes
- stable across folds
- ideally probabilistic/calibrated (quantiles)

SOTA is **not** the fanciest neural network.


### 1.2 Target (15m)
We predict **15-minute log-return**:

\[
y_t = \log\left(\frac{close[t+15m]}{close[t]}\right)
\]

We do **not** predict raw price.

Optional (Phase 2+):
- direction classification: `y_t > 0`
- quantiles: P10/P50/P90
- volatility/range as a second head


### 1.3 Horizon
**Default horizon: 15m**

Do not add additional horizons until:
- the 15m pipeline is stable
- models consistently beat baselines OOS


### 1.4 Model Zoo (Tournament)
All models compete under the same evaluation protocol.

#### Tier A: Baselines (mandatory)
- random walk baseline (predict 0 return)
- momentum baseline (predict last 15m return)
- ridge regression / linear regression
- LightGBM or CatBoost

#### Tier B: Deep models (Phase 2+)
Implement 1–2 first, then expand:
- PatchTST
- TimesNet
- TFT / N-BEATS / N-HiTS family

No model is “blessed.” The evaluator decides.


### 1.5 Features (No Leakage)
All features must be computable using only information available at candle close `t`.

Phase 1 (price-only):
- returns: 1, 2, 4, 8, 16, 32 intervals (interval = 15m)
- rolling volatility: 4, 8, 16, 32 windows
- rolling max/min/range: 4, 8, 16 windows
- volume deltas/z-scores: 4, 8, 16 windows
- candle structure: body, wicks, ratios
- time-of-day / day-of-week

Phase 2 (sentiment, optional):
- X API ingestion:
  - curated keywords and curated accounts
- bucketed to 15m
- raw posts stored separately from aggregates

Sentiment is not required until price-only is stable.


### 1.6 Evaluation (The Product)
Evaluation is the most important component.

#### Walk-forward validation (mandatory)
- rolling or expanding training window
- test windows aligned to time
- no random splits

#### Metrics
Forecast metrics:
- MAE, RMSE on 15m returns
- directional accuracy
- correlation(pred, realized)
- quantile loss (if probabilistic)

Robustness:
- performance by regime:
  - high-vol vs low-vol
  - trending vs mean-reverting
  - bull vs bear vs sideways
- variance across folds

#### Baseline gating
A model cannot be promoted unless it beats baselines across multiple folds.


### 1.7 Optimization Strategy
Phase 1:
- baselines + one deep model
- consistent OOS wins

Phase 2:
- Optuna HPO
- ensembles across model families and seeds
- ablations (features on/off, sentiment on/off)

Phase 3:
- regime adaptation (mixture-of-experts / gating)
- retraining triggers based on drift/performance decay


### 1.8 Research Rules (Non-negotiable)
1) No leakage.
2) Walk-forward only.
3) Baselines always included.
4) Every run is reproducible from:
   - dataset version
   - config snapshot
   - git commit hash
5) If it can’t be reproduced, it doesn’t count.


---

## 2) Infrastructure Strategy (Minimal Cost + No Lock-in)

### 2.1 Operating Philosophy
- local-first early (near-zero cost)
- portable compute via containers
- open formats (Parquet)
- burst compute for heavy jobs
- delay always-on cloud infra until value is proven

Two-plane architecture:
1) control plane (later): orchestration + metadata
2) compute plane: burst training (replaceable)


### 2.2 Cost Strategy
Phase 1 (Crawl): $0–$20/month
- personal computers
- local Parquet
- local MLflow (SQLite)

Optional:
- shared bucket ($5–$20/month)

Phase 2 (Walk): $20–$300/month
- burst GPU for sweeps
- shared bucket + light scheduling

Phase 3 (Run): $200–$2,000/month
- always-on inference + monitoring
- scheduled retraining and evaluation


### 2.3 Anti-Lock-in Rules
1) Everything is a container.
2) Infra defined in IaC (Terraform).
3) Data is open format (Parquet).
4) Object store is source of truth.
5) Stable training/inference contracts.


### 2.4 Data Lake Layout
- `data/raw/`
  - raw OHLCV
  - raw X posts (optional)
- `data/curated/`
  - cleaned OHLCV
  - resampled 15m candles
- `data/features/`
  - model-ready tables (15m aligned)

All Parquet.


### 2.5 Tooling Defaults
Core:
- Python 3.11+
- uv or poetry
- Makefile
- Docker

Data:
- CCXT (price ingestion)
- Polars or DuckDB
- Parquet

ML:
- LightGBM
- PyTorch
- Optuna

Tracking:
- MLflow (local first)


### 2.6 Compute Strategy
Personal computers handle:
- ingestion
- features
- baselines
- evaluation
- deep model prototyping

Burst compute handles:
- sweeps
- ensembles
- heavy refreshes

Replaceable backends:
- Modal (serverless GPU)
- Runpod (pods/clusters)
- spot GPUs on hyperscalers

Compute backends must remain interchangeable.


### 2.7 Orchestration
Phase 1:
- none required; Makefile is sufficient

Phase 2:
- Prefect or Airflow for scheduled ingestion/eval/retrain

Phase 3:
- K8s scheduling


### 2.8 Serving (Phase 3+)
- FastAPI inference service
- MLflow registry promotion gate
- Prometheus + Grafana monitoring
- drift detection (Evidently or custom)


---

## 3) System Contracts (Critical for Automation)

### 3.1 Dataset Contract
All datasets must have:
- stable schema
- timestamps in UTC
- candle close timestamps
- dataset version ID

OHLCV schema:
- timestamp (UTC)
- open, high, low, close, volume
- exchange (optional)

Canonical resolution: 15m.  
If ingest is 1m, resample to 15m in curated.


### 3.2 Feature Contract
Feature builders must:
- accept dataset version + config
- produce a features dataset version
- enforce no-leakage
- output:
  - X (features)
  - y (15m target)
  - metadata (horizon=15m, timestamps, windows)


### 3.3 Training Contract
Training must accept:
- features dataset version
- model config
- random seed
- git commit hash

Training must output:
- model artifact
- metrics
- logs
- config snapshot

All logged to MLflow.


### 3.4 Evaluation Contract
Evaluation must:
- run walk-forward splits
- compare to baselines
- output:
  - fold metrics
  - aggregate leaderboard row
  - regime breakdown


### 3.5 Promotion Contract
A model can be promoted only if:
- beats baselines across folds
- meets stability thresholds
- artifacts are reproducible


---

## 4) Repository Structure
```
btc-forecasting/
  README.md
  SYSTEM.md
  Makefile
  pyproject.toml

  configs/
    data.yaml
    features.yaml
    models/
      lgbm.yaml
      patchtst.yaml
      timesnet.yaml
    eval.yaml

  src/
    btc_forecasting/
      __init__.py

      data/
        ingest_price.py
        ingest_x.py
        schemas.py
        validation.py
        resample.py

      features/
        build_features.py
        feature_defs.py
        leakage_checks.py

      models/
        baselines.py
        lgbm.py
        patchtst.py
        timesnet.py
        train.py
        predict.py

      eval/
        walk_forward.py
        metrics.py
        regimes.py
        leaderboard.py

      registry/
        mlflow_utils.py
        promote.py

      utils/
        time.py
        io.py
        config.py
        logging.py

  notebooks/
    exploration.ipynb  (optional, non-authoritative)

  reports/
    leaderboard.md
    figures/
```


---

## 5) Standard Commands (Makefile Targets)
The system should support these entry points:

- `make ingest-price`
- `make resample-15m`
- `make build-features`
- `make train-baselines`
- `make train-model MODEL=patchtst`
- `make eval`
- `make leaderboard`
- `make promote`


---

## 6) Implementation Priority Order
1) price ingestion
2) resample to 15m (if needed)
3) feature builder + leakage checks
4) baselines
5) evaluator + leaderboard
6) deep model
7) HPO + ensembles
8) sentiment
9) registry + serving
