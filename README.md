# Commit Risk Predictor

**ML-Based Commit Risk Scoring for CI/CD** — flags high-risk code commits before they reach production using machine learning on git diff semantics, code churn, and author history.

[![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square)](https://python.org)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker)](https://docker.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

---

## Overview

Most production incidents are traceable to a single commit. This system analyzes every commit in your pipeline — extracting semantic features from diffs, code churn patterns, and contributor history — and outputs a **risk score before the merge happens**.

Integrates as a webhook into any CI/CD pipeline. High-risk commits get flagged automatically, low-risk commits pass through uninterrupted.

---

## Architecture

```
GitHub Webhook
      │
      ▼
┌─────────────────┐
│  Ingestion Layer │  ← src/ingestion/   Receives and queues commit events
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Feature Engine  │  ← src/features/    Diff parsing, code churn, author stats
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Training Layer  │  ← src/training/    Model training, evaluation, versioning
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Inference API   │  ← src/inference/   REST API for real-time risk scoring
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Decision Layer │  ← src/decision/    Thresholding, label assignment, routing
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Monitoring    │  ← src/monitoring/  Drift detection, score logging, alerts
└─────────────────┘
         │
         ▼
  Streamlit Dashboard  ← dashboard.py   Real-time risk visualization
```

---

## Features

- **End-to-end ML pipeline** — ingestion → feature engineering → training → inference → monitoring
- **Webhook-first design** — plugs directly into GitHub, GitLab, or any CI system
- **Rich feature extraction** — diff semantics, lines added/deleted, file churn, author risk history
- **Real-time scoring API** — low-latency REST endpoint for inline CI/CD gating
- **Streamlit dashboard** — live commit risk feed with score trends and model metrics
- **Docker Compose** — entire stack runs with a single command
- **Configurable thresholds** — tune LOW / MEDIUM / HIGH risk boundaries per repo

---

## Project Structure

```
Commit-Risk-predictor/
│
├── src/
│   ├── ingestion/          # Commit event collection and preprocessing
│   ├── features/           # Feature extraction from git diffs and metadata
│   ├── training/           # Model training, cross-validation, artifact saving
│   ├── inference/          # REST API for real-time risk prediction
│   ├── decision/           # Risk label assignment and routing logic
│   ├── monitoring/         # Score drift, model health, logging
│   ├── utils/              # Shared helpers and constants
│   └── webhook/            # GitHub webhook handler
│
├── scripts/
│   ├── run_api.py                  # Start the inference API
│   ├── run_training.py             # Trigger model training
│   ├── run_ingestion.py            # Run commit ingestion pipeline
│   ├── run_feature_engineering.py  # Extract features from raw commits
│   ├── run_labeling.py             # Label commits from bug-fix history
│   ├── run_dashboard.py            # Launch Streamlit dashboard
│   └── generate_test_data.py       # Generate synthetic commits for testing
│
├── config/                 # Environment and model configuration
├── data/                   # Raw commits, features, and labeled datasets
├── models/                 # Saved model artifacts
├── notebooks/              # Exploratory analysis
├── tests/                  # Unit and integration tests
├── docs/                   # Architecture and API documentation
├── logs/                   # Runtime and inference logs
│
├── dashboard.py            # Streamlit risk dashboard
├── Dockerfile              # API container
├── Dockerfile.dashboard    # Dashboard container
├── docker-compose.yml      # Full stack orchestration
├── requirements.txt        # Core dependencies
├── requirements-api.txt    # API-specific dependencies
├── requirements-dashboard.txt
├── render.yaml             # Render.com deployment config
└── .env.example            # Environment variable template
```

---

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/Sahojit/Commit-Risk-predictor.git
cd Commit-Risk-predictor
cp .env.example .env
# Fill in your GitHub token and other config in .env
```

### 2. Run with Docker

```bash
docker-compose up --build
```

This starts:
- **Inference API** on `http://localhost:8000`
- **Streamlit Dashboard** on `http://localhost:8501`

### 3. Run locally

```bash
pip install -r requirements.txt

# Ingest commits from a repo
python scripts/run_ingestion.py

# Engineer features
python scripts/run_feature_engineering.py

# Label commits using bug-fix history (SZZ algorithm)
python scripts/run_labeling.py

# Train the risk model
python scripts/run_training.py

# Start the inference API
python scripts/run_api.py

# Launch the dashboard
python scripts/run_dashboard.py
```

---

## API Usage

### Score a commit

```bash
POST /predict
Content-Type: application/json

{
  "commit_sha": "a3f92c1",
  "repo": "owner/repo",
  "diff": "...",
  "author": "dev@example.com"
}
```

**Response:**

```json
{
  "commit_sha": "a3f92c1",
  "risk_score": 0.87,
  "risk_label": "HIGH",
  "flagged_files": [
    "src/auth/token_handler.py",
    "src/db/session.py"
  ],
  "top_features": {
    "lines_deleted": 67,
    "author_bug_rate": 0.31,
    "files_changed": 3,
    "churn_ratio": 0.82
  }
}
```

### GitHub Webhook

Point your repo's webhook at:

```
POST https://<your-host>/webhook/github
```

Every push event is automatically scored and logged to the dashboard.

---

## Configuration

Copy `.env.example` to `.env` and set:

| Variable | Description |
|---|---|
| `GITHUB_TOKEN` | GitHub API token for commit fetching |
| `RISK_THRESHOLD_HIGH` | Score threshold for HIGH risk (default: `0.75`) |
| `RISK_THRESHOLD_MEDIUM` | Score threshold for MEDIUM risk (default: `0.45`) |
| `MODEL_PATH` | Path to saved model artifact |
| `LOG_LEVEL` | Logging verbosity (`INFO`, `DEBUG`) |

---

## Dashboard

The Streamlit dashboard shows:

- **Live commit risk feed** — every scored commit with label and score
- **Risk distribution** — histogram of scores over time
- **Model metrics** — precision, recall, AUC-ROC on recent predictions
- **High-risk alerts** — commits above threshold highlighted for review

```bash
python scripts/run_dashboard.py
# or
streamlit run dashboard.py
```

---

## Training Your Own Model

```bash
# 1. Ingest commit history from target repo
python scripts/run_ingestion.py --repo owner/repo --limit 5000

# 2. Extract features from raw diffs
python scripts/run_feature_engineering.py

# 3. Auto-label using SZZ (links bug-fix commits to bug-introducing ones)
python scripts/run_labeling.py

# 4. Train and evaluate
python scripts/run_training.py

# Model saved to models/ with evaluation report
```

---

## Testing

```bash
# Generate synthetic test data
python scripts/generate_test_data.py

# Run tests
pytest tests/ -v
```

---

## Deployment

The repo includes a `render.yaml` for one-click deployment to [Render](https://render.com):

```bash
# Push to main — auto-deploy triggers via render.yaml
```

Or deploy with Docker to any cloud VM:

```bash
docker-compose up -d
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11 |
| ML | scikit-learn, gradient boosting |
| Feature Extraction | gitpython, unidiff |
| API | FastAPI |
| Dashboard | Streamlit |
| Containerization | Docker, Docker Compose |
| CI Integration | GitHub Webhooks |
| Deployment | Render |
| Logging | Python logging + file rotation |

---

## Author

**Sahojit Karmakar** — AI/ML Engineer

[GitHub](https://github.com/Sahojit) · [LinkedIn](https://linkedin.com/in/sahojit-karmakar-38972b28a) · [Email](mailto:sahojitxd26@gmail.com)
