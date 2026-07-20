# URL Inference Pipeline

A production-grade URL classification system that detects malicious URLs. Classifies a URL as **safe** or assigns a **malicious category**: phishing, malware, C2, spam, scam, cryptomining.

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           Inference Gateway              │
                    │  (API layer: route, ensemble, respond)   │
                    └──────────────┬──────────────────────────┘
                                   │
          ┌────────────────────────┼──────────────────────────┐
          │                        │                           │
   ┌──────▼──────┐         ┌──────▼──────┐          ┌────────▼──────┐
   │ Rule Engine │         │ Classical ML│          │  LLM Router   │
   │ (regex/     │         │ (XGBoost,   │          │  (Gemini /    │
   │  blocklists)│         │  LR, RF)    │          │   Claude API) │
   └─────────────┘         └─────────────┘          └───────────────┘
          │                        │                           │
          └────────────────────────┼──────────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │  Result Merger  │
                          │  + Confidence   │
                          │  + Category     │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │  Feedback Store │
                          │  (FP/FN loop)   │
                          └─────────────────┘
```

**Routing logic**: Rule Engine (confidence ≥ 0.95 → return) → Classical ML (confidence ≥ 0.90 → return) → LLM (ambiguous band only). Thresholds are env-var configurable.

## Malicious Categories

| Category | Description |
|---|---|
| `phishing` | Credential harvesting pages |
| `malware` | Drive-by downloads, exploit kits |
| `c2` | Command & control endpoints |
| `spam` | Unsolicited link spam |
| `scam` | Fraud, fake shops, advance-fee |
| `cryptomining` | Browser-based coin mining |
| `safe` | No threat detected |

Categories are data-driven — new ones can be added via the category registry without code changes.

## Setup

```bash
pip install -e ".[dev]"
cp .env.example .env   # set GCP_PROJECT_ID and API keys
```

Required GCP APIs: Cloud Run, Vertex AI, BigQuery, Pub/Sub, Cloud Storage, Cloud Build.

## Running Locally

```bash
uvicorn src.api.main:app --reload
```

```
POST /classify      { "url": "https://..." }
POST /feedback      { "request_id": "...", "reported_label": "safe", "reporter": "user@example.com" }
GET  /categories    # list active categories
GET  /models        # list active models and weights
```

## Tech Stack

| Layer | Technology |
|---|---|
| API | Python + FastAPI |
| Classical ML | XGBoost, scikit-learn |
| Feature extraction | URLLib, tldextract, whois (optional) |
| LLM inference | Vertex AI (Gemini), Anthropic API |
| Training | Vertex AI Pipelines / Cloud Build |
| Model registry | Vertex AI Model Registry |
| Storage | BigQuery (datasets, feedback, predictions), Cloud Storage (artifacts) |
| Serving | Cloud Run |
| Feedback queue | Pub/Sub → Cloud Function → BigQuery |
| Monitoring | Cloud Monitoring + Cloud Logging |

## Roadmap

See [PLAN.md](PLAN.md) for the 8-phase implementation plan covering data infrastructure, feature engineering, classical ML, rule engine, inference gateway, feedback loop, GCP deployment, and performance analysis.
