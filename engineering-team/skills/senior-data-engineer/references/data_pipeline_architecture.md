# Data Pipeline Architecture

Architectural patterns for how data moves from source systems to consumption layers, and the tradeoffs behind each pattern.

## 1. Lambda Architecture

Two parallel paths process the same source data: a **batch layer** for completeness/correctness and a **speed layer** for low-latency approximate results, reconciled at serving time.

```
                ┌──> Batch layer (Spark/Airflow, hours) ──> Batch views (authoritative) ──┐
Source (Kafka) ─┤                                                                          ├──> Serving layer (merge)
                └──> Speed layer (Flink/Spark Streaming, seconds) ──> Real-time views ──────┘
```

- **Batch layer**: reprocesses full history nightly, recomputing aggregates from raw immutable log — this is the source of truth.
- **Speed layer**: computes the same aggregates incrementally in near-real-time to fill the gap until the batch layer catches up.
- **Serving layer**: query pattern is "recent from speed view + historical from batch view," e.g., a Druid/Pinot table that swaps in batch-computed segments once available and discards the real-time segment for that period.

**Tradeoff**: you maintain two codebases computing logically the same metric (batch SQL/Spark job and streaming job) — a classic source of "why do batch and real-time numbers disagree" incidents. Justify this complexity only when you truly need both ML-grade historical reprocessing *and* sub-minute freshness.

## 2. Kappa Architecture

Everything is a stream. Historical reprocessing is done by replaying the log from an earlier offset through the same stream processing code, not a separate batch codebase.

```
Kafka (retained log, e.g. 30–90 days or "infinite" via tiered storage)
        │
        ▼
Flink/Kafka Streams job (single codebase)
        │
        ├──> real-time sink (low latency)
        └──> reprocess: same job, replayed from offset 0 → backfilled sink
```

- Requires a log with long/compacted retention (Kafka with tiered storage, or Pulsar) since "batch" == "replay from earlier offset."
- Reprocessing at scale (e.g., re-deriving a year of history) is throughput-bound by the stream engine, which can be slower than a purpose-built Spark batch job — mitigate with parallelism (more partitions) or a scheduled Spark job over the same landed raw topic dump for genuinely large historical rebuilds.
- **Choose Kappa when**: event-sourced domain, one deployable artifact desired, team doesn't want to keep batch and stream logic in sync. **Choose Lambda when**: you need heavyweight batch-only transforms (large joins, ML feature backfills) that aren't practical as streaming operators.

## 3. Medallion Architecture (Bronze / Silver / Gold)

The dominant lakehouse pattern (Databricks-popularized, applies equally on Iceberg/Hudi):

| Layer | Contents | Transformations | Format/engine |
|---|---|---|---|
| **Bronze** | Raw, immutable, as-ingested (including bad records) | None — append-only landing | Delta/Iceberg/Hudi table mirroring source schema |
| **Silver** | Cleaned, deduplicated, conformed types, PII handled | Schema enforcement, dedup, joins to reference data | Delta/Iceberg tables, `MERGE` for CDC application |
| **Gold** | Business-level aggregates, dimensional marts | Aggregation, dimensional modeling, metric definitions | Delta/Iceberg or warehouse-native tables consumed by BI |

```python
# Bronze: append raw, no transformation, preserve everything for replay
raw_df.write.format("delta").mode("append").save("s3://lake/bronze/orders")

# Silver: dedup + type conformance + merge CDC
from delta.tables import DeltaTable
silver = DeltaTable.forPath(spark, "s3://lake/silver/orders")
(silver.alias("t")
  .merge(bronze_cdc_batch.alias("s"), "t.order_id = s.order_id")
  .whenMatchedUpdateAll("s.op != 'DELETE'")
  .whenMatchedDelete("s.op = 'DELETE'")
  .whenNotMatchedInsertAll()
  .execute())

# Gold: dbt or Spark aggregation into star schema fact table
```

Why this pattern works operationally: bronze gives you replay-ability (source of truth for reprocessing, satisfies "never lose raw data"), silver isolates the messy cleaning logic once, gold optimizes for the query patterns BI/ML actually run — decoupling "correct" from "fast" so you don't overfit raw ingestion to today's dashboard.

## 4. Change Data Capture (CDC) Patterns

| Pattern | Mechanism | Pros | Cons |
|---|---|---|---|
| **Log-based CDC** | Debezium reading DB WAL/binlog → Kafka | Low source impact, captures deletes, near-real-time | Requires log access/replication config, schema evolution handling |
| **Query-based CDC** | Poll `WHERE updated_at > last_watermark` | Simple, no special DB privileges | Misses hard deletes, misses updates without a reliable `updated_at`, polling lag |
| **Trigger-based CDC** | DB triggers write to a shadow change table | Captures everything incl. deletes | Adds write load/latency to source OLTP system |
| **Snapshot + diff** | Periodic full extract, diff against previous | Simple, works with any source | Expensive at scale, coarse latency |

Debezium → Kafka → Delta pattern:
```
Postgres WAL → Debezium connector → Kafka topic (orders.cdc, Avro w/ schema registry)
   → Spark Structured Streaming consumer → MERGE into silver.orders (upsert/delete by op type)
```

CDC events carry an `op` field (`c`=create, `u`=update, `d`=delete, `r`=snapshot read) — the downstream merge logic must branch on it; naively treating everything as an upsert silently loses deletes.

**Handling schema evolution in CDC**: pair with a schema registry (Confluent Schema Registry / AWS Glue Schema Registry) using backward-compatible evolution rules (new nullable columns only) so old consumers don't break on deploy.

## 5. Delivery Semantics: Exactly-Once vs At-Least-Once vs At-Most-Once

| Semantics | Guarantee | Mechanism | Cost |
|---|---|---|---|
| **At-most-once** | May lose messages, never duplicates | Fire-and-forget producer, no retry | Cheapest, rarely acceptable for analytics |
| **At-least-once** | Never loses messages, may duplicate | Producer retries + consumer commits offset *after* processing | Requires idempotent downstream writes |
| **Exactly-once** | No loss, no duplication (end-to-end) | Idempotent producer + transactional writes + offset commit atomic with output write | Highest complexity/cost |

**How exactly-once is actually achieved in practice** (it's rarely "true" exactly-once end to end — it's at-least-once delivery + idempotent processing):
- **Kafka producer**: `enable.idempotence=true` + transactional producer (`transactional.id`) prevents duplicate writes from producer retries.
- **Kafka Streams / Flink**: `processing.guarantee=exactly_once_v2` (Kafka Streams) or Flink's two-phase-commit sink + checkpoint barriers align input offset commits with output writes atomically.
- **Spark Structured Streaming**: exactly-once *at the sink* requires an idempotent sink (Delta `MERGE` keyed on a dedup key, or a sink that supports transactional writes) — Spark's checkpoint + WAL guarantees the read side is replayed consistently, but write-side exactly-once is only as good as the sink's idempotency.
- **Practical default**: build for at-least-once delivery + idempotent/dedupable writes (`MERGE ... WHEN NOT MATCHED`, unique constraint + `INSERT ... ON CONFLICT DO NOTHING`, or a dedup key + `ROW_NUMBER() OVER (PARTITION BY key ORDER BY event_time DESC) = 1`). This is cheaper to build and reason about than chasing true exactly-once everywhere.

## 6. Error Handling and Dead Letter Queues (DLQ)

```
Kafka topic ──> Consumer ──try process──> Sink
                    │
                    └─fail (schema mismatch / poison message)──> DLQ topic (orders.dlq)
                                                                       │
                                                            Alert + manual/automated replay
```

- Never let a single malformed record block a partition's consumer indefinitely — catch deserialization/validation errors per-record, route to DLQ with the original payload + error reason + timestamp, and continue.
- DLQ messages should be replayable: store enough context (original topic/partition/offset) to reprocess after a fix, and set retention/alerting so DLQ volume above a threshold pages someone rather than silently accumulating.
- In Airflow/batch context, the equivalent is a **quarantine table/partition**: rows failing validation get written to `quarantine.orders` with a `rejection_reason` column instead of failing the whole batch, so 1 bad record doesn't block 999,999 good ones.

## 7. Choosing a Table Format for the Lakehouse

| Feature | Delta Lake | Apache Iceberg | Apache Hudi |
|---|---|---|---|
| ACID transactions | Yes | Yes | Yes |
| Best ecosystem fit | Databricks/Spark | Multi-engine (Spark, Trino, Flink, Snowflake) | Spark, streaming-heavy upsert workloads |
| Upsert/merge performance | Good | Good | Optimized for frequent upserts (merge-on-read) |
| Time travel | Yes | Yes | Yes |
| Vendor neutrality | Improving, historically Databricks-centric | Strongest multi-vendor support | Strong in streaming-first shops |

Rule of thumb: multi-engine open lakehouse → Iceberg; Databricks-native stack → Delta; heavy streaming-upsert workload (CDC-heavy, many small merges) → Hudi's merge-on-read tables.
