# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

**Pre-implementation.** The repo currently contains only `README.md`, `PLAN.md`, and this file — no `src/`, `tests/`, or `pyproject.toml` yet. The rest of this document describes the *target* architecture; treat it as a spec, not a description of existing code. The commands below will not work until the corresponding modules exist.

`PLAN.md` is the source of truth for what to build and in what order: an 8-phase plan (data infra → feature engineering → classical ML → rule engine → inference gateway → feedback loop → GCP deploy → perf analysis), each with concrete tasks and exit criteria. Consult it before starting new work, and keep it and this file in sync as things get built.

## Commands

```bash
# Install
pip install -e ".[dev]"

# Run inference service locally
uvicorn src.api.main:app --reload

# Run all tests
pytest tests/

# Run a single test
pytest tests/path/to/test_file.py::test_name -v

# Train a model locally (small dataset)
python -m src.pipeline.train --model classical --dataset data/base/

# Load a dataset into BigQuery
python -m src.pipeline.load_data --source phishtank

# Register a trained model in Vertex AI
python -m src.pipeline.register --run-id <id>

# Submit training to Vertex AI
python -m src.pipeline.submit --env prod
```

## Directory Structure

```
src/
├── api/               # FastAPI app, request/response schemas
├── models/
│   ├── rule_engine/   # Blocklist + regex rules (YAML in rules/, hot-reloadable)
│   ├── classical/     # XGBoost, LR, RF training + inference
│   └── llm/           # LLM adapter (Gemini, Claude)
├── features/          # URL feature extraction — no network calls by default
├── categories/        # Category registry backed by BQ
├── feedback/          # Feedback ingestion, FP/FN schema
├── evaluation/        # Metrics, confusion matrix, A/B analysis
└── pipeline/          # Training pipeline DAGs
data/
├── base/              # Seed datasets (Parquet: url, label, category, source, timestamp)
└── schemas/           # BQ table schemas
infra/
├── terraform/         # GCP resources
└── cloudbuild/        # CI/CD configs
```

## Key Design Decisions

**Classifier protocol**: all models implement `predict(url: str) -> ClassificationResult` where `ClassificationResult` includes `{category, confidence, model_id, feature_vector}`.

**Confidence threshold routing**: Rule Engine → Classical ML → LLM, short-circuiting at ≥ 0.95 and ≥ 0.90 respectively. LLM only invoked for the ambiguous 0.50–0.90 band. Thresholds are env-var configurable without code changes.

**Async feedback**: FP/FN reports go to Pub/Sub → Cloud Function → BQ `feedback` table. Never synchronous with inference. Nightly retraining triggers when queued feedback count > 500 (configurable).

**Network features off by default**: WHOIS/DNS lookups add 100–500ms. Gated by `ENABLE_NETWORK_FEATURES=true`. Default feature extraction must run in < 5ms p99.

**Category registry**: categories live in BQ `categories` table. New categories need ≥ 100 labeled examples and an optional rule hint — no model rewrite needed.

**Ensemble weights**: stored in config, tunable without redeployment.

## GCP Setup

- Project: `GCP_PROJECT_ID` env var; Region: `us-central1`
- Service account: `url-inference-sa@{project}.iam.gserviceaccount.com`
- BQ tables: `urls_labeled`, `predictions`, `feedback`, `categories`, `model_registry`
- Secrets via Secret Manager — no inline credentials

## Coding Conventions

- Python 3.11+; type annotations on all public functions
- Pydantic v2 for all API schemas and config
- `pytest` for tests; no mocking of feature extraction — use real URL fixtures
