# Data Engineering Workflows

End-to-end workflow patterns for building, orchestrating, and operating data pipelines. Use this as the playbook for translating a business requirement into a running pipeline.

## 1. Workflow Selection: Batch, Micro-Batch, or Streaming

Before writing any code, classify the workload:

| Pattern | Typical latency | Tooling | Example use case |
|---|---|---|---|
| **Batch** | Hours–days | Airflow + Spark/dbt on Snowflake/BigQuery | Nightly financial close, daily marketing attribution |
| **Micro-batch** | 1–15 min | Structured Streaming (trigger interval), Spark on Kubernetes, Flink windowed batch | Near-real-time dashboards, fraud scoring with tolerance for small delay |
| **Streaming (continuous)** | Sub-second–seconds | Kafka + Flink / Kafka Streams / Spark Structured Streaming (continuous mode) | Real-time personalization, alerting, clickstream session tracking |

**Decision heuristic** (mirrors the skill's architecture framework): if the consumer can act on data that's minutes old, prefer micro-batch — it is operationally simpler than continuous streaming and cheaper than provisioning always-on stream processors. Reserve true continuous streaming for cases where sub-second reaction time has measurable business value (fraud blocking, bidding, safety systems).

## 2. The Canonical Batch Workflow (Extract → Land → Transform → Publish → Validate)

```
Source Systems         Landing (raw)         Transform             Serving
──────────────         ─────────────         ─────────             ───────
Postgres CDC     ┐                     ┌──> bronze (raw)     ┐
Salesforce API   ├──> S3/GCS (parquet) ┤                     ├──> silver (cleaned) ──> gold (marts) ──> BI/ML
Event logs       ┘     landing/dt=...  └──> dbt staging      ┘
```

Airflow DAG skeleton implementing this:

```python
from airflow.decorators import dag, task
from airflow.providers.amazon.aws.transfers.sql_to_s3 import SqlToS3Operator
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator
from pendulum import datetime

@dag(
    schedule="0 5 * * *",
    start_date=datetime(2026, 1, 1),
    catchup=False,
    max_active_runs=1,
    default_args={"retries": 3, "retry_delay": 300},
    tags=["sales", "batch"],
)
def sales_daily_pipeline():

    extract = SqlToS3Operator(
        task_id="extract_postgres_sales",
        sql_conn_id="postgres_sales",
        query="SELECT * FROM sales WHERE updated_at >= '{{ ds }}'",
        s3_bucket="lake-raw",
        s3_key="sales/dt={{ ds }}/sales.parquet",
        file_format="parquet",
    )

    spark_transform = KubernetesPodOperator(
        task_id="spark_bronze_to_silver",
        image="myrepo/spark-etl:2.4",
        cmds=["spark-submit", "jobs/bronze_to_silver.py", "--date", "{{ ds }}"],
    )

    @task.bash
    def run_dbt():
        return "dbt run --select tag:sales --vars '{run_date: {{ ds }}}'"

    @task.bash
    def run_dbt_tests():
        return "dbt test --select tag:sales"

    @task
    def check_freshness_sla():
        # raise AirflowFailException if row count / max(updated_at) is stale
        ...

    extract >> spark_transform >> run_dbt() >> run_dbt_tests() >> check_freshness_sla()

sales_daily_pipeline()
```

Key workflow disciplines:
- **Idempotency**: every task should be safely re-runnable (`INSERT OVERWRITE PARTITION`, `MERGE`, or dbt `incremental` with a unique key) — Airflow *will* retry and backfill.
- **Partitioning by logical date** (`ds`), not wall-clock time, so backfills reprocess the right slice.
- **`catchup=False` + explicit backfill command** (`airflow dags backfill`) rather than relying on scheduler catchup for large historical loads.
- **SLA + freshness check as an explicit terminal task**, not an afterthought — alert if `max(event_time)` lags more than N hours.

## 3. The Canonical Streaming Workflow (Kafka → Flink/Spark → Sink)

```
Producers → Kafka topic (partitioned by key) → Stream processor (stateful) → Sink
                                                     │
                                          checkpoint to S3/RocksDB
```

Spark Structured Streaming example (micro-batch under the hood, exactly-once with idempotent sink):

```python
df = (spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "broker:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", "latest")
      .option("maxOffsetsPerTrigger", 200000)
      .load())

parsed = (df.select(from_json(col("value").cast("string"), order_schema).alias("d"))
            .select("d.*")
            .withWatermark("event_time", "10 minutes"))

agg = (parsed
       .groupBy(window("event_time", "5 minutes"), "region")
       .agg(sum("amount").alias("revenue")))

(agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://lake/chk/orders_revenue")
    .outputMode("update")
    .trigger(processingTime="1 minute")
    .toTable("gold.revenue_by_region_5min"))
```

Design points:
- **Watermarking** (`withWatermark`) bounds state size and defines how long to wait for late data before finalizing a window — set it based on p99 event lateness, not a guess.
- **Checkpointing** to durable storage (S3, HDFS) is what makes restart-after-failure resume exactly where it left off.
- **`maxOffsetsPerTrigger`** rate-limits each micro-batch to avoid a backlog surge overwhelming the cluster on restart.

## 4. Orchestration DAG Design Patterns

**Fan-out/fan-in (parallel extracts, single transform):**
```
extract_orders ─┐
extract_customers ─┼─> merge_raw_layer ─> dbt_run ─> dbt_test ─> publish
extract_products ─┘
```

**Cross-DAG dependency via sensors/datasets (Airflow 2.4+ Datasets, or Dagster Software-Defined Assets):**
```python
orders_dataset = Dataset("s3://lake/silver/orders")

@dag(schedule=[orders_dataset])
def downstream_marts():
    ...
```
Prefer dataset/asset-based scheduling over `ExternalTaskSensor` polling where the orchestrator supports it — it's event-driven, not poll-and-wait, and makes lineage explicit.

**Dagster software-defined asset equivalent:**
```python
from dagster import asset

@asset(deps=[bronze_orders])
def silver_orders(context) -> None:
    ...

@asset(deps=[silver_orders])
def gold_revenue_mart(context) -> None:
    ...
```

**Task-level idempotency + checkpointing pattern for long ETL:**
- Break a large job into resumable stages, each writing a `_SUCCESS` marker or a control-table row (`pipeline_run_id`, `stage`, `status`, `rows_processed`) so a mid-run failure resumes from the last completed stage instead of restarting extract.

## 5. Backfills and Reprocessing

- Always design transforms so a backfill for `2026-06-01` produces identical output whether run on the scheduled date or three months later (no `CURRENT_TIMESTAMP()`-dependent logic baked into historical partitions).
- For dbt: `dbt run --select my_model --vars '{"start_date": "2026-01-01","end_date":"2026-01-31"}'` combined with `is_incremental()` guarded logic keeps backfill and daily-run code paths identical.
- For Spark: prefer `MERGE INTO` (Delta/Iceberg) keyed on business key + partition, so reruns are naturally idempotent rather than `INSERT INTO` which would duplicate.

## 6. Workflow Checklist Before Shipping a Pipeline

1. Idempotent on rerun/backfill?
2. Explicit schedule + SLA + alerting on breach?
3. Retries with backoff configured, and do retries actually help (transient vs. deterministic failure)?
4. Data quality gate (dbt test / Great Expectations) between transform and publish?
5. Lineage visible (dbt docs, OpenLineage, Dagster asset graph)?
6. Backfill procedure documented and tested on a sample range?
