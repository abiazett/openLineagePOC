# Marquez - The Lineage Backend Server

## What is Marquez?

Marquez is the **reference implementation** for collecting, storing, and visualizing OpenLineage data. It's an open-source metadata service that:

1. **Receives** OpenLineage events via a REST API (`POST /api/v1/lineage`)
2. **Stores** them in PostgreSQL (raw events + a normalized model)
3. **Builds** a lineage graph connecting datasets and jobs
4. **Exposes** that graph via REST API and a web UI

Think of it as: **OpenLineage defines the language; Marquez is the database and dashboard.**

## Architecture Overview

```
                    OpenLineage Events
                          │
                          ▼
┌─────────────────────────────────────────────┐
│              Marquez API Server              │
│          (Java / Dropwizard 2.1.12)          │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Lineage  │  │ Dataset  │  │   Job    │   │
│  │ Resource │  │ Resource │  │ Resource │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │        │
│  ┌────┴──────────────┴──────────────┴────┐   │
│  │         Service Layer                  │   │
│  │  (OpenLineageService, RunService, ...) │   │
│  └────────────────┬──────────────────────┘   │
│                   │                          │
│  ┌────────────────┴──────────────────────┐   │
│  │       Data Access Layer (JDBI3)        │   │
│  │   (LineageDao, DatasetDao, JobDao)     │   │
│  └────────────────┬──────────────────────┘   │
└───────────────────┼──────────────────────────┘
                    │
                    ▼
            ┌──────────────┐
            │ PostgreSQL 14│
            │              │
            │ - lineage_   │
            │   events     │  (raw JSONB events)
            │ - datasets   │
            │ - jobs       │  (normalized model)
            │ - runs       │
            │ - job_facets │  (extensible metadata)
            │ - dataset_   │
            │   facets     │
            └──────────────┘

┌─────────────────────────────────────────────┐
│           Marquez Web UI                     │
│        (React + TypeScript + Redux)          │
│                                              │
│  Dashboard | Jobs | Datasets | Lineage Graph │
│           Column-level Lineage               │
│              Event History                   │
│                                              │
│  Port 3000                                   │
└─────────────────────────────────────────────┘
```

## REST API

### The Main Endpoint: Receiving Lineage Events

```
POST /api/v1/lineage
Content-Type: application/json

{
  "eventType": "COMPLETE",
  "eventTime": "2024-01-15T10:30:00Z",
  "run": { "runId": "abc-123" },
  "job": { "namespace": "my-ns", "name": "etl_job" },
  "inputs": [...],
  "outputs": [...],
  "producer": "https://spark"
}
```

This single endpoint accepts all three OpenLineage event types (RunEvent, DatasetEvent, JobEvent). Events are processed asynchronously -- the API returns 201 immediately, then:
1. Stores the raw event as JSONB in `lineage_events`
2. Updates the normalized model (creates/updates datasets, jobs, runs)
3. Triggers search indexing (if OpenSearch is enabled)

### Querying the Lineage Graph

```
GET /api/v1/lineage?nodeId=dataset:my-ns:my-table&depth=20
```

Returns a graph of datasets and jobs connected to the specified node, traversing up to `depth` levels upstream and downstream.

### Full API Endpoint Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| **Lineage** | | |
| `POST` | `/api/v1/lineage` | Receive OpenLineage events |
| `GET` | `/api/v1/lineage?nodeId=...&depth=20` | Get lineage graph |
| `GET` | `/api/v1/events/lineage` | List lineage events (paginated) |
| **Namespaces** | | |
| `PUT` | `/api/v1/namespaces/{ns}` | Create/update namespace |
| `GET` | `/api/v1/namespaces/{ns}` | Get namespace |
| `GET` | `/api/v1/namespaces` | List namespaces |
| **Datasets** | | |
| `GET` | `/api/v1/namespaces/{ns}/datasets/{ds}` | Get dataset details |
| `GET` | `/api/v1/namespaces/{ns}/datasets/{ds}/versions` | List dataset versions |
| `POST` | `/api/v1/namespaces/{ns}/datasets/{ds}/tags/{tag}` | Tag a dataset |
| **Jobs** | | |
| `GET` | `/api/v1/namespaces/{ns}/jobs/{job}` | Get job details |
| `GET` | `/api/v1/namespaces/{ns}/jobs/{job}/runs` | List job runs |
| **Runs** | | |
| `GET` | `/api/v1/jobs/runs/{runId}` | Get run details |
| **Sources** | | |
| `PUT` | `/api/v1/sources/{source}` | Register a data source |
| `GET` | `/api/v1/sources` | List data sources |
| **Search** | | |
| `GET` | `/api/v1/search?q=...` | Full-text search |
| **GraphQL** | | |
| `POST` | `/api/v2beta/graphql` | GraphQL endpoint (beta) |

## Database Schema

### Core Tables

**`lineage_events`** -- Raw event storage:
```sql
CREATE TABLE lineage_events (
    event_time  TIMESTAMP,
    event       JSONB,         -- the full OpenLineage event
    event_type  VARCHAR,       -- START, COMPLETE, FAIL, etc.
    run_id      UUID,
    job_name    VARCHAR,
    job_namespace VARCHAR,
    producer    VARCHAR
);
```

**`datasets`** -- Normalized dataset records:
```sql
-- Fields: uuid, type, name, namespace, source, description, created_at, updated_at
```

**`jobs`** -- Normalized job records:
```sql
-- Fields: uuid, type, name, namespace, description, created_at, updated_at
```

**`runs`** -- Job execution instances:
```sql
-- Fields: uuid, job_version_uuid, run_args_uuid, created_at, updated_at
```

**`run_states`** -- State transitions:
```sql
-- Fields: uuid, transitioned_at, run_uuid, state (NEW, RUNNING, COMPLETED, FAILED, ABORTED)
```

**Facet tables** (JSONB for extensibility):
- `job_facets` -- Job-level metadata
- `dataset_facets` -- Dataset-level metadata  
- `run_facets` -- Run-level metadata
- `column_lineage` -- Column-level lineage tracking

**Versioning tables**:
- `dataset_versions` -- Dataset schema versions
- `job_versions` -- Job definition versions
- `job_versions_io_mapping` -- Input/output mapping per job version

Migrations are managed by **Flyway** (`api/src/main/resources/marquez/db/migration/`).

## Web UI

Built with **React 19 + TypeScript + Redux + Material-UI**.

### Views:
- **Dashboard** -- Overview of recent jobs and datasets
- **Jobs** -- Browse jobs by namespace, see run history
- **Datasets** -- Browse datasets, see schema and versions
- **Table-level Lineage** -- Interactive graph showing dataset-to-job-to-dataset connections
- **Column-level Lineage** -- Drill into which output columns derive from which input columns
- **Events** -- Raw event history viewer

### Tech Stack:
- State management: Redux + Redux-Saga
- Routing: React Router v6
- Charts: D3.js, MUI X Charts
- i18n: i18next

## Running Marquez Locally

### Quick Start (Docker)

```bash
cd marquez
./docker/up.sh
```

This starts:
- **Marquez API** on `http://localhost:5000` (admin on 5001)
- **PostgreSQL 14** on port 5432

To include the web UI:
```bash
./docker/up.sh --web
```
- **Marquez Web UI** on `http://localhost:3000`

To seed with sample data:
```bash
./docker/up.sh --seed
```

### Docker Compose Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `api` | `marquezproject/marquez` | 5000, 5001 | API server |
| `db` | `postgres:14` | 5432 | Database |
| `web` | `marquezproject/marquez-web` | 3000 | Web UI (optional) |

### Configuration

`marquez.example.yml`:
```yaml
server:
  applicationConnectors:
    - type: http
      port: 8080
  adminConnectors:
    - type: http
      port: 8081

db:
  driverClass: org.postgresql.Driver
  url: jdbc:postgresql://localhost:5432/marquez
  user: marquez
  password: marquez

graphql:
  enabled: true

search:
  enabled: false  # set true + add OpenSearch for full-text search
```

## Marquez Java & Python Clients

### Java Client

```java
MarquezClient client = Clients.newClient("http://localhost:5000");

// List namespaces
List<Namespace> namespaces = client.listNamespaces();

// Get a specific dataset
Dataset dataset = client.getDataset("my-namespace", "my-dataset");

// Get lineage
LineageGraph graph = client.getLineage("dataset:ns:name", 10);
```

### Python Client

```python
from marquez_client import MarquezClient

client = MarquezClient(url="http://localhost:5000")

# List namespaces
namespaces = client.list_namespaces()

# Get dataset details
dataset = client.get_dataset("my-namespace", "my-dataset")
```

## How OpenLineage Events Become Lineage

The flow from event to graph:

```
1. Spark job completes, emits RunEvent:
   job: "etl.transform"
   inputs: [postgres://db/raw.customers]
   outputs: [postgres://db/clean.customer_features]

2. dbt model runs, emits RunEvent:
   job: "dbt.customer_summary"
   inputs: [postgres://db/clean.customer_features]
   outputs: [postgres://db/analytics.customer_summary]

3. Marquez stores both events and builds:

   raw.customers ──> etl.transform ──> customer_features ──> dbt.customer_summary ──> customer_summary
   (dataset)         (job)              (dataset)             (job)                    (dataset)
```

The key insight: **datasets are linked across jobs by their `(namespace, name)` identity**. When one job outputs a dataset and another job reads it, Marquez automatically connects them in the lineage graph.

## Key Takeaways

1. **Marquez is the reference backend** for OpenLineage -- it's not the only option, but it's the most feature-complete open-source one
2. **PostgreSQL + JSONB** stores both raw events and a normalized model for fast querying
3. **The Web UI** provides interactive lineage visualization, including column-level lineage
4. **Docker-first** -- `./docker/up.sh` gets you running in seconds
5. **Production considerations** -- Marquez supports OpenSearch for full-text search, Prometheus metrics, Sentry error tracking, and Kubernetes Helm charts
6. **The API is simple** -- `POST /api/v1/lineage` to send events, `GET /api/v1/lineage` to query the graph
