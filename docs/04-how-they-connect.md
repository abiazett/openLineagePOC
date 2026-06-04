# How These Three Projects Connect

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     OpenLineage (the standard)                      │
│                                                                     │
│  Defines: RunEvent, DatasetEvent, JobEvent, Facets, JSON Schema     │
│  Provides: Python client, Java client, Spark/Airflow/dbt/Flink     │
│            integrations                                             │
│                                                                     │
│  Repo: github.com/OpenLineage/OpenLineage                          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                   Events emitted in OL format
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Marquez (the backend)                          │
│                                                                     │
│  Receives: OpenLineage events via POST /api/v1/lineage              │
│  Stores:   PostgreSQL (raw JSONB + normalized model)                │
│  Exposes:  REST API + Web UI for lineage graph visualization        │
│                                                                     │
│  Repo: github.com/MarquezProject/marquez                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                  Used as lineage backend by
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              lineage-demo-pipeline (the POC)                        │
│                                                                     │
│  Demonstrates: Full ML pipeline with OL tracking                    │
│  Uses:   MinIO -> ETL -> PostgreSQL -> Feast -> XGBoost -> MLflow   │
│  Tracks: Lineage via Spark listener, Feast OL, MLflow adapter       │
│  Sends:  Events to Marquez for visualization                        │
│                                                                     │
│  Repo: github.com/rh-waterford-et/lineage-demo-pipeline            │
└─────────────────────────────────────────────────────────────────────┘
```

## Relationship Summary

| Project | Role | Analogy |
|---------|------|---------|
| **OpenLineage** | The specification + SDKs + integrations | Like HTTP spec + curl + browser plugins |
| **Marquez** | The lineage database + API + UI | Like a web server that stores and serves the data |
| **lineage-demo-pipeline** | A real-world application using both | Like a web app that uses HTTP and a web server |

## Data Flow Through All Three

```
1. lineage-demo-pipeline runs a Spark ETL job
       │
       ├── Spark uses OpenLineage's Spark listener (from OpenLineage repo)
       │   to auto-detect input/output datasets
       │
       ├── The listener creates RunEvents using OpenLineage's JSON schema
       │   (from OpenLineage repo's spec/)
       │
       └── Events are sent via OpenLineage's HTTP transport
           to Marquez's POST /api/v1/lineage endpoint
                │
                └── Marquez stores the event and updates its lineage graph
                    │
                    └── You view the graph in Marquez's React web UI
```

## What You Need to Run Everything

For the **full experience** (pipeline + lineage visualization):

```
lineage-demo-pipeline services:
  ├── MinIO          (object storage)
  ├── PostgreSQL     (data warehouse + Feast offline store)
  ├── Redis          (Feast online store)
  ├── MLflow         (experiment tracking)
  └── Inference API  (FastAPI predictions)

Marquez services (add separately):
  ├── Marquez API    (receives OL events, port 5000)
  ├── PostgreSQL     (Marquez's own database)
  └── Marquez Web    (lineage visualization, port 3000)

OpenLineage:
  └── (used as libraries -- pip install openlineage-python,
       Spark JAR for the listener, Feast built-in support)
```

## Quick Reference: Key Files Across Repos

| What | Where |
|------|-------|
| **OpenLineage JSON Schema** | `OpenLineage/spec/OpenLineage.json` |
| **Standard Facets** | `OpenLineage/spec/facets/*.json` |
| **Python client** | `OpenLineage/client/python/` |
| **Spark listener** | `OpenLineage/integration/spark/` |
| **Marquez API server** | `marquez/api/src/main/java/marquez/` |
| **Marquez Web UI** | `marquez/web/src/` |
| **Marquez Docker setup** | `marquez/docker-compose.yml` + `marquez/docker/up.sh` |
| **Demo ETL code** | `lineage-demo-pipeline/src/etl/` |
| **Demo ML pipeline** | `lineage-demo-pipeline/src/pipeline/components.py` |
| **Demo OL adapters** | `lineage-demo-pipeline/openlineage-oai/` |
| **Lessons learned** | `lineage-demo-pipeline/docs/intro-and-lessons-learned.md` |
| **Demo Docker setup** | `lineage-demo-pipeline/docker-compose.yml` |
