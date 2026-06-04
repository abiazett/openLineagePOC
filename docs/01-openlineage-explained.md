# OpenLineage - The Open Standard for Data Lineage

## What is OpenLineage?

OpenLineage is an **open standard** for collecting and analyzing data lineage. It defines a common JSON-based format for how data pipelines report what they read, what they write, and the jobs that connect them. Think of it as "OpenTelemetry, but for data pipelines" -- it gives every tool in your data stack a shared language to describe data flow.

It is an Apache 2.0 licensed project under the **LF AI & Data Foundation**.

## Why Does It Exist?

Before OpenLineage, every tool (Spark, Airflow, dbt, Flink) had its own way of tracking lineage -- or none at all. If you wanted a unified view of data flowing from an S3 file through Spark into a database and then into a dbt model, you had to build custom integrations for each tool. OpenLineage solves this by providing:

1. **A single spec** -- one JSON schema that all tools can emit
2. **Pre-built integrations** -- plugins for Spark, Airflow, dbt, Flink, etc.
3. **Language clients** -- Python, Java, and Go SDKs to emit events from custom code
4. **Transport flexibility** -- send events over HTTP, Kafka, to files, or cloud services

## The Core Concept: Events and Facets

### Events

Everything revolves around three event types:

```
RunEvent     -- "Job X started/completed/failed, reading datasets A and B, writing dataset C"
DatasetEvent -- "Dataset D has this schema" (independent of any job)
JobEvent     -- "Job X reads from A, B and writes to C" (independent of any run)
```

The **RunEvent** is the most common. It captures the lifecycle of a job execution:

```
START   --> RUNNING --> COMPLETE
                   \--> FAIL
                   \--> ABORT
```

### The RunEvent Structure

```json
{
  "eventType": "COMPLETE",
  "eventTime": "2024-01-15T10:30:00Z",
  "producer": "https://my-spark-cluster",
  "schemaURL": "https://openlineage.io/spec/2-0-2/OpenLineage.json#/definitions/RunEvent",
  "run": {
    "runId": "a1b2c3d4-...",
    "facets": { ... }
  },
  "job": {
    "namespace": "my-spark-namespace",
    "name": "etl.customer_transform",
    "facets": { ... }
  },
  "inputs": [
    {
      "namespace": "s3://raw-data-bucket",
      "name": "customers.csv",
      "facets": { ... }
    }
  ],
  "outputs": [
    {
      "namespace": "postgres://warehouse:5432",
      "name": "public.customer_features",
      "facets": {
        "schema": {
          "fields": [
            { "name": "customer_id", "type": "INTEGER" },
            { "name": "tenure_months", "type": "FLOAT" }
          ]
        }
      }
    }
  ]
}
```

### Facets: Extensible Metadata

Facets are atomic pieces of metadata attached to runs, jobs, or datasets. The spec defines 40+ standard facets:

| Category | Examples |
|----------|----------|
| **Run facets** | `nominalTime`, `parent` (parent job/run), `errorMessage`, `processingEngine` |
| **Job facets** | `sourceCodeLocation` (git repo), `sql` (the query), `ownership`, `jobType` |
| **Dataset facets** | `schema` (field names/types), `columnLineage`, `dataSource`, `dataQualityMetrics` |
| **Input facets** | `dataQualityMetrics` (row count, null count), `inputStatistics` |
| **Output facets** | `outputStatistics` (rows written, bytes written) |

You can also create **custom facets** for domain-specific metadata.

### Dataset Identity: The Correlation Key

A dataset is uniquely identified by `(namespace, name)`:

| Storage Type | Namespace | Name |
|-------------|-----------|------|
| PostgreSQL | `postgres://host:5432` | `database.schema.table` |
| S3 | `s3://bucket-name` | `path/to/object` |
| Local file | `file` | `/path/to/file.csv` |

This identity is how Marquez (or any consumer) links the output of one job to the input of another, forming the lineage graph.

## Repository Structure

```
OpenLineage/
├── spec/               # The JSON Schema specification (OpenLineage.json, facets/)
├── client/
│   ├── python/         # Python SDK (pip install openlineage-python)
│   ├── java/           # Java SDK (Maven: io.openlineage:openlineage-java)
│   └── go/             # Go SDK
├── integration/
│   ├── spark/          # Apache Spark listener (auto-tracks reads/writes)
│   ├── flink/          # Apache Flink integration
│   ├── dbt/            # dbt wrapper (dbt-ol run)
│   ├── hive/           # Apache Hive integration
│   ├── sql/            # SQL parser for column-level lineage
│   └── common/         # Shared code across integrations
├── proxy/fluentd/      # Fluentd plugin for validation and routing
├── website/            # openlineage.io documentation
└── proposals/          # Enhancement proposals
```

## The Python Client

**Install**: `pip install openlineage-python`

### Basic Usage

```python
from openlineage.client import OpenLineageClient
from openlineage.client.run import RunEvent, RunState, Run, Job, Dataset
from openlineage.client.uuid import generate_new_uuid

# Create client -- reads config from openlineage.yml or env vars
client = OpenLineageClient(url="http://marquez:5000")

# Build a RunEvent
event = RunEvent(
    eventType=RunState.COMPLETE,
    eventTime="2024-01-15T10:30:00Z",
    run=Run(runId=str(generate_new_uuid())),
    job=Job(namespace="my-namespace", name="my-etl-job"),
    inputs=[Dataset(namespace="s3://bucket", name="raw/data.csv")],
    outputs=[Dataset(namespace="postgres://db:5432", name="public.clean_data")],
    producer="https://my-pipeline"
)

# Send it
client.emit(event)
```

### Configuration

Configure via YAML file (`openlineage.yml`) or environment variables:

```yaml
transport:
  type: http
  url: http://marquez:5000
  auth:
    type: api_key
    apiKey: "my-api-key"
```

Or environment variables:
```bash
export OPENLINEAGE__TRANSPORT__TYPE=http
export OPENLINEAGE__TRANSPORT__URL=http://marquez:5000
```

### Transport Options

| Transport | Use Case |
|-----------|----------|
| `http` | Send to Marquez or any HTTP endpoint |
| `kafka` | High-throughput event streaming |
| `console` | Debugging (prints to stdout) |
| `file` | Write JSON-L to local/cloud files |
| `composite` | Fan-out to multiple backends |
| `gcplineage` | Google Cloud Data Catalog Lineage |
| `datadog` | Datadog APM integration |
| `amazon_datazone` | AWS DataZone |

## The Java Client

**Maven**: `io.openlineage:openlineage-java`

Similar API to the Python client but with a builder pattern:

```java
OpenLineageClient client = Clients.newClient();
client.emit(
    RunEvent.builder()
        .eventType(RunState.COMPLETE)
        .run(new Run(UUID.randomUUID()))
        .job(new Job("my-namespace", "my-job"))
        .build()
);
```

Features circuit-breaker pattern and Micrometer metrics for production use.

## Integrations: How They Hook In

### Apache Spark Integration

The Spark integration is the most mature. It works by adding a **Spark listener** that automatically intercepts query execution plans:

```bash
spark-submit \
  --jars openlineage-spark.jar \
  --conf spark.extraListeners=io.openlineage.spark.agent.OpenLineageSparkListener \
  --conf spark.openlineage.transport.type=http \
  --conf spark.openlineage.transport.url=http://marquez:5000 \
  my_job.py
```

What it captures automatically:
- Input/output datasets from the query plan
- Schema information
- Column-level lineage (which output columns derive from which input columns)
- Run lifecycle events (START, COMPLETE, FAIL)

Supports Spark 2.x through 4.0 with version-specific modules.

### Apache Airflow Integration

Maintained in the official Apache Airflow project. Operators that are instrumented emit OpenLineage events when tasks execute. Supported operators include PostgresOperator, BigQueryOperator, S3 operators, and many more.

### dbt Integration

A wrapper command that replaces `dbt run`:

```bash
pip install openlineage-dbt
dbt-ol run    # instead of: dbt run
```

Parses `manifest.json` and run results to emit lineage events. Supports 13+ database adapters (BigQuery, Snowflake, Postgres, etc.).

### Apache Flink Integration

Similar to Spark -- a custom connector that captures input/output metadata from Flink job execution graphs.

## How It All Fits Together

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Spark   │    │ Airflow  │    │   dbt    │
│ Listener │    │ Provider │    │ Wrapper  │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     │  OpenLineage RunEvents (JSON)  │
     │               │               │
     └───────┬───────┴───────┬───────┘
             │               │
             ▼               ▼
      ┌────────────┐  ┌────────────┐
      │  HTTP API  │  │   Kafka    │
      │ (Marquez)  │  │  (stream)  │
      └────────────┘  └────────────┘
             │
             ▼
      ┌────────────┐
      │  Marquez   │  <-- stores events, builds lineage graph
      │  Web UI    │  <-- visualizes the graph
      └────────────┘
```

## Key Takeaways

1. **OpenLineage is a spec, not a product** -- it defines the event format; Marquez is the reference backend
2. **Dataset identity (`namespace + name`) is the correlation key** -- this is how lineage is connected across jobs
3. **Facets make it extensible** -- standard facets cover common cases; custom facets handle domain-specific needs
4. **Integrations do the heavy lifting** -- you usually don't write events manually; the Spark listener, Airflow provider, or dbt wrapper does it for you
5. **Transport-agnostic** -- the same events can go to HTTP, Kafka, files, or cloud services
