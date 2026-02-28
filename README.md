<div align="center">

<img src="docs/assets/banner.svg" alt="MLflow-Sentinel Banner" width="100%"/>

# MLflow-Sentinel

**Production ML Pipeline Observability — From Training to Drift**

[![CI](https://github.com/yourusername/mlflow-sentinel/actions/workflows/ci.yml/badge.svg)](https://github.com/yourusername/mlflow-sentinel/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/yourusername/mlflow-sentinel/branch/main/graph/badge.svg)](https://codecov.io/gh/yourusername/mlflow-sentinel)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-ready-2496ED?logo=docker&logoColor=white)](docker/docker-compose.yml)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

[**Live Demo**](https://mlflow-sentinel-demo.vercel.app) · [**Docs**](docs/) · [**Architecture**](#architecture) · [**Quickstart**](#quickstart)

</div>

---

## The Problem

You trained a great model. MLflow tracked every run. You shipped it.

Then, silently — the data distribution shifted. Predictions degraded. You found out from a user complaint three weeks later.

**MLflow-Sentinel closes that gap.**

It connects your MLflow tracking server to a real-time observability layer: ingesting training run metadata, monitoring live prediction logs, detecting statistical data drift, and surfacing everything in a single operations dashboard — so your team knows the moment something goes wrong, not weeks after.

---

## What It Does

| Capability | Description |
|---|---|
| 🔄 **Pipeline Ingestion** | Polls MLflow runs, experiment metadata, and artifact manifests on a configurable schedule |
| 📊 **Drift Detection** | Runs PSI, KS-test, and Jensen-Shannon divergence against your training baseline |
| 🚨 **Alerting** | Webhook-native alerting to Slack, PagerDuty, or any HTTP endpoint |
| 📡 **REST API** | FastAPI backend exposing all metrics with OpenAPI docs at `/docs` |
| 🖥️ **Live Dashboard** | React + Recharts dashboard with WebSocket-powered real-time updates |
| 🗃️ **Storage Adapters** | Pluggable backends — SQLite for local dev, PostgreSQL for production |
| 🐳 **Docker-first** | One command to run the entire stack locally |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MLflow-Sentinel                             │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │   Ingestion  │───▶│Transformation│───▶│   Storage Layer      │  │
│  │   Engine     │    │  & Drift     │    │  (SQLite/PostgreSQL)  │  │
│  │              │    │  Detection   │    │                      │  │
│  │ • MLflow API │    │ • PSI        │    │ • Run snapshots      │  │
│  │ • Pred logs  │    │ • KS-test    │    │ • Drift history      │  │
│  │ • Scheduler  │    │ • JS-div     │    │ • Alert log          │  │
│  └──────────────┘    └──────────────┘    └──────────┬───────────┘  │
│                                                     │              │
│  ┌──────────────────────────────────────────────────▼───────────┐  │
│  │                      FastAPI Backend                         │  │
│  │         REST + WebSocket · OpenAPI Docs · Auth               │  │
│  └──────────────────────────────────┬───────────────────────────┘  │
│                                     │                              │
│  ┌──────────────────────────────────▼───────────────────────────┐  │
│  │               React Dashboard (Vite + Recharts)              │  │
│  │    Live metrics · Drift charts · Alert feed · Run explorer   │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
         │                                          │
         ▼                                          ▼
  ┌─────────────┐                          ┌───────────────┐
  │ MLflow      │                          │  Alert Sinks  │
  │ Tracking    │                          │  Slack/PD/HTTP│
  │ Server      │                          └───────────────┘
  └─────────────┘
```

### Design Principles

**Pluggable by default.** Every major component — storage, alerting, drift algorithms — is backed by an abstract interface. Swap SQLite for PostgreSQL with one config change. Add a new drift metric by implementing a single abstract method.

**No MLflow coupling beyond the API.** The ingestion engine treats MLflow as a data source, not a dependency. If your team switches tracking tools, only the ingestion adapter changes.

**Pipeline over magic.** Data flows through explicit, testable stages: ingest → transform → store → alert. No hidden side effects. Each stage has its own module, its own tests, and clear inputs/outputs.

---

## Project Structure

```
mlflow-sentinel/
│
├── src/                          # Core Python package
│   ├── ingestion/                # MLflow polling & log ingestion
│   │   ├── __init__.py
│   │   ├── mlflow_client.py      # MLflow REST API wrapper
│   │   ├── log_ingester.py       # Prediction log file reader
│   │   └── scheduler.py          # APScheduler-based polling loop
│   │
│   ├── transformation/           # Data normalization & drift detection
│   │   ├── __init__.py
│   │   ├── normalizer.py         # Schema normalization for run data
│   │   ├── drift_detector.py     # PSI, KS-test, JS-divergence
│   │   └── baseline_manager.py   # Training distribution snapshots
│   │
│   ├── storage/                  # Pluggable storage adapters
│   │   ├── __init__.py
│   │   ├── base.py               # Abstract StorageAdapter interface
│   │   ├── sqlite_adapter.py     # SQLite (dev/local)
│   │   └── postgres_adapter.py   # PostgreSQL (production)
│   │
│   ├── monitoring/               # Alerting & notification
│   │   ├── __init__.py
│   │   ├── alert_engine.py       # Threshold evaluation & routing
│   │   └── sinks/
│   │       ├── slack.py          # Slack webhook sink
│   │       ├── pagerduty.py      # PagerDuty Events API sink
│   │       └── http.py           # Generic HTTP webhook sink
│   │
│   └── api/                      # FastAPI application
│       ├── __init__.py
│       ├── main.py               # App factory, middleware, lifespan
│       ├── routers/
│       │   ├── runs.py           # /api/v1/runs endpoints
│       │   ├── drift.py          # /api/v1/drift endpoints
│       │   └── alerts.py         # /api/v1/alerts endpoints
│       ├── schemas.py            # Pydantic request/response models
│       └── websocket.py          # WebSocket manager for live updates
│
├── dashboard/                    # React frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── DriftChart.jsx    # Recharts drift timeline
│   │   │   ├── RunExplorer.jsx   # Searchable run table
│   │   │   ├── AlertFeed.jsx     # Live alert stream
│   │   │   └── MetricCard.jsx    # KPI summary cards
│   │   ├── hooks/
│   │   │   ├── useWebSocket.js   # WebSocket connection hook
│   │   │   └── useMetrics.js     # SWR data fetching hook
│   │   └── App.jsx
│   ├── package.json
│   └── vite.config.js
│
├── tests/
│   ├── unit/                     # Pure unit tests (no I/O)
│   │   ├── test_drift_detector.py
│   │   ├── test_normalizer.py
│   │   └── test_alert_engine.py
│   └── integration/              # Integration tests with test containers
│       ├── test_pipeline.py
│       └── test_api.py
│
├── scripts/
│   ├── seed_demo_data.py         # Populate DB with realistic fake data
│   └── generate_baseline.py     # Create a training baseline snapshot
│
├── docker/
│   ├── Dockerfile.api            # API service image
│   ├── Dockerfile.dashboard      # Dashboard build image
│   └── docker-compose.yml        # Full stack orchestration
│
├── docs/
│   ├── architecture.md           # Deep-dive architecture doc
│   ├── configuration.md          # All config options explained
│   ├── drift_algorithms.md       # Math behind drift detection
│   └── api_reference.md          # Auto-generated from OpenAPI
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                # Lint, test, coverage on PR
│   │   └── release.yml           # Docker build & push on tag
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.md
│       └── feature_request.md
│
├── config.example.yaml           # Annotated configuration template
├── docker-compose.yml            # Symlink → docker/docker-compose.yml
├── pyproject.toml                # Package metadata & tool config
├── requirements.txt              # Runtime dependencies
├── requirements-dev.txt          # Dev/test dependencies
└── README.md
```

---

## Quickstart

### Prerequisites

- Docker & Docker Compose
- A running MLflow tracking server (or use the included demo)

### Option 1 — Full Stack with Docker (Recommended)

```bash
# Clone the repo
git clone https://github.com/yourusername/mlflow-sentinel.git
cd mlflow-sentinel

# Copy and configure
cp config.example.yaml config.yaml
# Edit config.yaml — set your MLFLOW_TRACKING_URI at minimum

# Launch everything
docker compose up -d

# Seed the dashboard with realistic demo data (optional)
docker compose exec api python scripts/seed_demo_data.py
```

Then open:
- **Dashboard** → http://localhost:5173
- **API Docs** → http://localhost:8000/docs

### Option 2 — Local Development

```bash
# Python environment
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

# Configure
cp config.example.yaml config.yaml
# Edit config.yaml

# Run the API
uvicorn src.api.main:app --reload --port 8000

# In another terminal, run the pipeline
python -m src.ingestion.scheduler

# Frontend
cd dashboard
npm install
npm run dev
```

---

## Configuration

All configuration lives in `config.yaml`. Environment variables override any key using the `SENTINEL_` prefix.

```yaml
mlflow:
  tracking_uri: "http://localhost:5000"   # Your MLflow server
  poll_interval_seconds: 60               # How often to check for new runs
  experiments:                            # Filter to specific experiments
    - "production-models"
    - "staging-*"                         # Glob patterns supported

storage:
  backend: sqlite                         # sqlite | postgres
  sqlite:
    path: "./sentinel.db"
  postgres:
    dsn: "postgresql://user:pass@host/db" # Or use SENTINEL_POSTGRES_DSN env var

drift:
  algorithms: [psi, ks_test, js_divergence]  # Enable/disable per algorithm
  thresholds:
    psi: 0.2                              # PSI > 0.2 = significant drift
    ks_test_pvalue: 0.05                  # KS p-value threshold
    js_divergence: 0.1

alerting:
  enabled: true
  cooldown_minutes: 30                    # Suppress repeated alerts
  sinks:
    slack:
      enabled: true
      webhook_url: "${SENTINEL_SLACK_WEBHOOK}"   # From env var
    pagerduty:
      enabled: false
      routing_key: "${SENTINEL_PD_ROUTING_KEY}"

api:
  host: "0.0.0.0"
  port: 8000
  cors_origins: ["http://localhost:5173"]
```

---

## Drift Detection — How It Works

MLflow-Sentinel compares live prediction feature distributions against a **baseline snapshot** captured at training time. Three complementary algorithms are run on every evaluation window:

| Algorithm | What It Catches | When to Use |
|---|---|---|
| **Population Stability Index (PSI)** | Gradual population shifts | Tabular features, binnable data |
| **Kolmogorov-Smirnov Test** | Any distributional difference | Continuous features |
| **Jensen-Shannon Divergence** | Symmetric distribution distance | Categorical or mixed features |

A drift event is recorded when any enabled algorithm exceeds its configured threshold. Multiple signals on the same window are correlated into a single event with a computed severity score.

See [`docs/drift_algorithms.md`](docs/drift_algorithms.md) for the full mathematical treatment and worked examples.

---

## API Reference

The FastAPI backend auto-generates interactive docs. Key endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/runs` | List ingested MLflow runs with filters |
| `GET` | `/api/v1/runs/{run_id}` | Full run detail with metrics history |
| `GET` | `/api/v1/drift` | Drift events, paginated, with severity filter |
| `GET` | `/api/v1/drift/summary` | Aggregated drift stats per experiment |
| `POST` | `/api/v1/baseline` | Register a new training baseline |
| `GET` | `/api/v1/alerts` | Alert history with resolution status |
| `WS` | `/ws/live` | WebSocket stream for real-time dashboard updates |

Full reference at [`docs/api_reference.md`](docs/api_reference.md) or `/docs` when running.

---

## Running Tests

```bash
# Full test suite with coverage
pytest --cov=src --cov-report=term-missing

# Unit tests only (fast, no containers needed)
pytest tests/unit/ -v

# Integration tests (requires Docker for test containers)
pytest tests/integration/ -v

# Specific module
pytest tests/unit/test_drift_detector.py -v
```

Coverage is enforced at 80% via CI. PRs that drop below will fail the check.

---

## Contributing

Contributions are welcome. Please read [`CONTRIBUTING.md`](CONTRIBUTING.md) before opening a PR.

**Adding a new drift algorithm:**
1. Implement `BaseDriftAlgorithm` in `src/transformation/drift_detector.py`
2. Add configuration schema entry in `src/api/schemas.py`
3. Register in the algorithm registry at the bottom of `drift_detector.py`
4. Add unit tests in `tests/unit/test_drift_detector.py`

**Adding a new alert sink:**
1. Implement `BaseAlertSink` in a new file under `src/monitoring/sinks/`
2. Register in `src/monitoring/alert_engine.py`
3. Add config block to `config.example.yaml`

---

## Roadmap

- [ ] **v0.2** — Prometheus metrics endpoint (`/metrics`) for Grafana integration
- [ ] **v0.3** — SHAP-based feature importance drift (not just distribution drift)
- [ ] **v0.4** — Multi-tenant support with API key auth
- [ ] **v1.0** — Kubernetes Helm chart

---

## License

MIT © 2026. See [LICENSE](LICENSE) for details.

---

<div align="center">
Built to close the gap between model training and production confidence.
</div>
