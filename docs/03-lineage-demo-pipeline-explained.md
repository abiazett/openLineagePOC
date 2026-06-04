# Lineage Demo Pipeline - End-to-End ML Lineage POC

## What is This?

This is a **proof-of-concept** demonstrating end-to-end data lineage tracking across a realistic ML pipeline. It answers the question: *"Can we use OpenLineage + Marquez to track data as it flows from raw files through ETL, a feature store, model training, and into production inference?"*

The answer is yes, with some caveats documented in the lessons learned.

## The Use Case: Customer Churn Prediction

The pipeline predicts which customers will cancel their service (churn). It processes synthetic customer data through a full ML lifecycle:

```
Raw CSV (MinIO)
    │
    ▼
ETL: Extract, Clean, Normalize
    │
    ▼
PostgreSQL Warehouse (customer_features table)
    │
    ▼
Feast Feature Store (offline: PG, online: Redis)
    │
    ▼
ML Pipeline (6 steps):
  1. Data extraction (point-in-time join from Feast)
  2. Data validation (Great Expectations)
  3. Feature engineering (derived metrics)
  4. Model training (XGBoost + MLflow logging)
  5. Evaluation (ROC-AUC, F1, precision, recall)
  6. Model registration (promote to "champion" if AUC >= 0.70)
    │
    ▼
FastAPI Inference Service (Feast online features + MLflow model)
```

## Technology Stack

| Component | Technology | What It Does |
|-----------|-----------|--------------|
| **Object Storage** | MinIO (S3-compatible) | Stores raw CSV, MLflow model artifacts |
| **Data Warehouse** | PostgreSQL 15 | Stores cleaned `customer_features` table |
| **Feature Store** | Feast | Defines features, does point-in-time joins, materializes to Redis |
| **Online Store** | Redis 7 | Low-latency feature cache for inference |
| **ML Framework** | XGBoost + scikit-learn | Binary churn classifier |
| **Experiment Tracking** | MLflow | Logs parameters, metrics, model artifacts |
| **Inference** | FastAPI | HTTP API for real-time predictions |
| **Data Validation** | Great Expectations | Schema and quality checks |
| **Lineage Tracking** | OpenLineage + Marquez | Tracks data flow across all stages |
| **Orchestration (local)** | Shell scripts | `run_all.sh` runs stages sequentially |
| **Orchestration (cluster)** | Kubeflow Pipelines v2 | Declarative pipeline on OpenShift AI |

## Project Structure

```
lineage-demo-pipeline/
├── configs/settings.py            # Centralized env-driven configuration
├── data/
│   ├── generate_dataset.py        # Generates 2,000-row synthetic CSV
│   └── customers.csv              # The generated data
├── src/
│   ├── etl/
│   │   ├── run_etl.py             # Local ETL orchestrator
│   │   ├── extract.py             # Download CSV from MinIO
│   │   ├── transform.py           # Clean, deduplicate, normalize
│   │   ├── load.py                # Write to PostgreSQL
│   │   ├── spark_etl.py           # PySpark version (with OL Spark listener)
│   │   └── spark_etl_native_lineage.py  # Spark + explicit OL events
│   ├── feature_store/
│   │   ├── definitions.py         # Feast entity + feature view definitions
│   │   └── feast_workflow.py      # Apply schema, materialize features
│   ├── pipeline/
│   │   ├── run_pipeline.py        # Local ML pipeline runner (6 steps)
│   │   ├── components.py          # The 6 pipeline step implementations
│   │   ├── kfp_pipeline.py        # Kubeflow Pipelines v2 DSL version
│   │   └── upload_pipeline.py     # Upload compiled pipeline to DSPA
│   ├── training/
│   │   ├── trainer.py             # XGBoost training + MLflow logging
│   │   └── registry.py            # Model registration + promotion
│   └── serving/
│       └── app.py                 # FastAPI inference service
├── openlineage-oai/               # Custom OpenLineage adapters
│   └── adapters/
│       ├── kfp/lineage.py         # KFP component context manager
│       ├── mlflow/tracking_store.py  # MLflow OL adapter
│       └── mlflow/dataset_source.py  # Training dataset lineage
├── docs/
│   ├── intro-and-lessons-learned.md  # Critical: naming conventions & findings
│   ├── ARCHITECTURE.md               # Architecture overview
│   ├── openlineage-integration-findings.md  # Integration details
│   └── pipeline-walkthrough.md       # Step-by-step guide
├── openshift/                     # OpenShift AI deployment manifests
├── scripts/
│   ├── start_services.sh          # Docker Compose up + health checks
│   ├── run_all.sh                 # Run all pipeline stages
│   └── test_inference.sh          # Test the prediction API
├── docker-compose.yml             # Local infrastructure
├── Dockerfile                     # Main app image
├── Dockerfile.api                 # Inference service image
├── Dockerfile.mlflow              # MLflow server image
├── Dockerfile.spark               # PySpark + OpenLineage image
└── requirements.txt               # Python dependencies
```

## How Each Stage Works

### Stage 1: ETL (`src/etl/`)

**Local version** (`run_etl.py`):
1. `extract.py` -- Downloads `customers.csv` from MinIO using the MinIO Python client
2. `transform.py` -- Deduplicates rows, coerces data types, imputes missing values (median), applies min-max normalization
3. `load.py` -- Writes cleaned data to PostgreSQL table `customer_features`

**Spark version** (`spark_etl.py`):
- Same logic but using PySpark
- Adds the **OpenLineage Spark listener** (`io.openlineage.spark.agent.OpenLineageSparkListener`)
- Lineage events are emitted automatically when Spark reads from S3 and writes to PostgreSQL

### Stage 2: Feature Store (`src/feature_store/`)

**`definitions.py`** -- Defines Feast entities and feature views:
- Entity: `customer` (keyed by `entity_id`)
- Feature View: `customer_features_view` with features like tenure, charges, tickets, contract type

**`feast_workflow.py`** -- Orchestrates Feast operations:
1. Generates `feature_store.yaml` dynamically (points to PG offline store, Redis online store)
2. `feast apply` -- Registers feature views with Feast
3. `feast materialize` -- Pushes latest features from PG to Redis for online serving

Feast has built-in OpenLineage support -- when enabled, it emits lineage events for materialization jobs.

### Stage 3: ML Pipeline (`src/pipeline/`)

**`components.py`** implements six steps:

1. **Data Extraction** -- Uses Feast `get_historical_features()` for a point-in-time join, producing a training DataFrame
2. **Data Validation** -- Runs Great Expectations checks (nulls, types, value ranges)
3. **Feature Engineering** -- Creates derived features (e.g., charges-per-month), clamps outliers
4. **Model Training** -- Label-encodes categoricals, trains XGBoost, logs everything to MLflow (params, metrics, model artifact)
5. **Evaluation** -- Computes ROC-AUC, F1, precision, recall on holdout set
6. **Model Registration** -- If ROC-AUC >= 0.70, registers the model in MLflow and promotes to "champion" alias

**`run_pipeline.py`** runs these steps sequentially in local Python.

**`kfp_pipeline.py`** defines the same pipeline as KFP v2 components for running on OpenShift AI.

### Stage 4: Inference (`src/serving/`)

**`app.py`** -- FastAPI service with endpoints:
- `GET /health` -- Health check
- `POST /predict` -- Accept entity IDs, fetch features from Feast online store (Redis), run through MLflow model, return churn probability
- `POST /reload-model` -- Hot-reload the model from MLflow registry

## OpenLineage Integration Details

### How Lineage Is Tracked

Each tool has a different integration approach:

| Tool | Integration Method | How It Works |
|------|-------------------|--------------|
| **Spark** | Native listener | JVM agent auto-captures reads/writes from query plans |
| **Feast** | Built-in support | Feast emits OL events during `apply` and `materialize` |
| **MLflow** | Custom adapter | `openlineage-oai/adapters/mlflow/` wraps MLflow tracking store |
| **KFP** | Custom context manager | `openlineage-oai/adapters/kfp/` emits events per component |
| **Great Expectations** | Not yet integrated | Validation results not tracked via OL |

### The Custom Adapters (`openlineage-oai/`)

This repo includes purpose-built adapters:

**KFP Adapter** (`adapters/kfp/lineage.py`):
- Context manager that wraps KFP component execution
- Emits START event on enter, COMPLETE/FAIL on exit
- Declares input/output datasets with schemas

**MLflow Adapter** (`adapters/mlflow/tracking_store.py`):
- Custom tracking URI scheme: `openlineage+http://mlflow:5000`
- Intercepts MLflow logging calls to emit OL events
- Links model artifacts to training datasets

### Naming Convention Lessons (Critical)

From `docs/intro-and-lessons-learned.md`:

**Dataset identity = `(namespace, name)` is the correlation key.** If two jobs use different namespace/name for the same physical dataset, Marquez sees them as separate -- lineage is broken.

Recommended conventions:
```
PostgreSQL:  namespace = "postgres://host:5432"    name = "database.schema.table"
S3/MinIO:    namespace = "s3://bucket"             name = "path/to/object"
Local file:  namespace = "file"                    name = "/path/to/file"
```

**Gotchas discovered**:
1. **Scheme normalization** -- `postgresql://` vs `postgres://` breaks correlation. The custom SDK normalizes to `postgres://`
2. **Spark S3 naming** -- Spark uses `s3a://` internally; must be rewritten to `s3://` via config:
   ```
   spark.openlineage.transport.urlParams.replaceDatasetNamespacePattern=s3a://->s3://
   ```
3. **Feast namespace** -- Feast appends its project name to the namespace, creating unexpected identities

## Running Locally

### Prerequisites
- Docker Desktop (4.x+)
- Python 3.10+ (3.11 recommended)

### Steps

```bash
# 1. Clone the repo
git clone https://github.com/rh-waterford-et/lineage-demo-pipeline.git
cd lineage-demo-pipeline

# 2. Start infrastructure (MinIO, PostgreSQL, Redis, MLflow, Inference API)
./scripts/start_services.sh
# Waits for health checks, then prints service URLs

# 3. Create a Python virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# 4. Install dependencies
pip install -r requirements.txt

# 5. Generate synthetic data (optional -- may already exist)
python data/generate_dataset.py

# 6. Run the full pipeline
./scripts/run_all.sh
# Or run individual stages:
./scripts/run_all.sh etl       # ETL only
./scripts/run_all.sh feast     # Feast apply + materialize
./scripts/run_all.sh pipeline  # ML pipeline (6 steps)

# 7. Test inference
./scripts/test_inference.sh
# Or manually:
curl -X POST http://localhost:8000/predict \
  -H 'Content-Type: application/json' \
  -d '{"entity_ids": [1, 2, 3]}'

# 8. Access UIs
# MinIO Console: http://localhost:9001 (minioadmin / minioadmin)
# MLflow UI:     http://localhost:5000
# Inference API: http://localhost:8000/docs (Swagger)
```

### Docker Compose Services

| Service | Port | Credentials |
|---------|------|-------------|
| MinIO (S3) | 9000 (API), 9001 (Console) | minioadmin / minioadmin |
| PostgreSQL | 5432 | feast / feast, DB: warehouse |
| Redis | 6379 | (no auth) |
| MLflow | 5000 | (no auth) |
| Inference API | 8000 | (no auth) |

### What Happens When You Run It

1. **`start_services.sh`** brings up Docker Compose, seeds MinIO with `customers.csv`
2. **`run_all.sh etl`** downloads CSV from MinIO, cleans it, loads into PostgreSQL
3. **`run_all.sh feast`** registers Feast feature views, materializes to Redis
4. **`run_all.sh pipeline`** extracts training data via Feast, validates, engineers features, trains XGBoost, evaluates, registers model in MLflow
5. **Inference API** is already running -- it loads the "champion" model from MLflow and serves predictions using Feast online features

### Note on Lineage Visualization

The local Docker Compose does **not** include Marquez. To see lineage graphs, you would need to:
1. Add Marquez to the docker-compose (API + DB + Web UI)
2. Configure OpenLineage transports to point to `http://marquez:5000`
3. Run the pipeline and view the lineage graph at `http://localhost:3000`

The OpenShift deployment (`openshift/lineage-openshift-ai.yaml`) includes Marquez.

## Key Lessons from the POC

From the team's `docs/intro-and-lessons-learned.md`:

1. **Dataset naming is everything** -- inconsistent `(namespace, name)` breaks lineage. Establish conventions early.
2. **Integration maturity varies** -- Spark listener is production-ready; MLflow/KFP needed custom adapters.
3. **Feast has a quirk** -- it appends its project name to the OL namespace, making cross-tool correlation harder.
4. **Four integration approaches**: native tool integration (best), auto-instrumentation (fragile), SDK (manual), pull/polling (avoid).
5. **Marquez limitations** -- the POC raised questions about data isolation and multi-tenancy for production use.
6. **Normalization is required** -- `postgresql://` vs `postgres://`, `s3a://` vs `s3://` -- these mismatches silently break lineage.
