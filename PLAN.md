# URL Inference Pipeline — Phase Plan

**Learning goals:** inference pipeline design, inference tuning, performance analysis  
**Deploy target:** GCP (Cloud Run + Vertex AI)

---

## Phase 1 — Foundation & Data Acquisition

**Goal:** Establish the project skeleton and a reproducible pipeline that downloads base datasets from public sources, understands them, refines them into a clean unified schema, and stores committed CSV files in the repo. No cloud dependencies yet — BigQuery/Vertex AI land in Phase 7.

### Tasks

1. **Project scaffolding**
   - `pyproject.toml` with dependency groups (`train`, `serve`, `dev`)
   - `src/` layout per CLAUDE.md
   - Pre-commit hooks: black, isort, mypy
   - `.env.example` with all required env vars
   - `.gitignore` excludes `data/raw/` (large, upstream-owned dumps)

2. **Download base datasets**
   - Re-runnable downloader scripts that fetch from source:
     - [PhishTank](https://phishtank.org/developer_info.php) — phishing URLs (labeled)
     - [Tranco top-1M](https://tranco-list.eu/) — likely-safe baseline
     - [URLhaus](https://urlhaus.abuse.ch/api/) — malware/C2 URLs
     - [OpenPhish](https://openphish.com/) — phishing feed
   - Raw downloads land in `data/raw/` (gitignored)
   - Command: `python -m src.pipeline.download --source phishtank`

3. **Understand the data (EDA)**
   - Explore each source: row counts, label/category distribution, URL length + token distributions, duplicate rate, cross-source overlap, malformed rows
   - Document per-source quirks (format, encoding, update cadence, rate limits, licensing) and class imbalance (safe vs. malicious volume) in `data/README.md`

4. **Refine & normalize**
   - Clean pipeline: dedupe (exact + normalized URL), drop malformed/empty rows, canonicalize scheme/host casing
   - Map each source's native labels to the unified schema: `{url, label, category, source, timestamp}`
     - `label` ∈ `{safe, malicious}`; `category` ∈ the 6 malicious categories or `safe`
   - Sample/balance a working set (e.g., cap the Tranco safe baseline to a manageable size)
   - Emit a data manifest (row counts per source/category, dedup stats) as a reproducibility record

5. **Store as CSV in the repo**
   - Refined, committed datasets in `data/base/` as CSV with the unified schema (e.g., `data/base/urls_labeled.csv`, optionally per-split files)
   - Keep committed files small enough to version (curate/sample if needed); large raw dumps stay in `data/raw/`
   - Loader `python -m src.pipeline.load_data` reads `data/base/*.csv` into a DataFrame for downstream phases

6. **Category definitions**
   - Define the initial 6 categories in a versioned, file-backed source (`data/base/categories.csv` + a Python enum): `{id, name, description}`
   - Keep it simple for now; the runtime-extensible, BQ-backed registry lands in Phase 7

**Exit criteria:** download + refine pipeline runs end-to-end and is re-runnable; `data/base/` holds committed CSV(s) with ≥10k labeled URLs across ≥3 categories; EDA findings and a data manifest are documented in `data/README.md`.

---

## Phase 2 — Feature Engineering

**Goal:** Extract a rich, consistent feature vector from any URL. This is the most leverage point for classical model quality.

### Feature Groups

| Group | Features |
|---|---|
| Lexical | URL length, token count, digit ratio, special char count, entropy |
| Structural | Protocol, subdomain depth, path depth, query param count, fragment present |
| TLD | TLD type (gTLD/ccTLD/new), TLD in known-malicious list |
| Host | IP address instead of hostname, hostname length, numeric chars in hostname |
| Brand abuse | Levenshtein distance to top-500 brand names (typosquatting signal) |
| Encoding | Percent-encoding count, double encoding, unusual unicode |
| Path signals | Suspicious keywords (login, secure, account, update, verify) |

### Tasks

1. Implement `src/features/extractor.py` — `extract(url: str) -> FeatureVector`
2. Benchmark extraction latency (target: < 5ms per URL, no network calls)
3. Add optional async enrichment (WHOIS, DNS) gated by a feature flag
4. Write fixtures in `tests/features/` with known-good and known-bad URLs

**Exit criteria:** Feature extractor runs in < 5ms (p99) on 1000 URLs without network. Full test suite passes.

---

## Phase 3 — Classical ML Models

**Goal:** Train, evaluate, and register multiple classical ML models. Establish baseline metrics and model versioning.

### Models

| Model | Library | Notes |
|---|---|---|
| Logistic Regression | scikit-learn | Fast, interpretable baseline |
| Random Forest | scikit-learn | Good for feature importance |
| XGBoost | xgboost | Primary model; best accuracy/speed tradeoff |

### Tasks

1. **Training pipeline** (`src/pipeline/train.py`)
   - Load labeled CSV from `data/base/`
   - Train/val/test split (stratified by category)
   - Grid search / Optuna HPO for XGBoost
   - Output: model artifact (.joblib), feature importance report

2. **Evaluation** (`src/evaluation/`)
   - Metrics: precision, recall, F1 per category, macro-average, AUC-ROC
   - Confusion matrix visualization
   - False positive rate on Tranco safe list (business-critical metric)
   - Save evaluation report as JSON artifact

3. **Model registry**
   - Version artifacts locally first: write `.joblib` + a metadata JSON (`model_type`, `dataset_version`, `eval_metrics`, `feature_schema_version`) under `models/` with a run-id manifest
   - Helper: `python -m src.pipeline.register --run-id <id>`
   - Promotion to Vertex AI Model Registry happens in Phase 7

4. **Local inference** (`src/models/classical/predictor.py`)
   - `ClassicalPredictor(model_path).predict(url)` → `ClassificationResult`
   - Returns: `{category, confidence, model_id, feature_vector}`

**Exit criteria:** XGBoost achieves F1 ≥ 0.90 macro on held-out test set. FP rate on Tranco ≤ 1%. Model artifact + metadata versioned locally under `models/` (Vertex AI registration deferred to Phase 7).

---

## Phase 4 — Rule Engine

**Goal:** A zero-latency first-pass filter that short-circuits inference for high-confidence cases.

### Tasks

1. **Blocklist loader** — load known-bad domains/IPs from GCS or local file, indexed in a trie for O(log n) lookup
2. **Regex rule set** — curated patterns for common phishing kits, URL shortener chains, newly-registered domain patterns
3. **Rule protocol** — each rule returns `{matched: bool, category, confidence, rule_id}`
4. **Priority chain** — blocklist → regex → return early if confidence ≥ 0.95, else pass to ML
5. **Rule management** — rules stored as YAML in `src/models/rule_engine/rules/`; hot-reloadable without restart

**Exit criteria:** Rule engine processes 10k URLs in < 100ms total. Known PhishTank URLs in blocklist return in < 1ms.

---

## Phase 5 — Multi-Model Inference Gateway

**Goal:** A single FastAPI service that orchestrates rule engine → classical ML → LLM, merges results, and returns a classification.

### API Design

```
POST /classify
Body: { "url": "https://example.com/login" }
Response: {
  "url": "...",
  "verdict": "phishing" | "malware" | "c2" | "spam" | "scam" | "cryptomining" | "safe",
  "confidence": 0.97,
  "model_used": ["rule_engine", "classical_xgb"],
  "explanation": "...",   // only present for LLM path
  "request_id": "uuid"
}

POST /feedback
Body: {
  "request_id": "uuid",
  "reported_label": "safe",   // what the user says it should be
  "reporter": "user@example.com"
}

GET /categories            # list all active categories
POST /categories           # add a new category (admin)
GET /models                # list active models and their weights
```

### Routing Logic

```
URL input
   │
   ▼
Rule Engine ──── confidence ≥ 0.95 ──→ return result
   │
   │ (low confidence)
   ▼
Classical ML ─── confidence ≥ 0.90 ──→ return result
   │
   │ (ambiguous: 0.50–0.90)
   ▼
LLM (Gemini/Claude) ──────────────────→ return result with explanation
```

### Tasks

1. FastAPI app with `/classify`, `/feedback`, `/categories`, `/models` endpoints
2. `InferenceOrchestrator` class that runs the routing logic
3. `ResultMerger` — weighted voting when multiple models contribute
4. Confidence thresholds configurable via env vars (no code change to tune)
5. Async LLM adapter (`src/models/llm/adapter.py`) supporting Gemini Pro + Claude as backends
6. Request/response logging via a pluggable sink (async, non-blocking): local CSV/JSONL pre-GCP, swapped for the BQ `predictions` table in Phase 7
7. Health check endpoint, Prometheus metrics endpoint

**Exit criteria:** `/classify` returns in < 200ms (p95) for rule/classical path, < 3s for LLM path. All endpoints pass integration tests with real URLs.

---

## Phase 6 — Feedback Loop & Category Management

**Goal:** Close the learning loop — user-reported FP/FN flows back into the training data; new categories can be added without a code deploy. Built local-first (append-only CSV store); the Pub/Sub → Cloud Function → BQ path is wired up in Phase 7.

### Tasks

1. **Feedback ingestion**
   - `POST /feedback` appends to a local feedback store (`data/feedback/feedback.csv`) behind the same pluggable sink used for predictions
   - Schema: `{request_id, original_verdict, reported_label, reporter, timestamp, status}`
   - In Phase 7 the sink is swapped for Pub/Sub topic `url-feedback` → Cloud Function → BQ `feedback` table (no API change)

2. **Feedback review workflow**
   - Admin endpoint `GET /feedback?status=pending` to review queue
   - `PATCH /feedback/{id}` to approve/reject (approved → appended to `data/base/urls_labeled.csv`)
   - Approval triggers a retraining job if queued feedback count > threshold (default: 500)

3. **Category management**
   - `POST /categories` creates a new category in the file-backed registry (`data/base/categories.csv`)
   - New category requires: `name`, `description`, at least one example URL
   - System creates a placeholder rule + prompts for labeled data upload
   - Category appears in classification output once at least 100 labeled examples exist

4. **Automated retraining**
   - Scheduled trigger (local cron pre-GCP; Cloud Scheduler in Phase 7) runs `training_pipeline` nightly
   - Pipeline: load labeled + approved-feedback CSV → retrain XGBoost → evaluate → auto-promote if metrics don't regress

**Exit criteria:** End-to-end feedback test: submit FP via API → approve in admin → verify it appears in the next training batch (`data/base/urls_labeled.csv`). New category created and returned in `/categories` response.

---

## Phase 7 — GCP Deployment & CI/CD

**Goal:** Deploy the inference service to Cloud Run, training pipeline to Vertex AI, with automated CI/CD.

### GCP Resources (Terraform)

```
infra/terraform/
├── main.tf              # provider, project
├── run.tf               # Cloud Run service
├── vertex.tf            # Vertex AI dataset, pipeline, model registry
├── bigquery.tf          # datasets, tables
├── pubsub.tf            # feedback topic + subscription
├── storage.tf           # GCS buckets (models, data)
├── iam.tf               # service accounts, bindings
└── monitoring.tf        # alerting policies, dashboards
```

### Cloud Run Configuration

- Min instances: 1 (avoid cold start on first request)
- Max instances: 10
- CPU: 2, Memory: 2Gi
- Concurrency: 80
- Environment vars via Secret Manager references

### CI/CD (Cloud Build)

```yaml
# Triggers on main branch push:
steps:
  - run tests
  - build Docker image → push to Artifact Registry
  - deploy to Cloud Run (canary: 10% traffic → promote if health check passes)
  - run smoke tests against new revision
  - full traffic cutover or rollback
```

### Tasks

1. Terraform modules for all GCP resources
2. **Data & registry migration to GCP** (formerly Phase 1's cloud setup)
   - Create BQ tables `urls_labeled`, `predictions`, `feedback`, `categories`, `model_registry`; schema files in `data/schemas/`
   - Load `data/base/*.csv` into `urls_labeled` and seed `categories` from `data/base/categories.csv`; move the category registry to BQ-backed (runtime-extensible)
   - Swap the pluggable prediction/feedback sinks (Phases 5–6) from local CSV to Pub/Sub → Cloud Function → BQ — no API change
   - Register the local model artifacts (Phase 3) in Vertex AI Model Registry
3. `Dockerfile` for inference service (multi-stage, distroless base)
4. `cloudbuild.yaml` for test → build → deploy pipeline
5. Secrets in Secret Manager: API keys, DB passwords
6. VPC connector for Cloud Run → Cloud SQL/BQ private access
7. Cloud Armor WAF rule (rate limit `/classify` to 100 req/s per IP)

**Exit criteria:** `terraform apply` provisions all resources. Cloud Build pipeline deploys on push to `main`. Service live at `https://url-inference-{hash}.run.app`.

---

## Phase 8 — Performance Analysis & Tuning

**Goal:** Measure everything, tune the bottlenecks, understand the accuracy vs. latency vs. cost tradeoffs.

### Latency Analysis

| Metric | Target | Instrument |
|---|---|---|
| Rule engine p99 | < 2ms | Custom histogram metric |
| Classical ML p99 | < 50ms | Custom histogram metric |
| LLM path p99 | < 3s | Custom histogram metric |
| End-to-end p95 | < 200ms (non-LLM) | Cloud Run request latency |

### Accuracy Analysis

- Daily dashboard: precision/recall per category vs. baseline
- FP rate on Tranco list (sampled weekly)
- Feedback acceptance rate (approved vs. rejected FP/FN reports)
- Model drift detection: rolling 7-day accuracy vs. 30-day baseline

### Cost Analysis

- Per-classification cost breakdown: LLM call % of total, Vertex AI serving cost
- Cost vs. accuracy curve: "what accuracy do we lose if we remove LLM path?"

### Tuning Experiments (in `notebooks/`)

1. **Threshold tuning** — sweep confidence thresholds, plot precision-recall tradeoff
2. **Feature ablation** — drop feature groups, measure accuracy drop, identify top-10 features
3. **Model comparison A/B** — split 20% traffic to challenger model, compare metrics over 7 days
4. **LLM prompt tuning** — compare zero-shot vs. few-shot classification prompts on ambiguous URLs
5. **Batch inference** — measure throughput at 10/100/1000 concurrent requests

### Deliverables

- `notebooks/01_threshold_analysis.ipynb`
- `notebooks/02_feature_ablation.ipynb`
- `notebooks/03_ab_test_results.ipynb`
- `notebooks/04_llm_prompt_comparison.ipynb`
- `notebooks/05_cost_accuracy_tradeoff.ipynb`
- Cloud Monitoring dashboard: latency, accuracy, cost, feedback queue depth

**Exit criteria:** All 5 notebooks complete with findings documented. Cloud Monitoring dashboard live. At least one tuning improvement implemented based on findings (e.g., threshold change that reduces LLM calls by ≥20% with < 1% accuracy drop).

---

## Timeline Summary

| Phase | Focus | Key Output |
|---|---|---|
| 1 | Data acquisition | Downloaded, refined CSV datasets in `data/base/` |
| 2 | Feature engineering | `FeatureVector` extractor, < 5ms |
| 3 | Classical ML | XGBoost F1 ≥ 0.90, Vertex AI model |
| 4 | Rule engine | Blocklist + regex, < 1ms fast path |
| 5 | Inference gateway | FastAPI service, multi-model routing |
| 6 | Feedback loop | FP/FN pipeline, category management |
| 7 | GCP deployment | Cloud Run + Terraform + CI/CD; CSV→BQ + Vertex migration |
| 8 | Perf analysis | Tuning notebooks, monitoring dashboard |

---

## Open Questions / Decisions To Make

- **LLM provider**: Gemini Pro (cheaper, stays on GCP) vs. Claude API (better reasoning) — Phase 5 should implement both and compare in Phase 8.
- **Online vs. batch retraining**: nightly batch is simpler; consider online learning (Vowpal Wabbit) if feedback volume is high.
- **URL features with network calls**: WHOIS and DNS lookups add 100-500ms. Default off; enable via `ENABLE_NETWORK_FEATURES=true` and route to async enrichment queue.
- **Private URL handling**: some URLs may be internal or sensitive. Consider URL anonymization (hash the path, keep domain) before logging to BQ.
