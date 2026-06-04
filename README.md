# OpenLineage POC

An end-to-end proof-of-concept for data lineage tracking across an ML pipeline using [OpenLineage](https://openlineage.io/) and [Marquez](https://marquezproject.ai/).

## What's in this repo

This repo ties together three upstream projects via git submodules:

| Directory | Upstream Repo | Role |
|-----------|--------------|------|
| `OpenLineage/` | [OpenLineage/OpenLineage](https://github.com/OpenLineage/OpenLineage) | The lineage standard, Python/Java clients, Spark/Airflow/dbt integrations |
| `marquez/` | [MarquezProject/marquez](https://github.com/MarquezProject/marquez) | Lineage backend: REST API, PostgreSQL storage, web UI |
| `lineage-demo-pipeline/` | [rh-waterford-et/lineage-demo-pipeline](https://github.com/rh-waterford-et/lineage-demo-pipeline) | Customer churn prediction pipeline with OL tracking |

Plus original content:

| Directory | Contents |
|-----------|----------|
| `docs/` | Explanation docs covering each project and how they connect |
| `patches/` | Local fixes needed to run lineage-demo-pipeline on macOS |

## The demo pipeline

The lineage-demo-pipeline implements a customer churn prediction workflow:

```
MinIO (raw CSV) → ETL → PostgreSQL → Feast feature store → XGBoost training → MLflow → FastAPI inference
```

Each stage emits OpenLineage events that can be visualized as a lineage graph in Marquez.

## Quick start

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/abiazett/openLineagePOC.git
cd openLineagePOC
```

### 2. Apply local fixes

The patch fixes MLflow DNS rebinding issues (macOS port conflict on 5000) and adds a Feast config fix for Docker networking:

```bash
cd lineage-demo-pipeline
git apply ../patches/local-fixes.patch
```

### 3. Start infrastructure

```bash
./scripts/start_services.sh
```

This brings up MinIO, PostgreSQL, Redis, MLflow, and the inference API.

### 4. Set environment variables

```bash
export MINIO_SECRET_KEY=minioadmin
export MLFLOW_TRACKING_URI=http://localhost:5050
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export MLFLOW_S3_ENDPOINT_URL=http://localhost:9000
```

### 5. Run the pipeline

```bash
# Create a venv and install deps
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -e "../openlineage-oai[mlflow]"  # Custom OL adapters

# Generate synthetic data (if not present)
python data/generate_dataset.py

# Run all stages
./scripts/run_all.sh
```

### 6. Test inference

```bash
curl -X POST http://localhost:8080/predict \
  -H 'Content-Type: application/json' \
  -d '{"entity_ids": [1, 2, 3]}'
```

### 7. Access UIs

| Service | URL | Credentials |
|---------|-----|-------------|
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin |
| MLflow | http://localhost:5050 | (none) |
| Inference API (Swagger) | http://localhost:8080/docs | (none) |
| Marquez Web (lineage) | http://localhost:3000 | (none) |
| Marquez API | http://localhost:5002 | (none) |

### 8. Start Marquez (lineage backend)

Marquez services are included in the docker-compose and start automatically with the other infrastructure. Verify they're running:

```bash
curl http://localhost:5002/api/v1/namespaces
```

### 9. Emit lineage events

```bash
python scripts/emit_lineage.py
```

This sends OpenLineage events for every pipeline stage to Marquez, creating a full lineage graph.

### 10. View lineage

Open http://localhost:3000, select the `demo-pipeline` namespace, and click any job to see its inputs, outputs, and run history.

## What the patch fixes

The `patches/local-fixes.patch` addresses these issues when running locally on macOS:

1. **MLflow port conflict** -- macOS AirPlay uses port 5000; remapped to 5050
2. **MLflow DNS rebinding** -- MLflow 3.10+ rejects non-localhost Host headers; adds `--allowed-hosts`
3. **Feast Docker networking** -- `feature_store.yaml` uses `localhost` which doesn't resolve inside containers; sed patches hostnames at startup
4. **Marquez services** -- Adds marquez-db, marquez-api, and marquez-web to docker-compose for lineage visualization
5. **Lineage emission script** -- Adds `scripts/emit_lineage.py` to populate Marquez with pipeline lineage events

## Documentation

See `docs/` for detailed explanations:

- [01 - OpenLineage Explained](docs/01-openlineage-explained.md) -- The lineage standard
- [02 - Marquez Explained](docs/02-marquez-explained.md) -- The lineage backend
- [03 - Lineage Demo Pipeline Explained](docs/03-lineage-demo-pipeline-explained.md) -- The POC pipeline
- [04 - How They Connect](docs/04-how-they-connect.md) -- How all three fit together
