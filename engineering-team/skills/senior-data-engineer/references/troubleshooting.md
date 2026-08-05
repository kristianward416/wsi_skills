# Troubleshooting Data Pipelines

Common failure modes in batch and streaming pipelines, how to diagnose them, and concrete fixes. Organized so you can jump straight to the symptom you're seeing.

## 1. Backpressure / Consumer Falling Behind

**Symptom**: Kafka consumer lag growing continuously; Spark Structured Streaming batch duration exceeds trigger interval; Flink job shows increasing `backPressuredTimeMsPerSecond`.

**Diagnosis:**
```bash
# Kafka consumer lag
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group orders-processor
# Look at LAG column trending up across polls, not just a point-in-time spike
```
- Flink: check the Flink UI's backpressure tab — red/high backpressure on an upstream operator means a *downstream* operator (often a slow sink or a skewed keyBy) is the bottleneck, not the operator showing red.
- Spark: check Structured Streaming UI's "Input Rate" vs "Process Rate" — sustained Input > Process means backlog is accumulating.

**Root causes and fixes:**
| Cause | Fix |
|---|---|
| Sink can't keep up (slow DB writes, small batch sizes) | Batch writes (JDBC `batchsize`, bulk insert APIs), increase sink parallelism, switch to a bulk-load-friendly sink |
| Insufficient partition/task parallelism | Increase Kafka topic partitions (note: can't decrease later) + consumer/task parallelism to match |
| Expensive per-record processing (e.g., synchronous external API call per event) | Batch external calls, cache lookups, move heavy enrichment to async/side-input pattern |
| Traffic spike beyond provisioned capacity | Autoscale consumers/executors; add `maxOffsetsPerTrigger` / rate limiting to prevent a huge catch-up batch from causing OOM on restart |
| Skewed partition key (see §4) | Fix key distribution before scaling naively |

## 2. Schema Drift

**Symptom**: Pipeline fails with `cannot resolve column`, silently drops a new column, or (worse) miscasts a retyped column producing wrong numbers with no error.

**Diagnosis:**
- Check schema registry compatibility mode violations (Confluent/Glue Schema Registry logs a rejection if producer pushed an incompatible schema).
- For batch: diff the Parquet/Delta schema of today's landing file vs. yesterday's:
  ```python
  today = spark.read.parquet("s3://lake/bronze/orders/dt=2026-08-05").schema
  yesterday = spark.read.parquet("s3://lake/bronze/orders/dt=2026-08-04").schema
  print(set(today.fieldNames()) - set(yesterday.fieldNames()))  # added columns
  print(set(yesterday.fieldNames()) - set(today.fieldNames()))  # dropped columns
  ```
- dbt: `dbt source freshness` won't catch this — use `contract: {enforced: true}` (see dataops_best_practices.md) so a retyped source column fails the build loudly instead of silently casting.

**Fixes:**
- Enforce **backward-compatible-only** evolution at the schema registry (new fields must be optional/nullable with defaults; no type changes, no required-field additions) so producers can't ship a breaking change without a version bump.
- In Spark, prefer explicit schemas (`spark.read.schema(known_schema)`) over `inferSchema=True`/schema-on-read for critical bronze ingestion — inferred schemas silently change when upstream data shape changes.
- Use `mergeSchema` (Delta) deliberately, not by default — auto-merging schema on every write hides the fact that a schema changed at all; instead, detect the change, alert, then explicitly evolve.
- Add a dead-letter/quarantine path (see data_pipeline_architecture.md §6) for records that fail schema validation so drift degrades gracefully instead of failing the whole batch.

## 3. Late-Arriving Data

**Symptom**: Streaming aggregates finalize a window before all events for that window have arrived, producing under-counted metrics that later "shouldn't" change but don't get corrected; or a batch pipeline computes "yesterday's totals" before an upstream system's delayed nightly load lands.

**Diagnosis:**
- Compare `event_time` distribution vs. `processing_time` for recent data: `SELECT event_time, processing_time, processing_time - event_time AS lateness FROM ... ORDER BY lateness DESC LIMIT 100` — find your actual p99 lateness, don't guess.
- Check watermark configuration against that measured lateness — a watermark of "5 minutes" when p99 lateness is 20 minutes guarantees systematic undercounting.

**Fixes:**
- **Streaming**: set `withWatermark(eventTimeCol, "<measured p99 lateness>")` generously, understanding the cost is larger state retention. Use `outputMode("update")` (not `"append"`) if you need a window's aggregate to keep revising as late data trickles in past the watermark's soft boundary, and reconcile with a downstream batch correction job for data arriving *after* the watermark is dropped.
- **Batch**: don't process "yesterday" the moment the clock rolls over — add an explicit **cutoff/grace period** (e.g., wait until 3am, or add an Airflow sensor gating on an upstream `_SUCCESS` marker/dataset) rather than a pure time-based schedule.
- **Lambda-architecture correction pattern**: let the speed layer produce an approximate real-time number, and have the batch layer's next full run authoritatively overwrite it once all late data has settled — make sure the serving layer swap logic (§ Lambda in data_pipeline_architecture.md) is actually happening, not just assumed.
- Always version/timestamp aggregate outputs (`computed_at`) so consumers can tell "final" numbers from "still settling" ones.

## 4. Data Skew

**Symptom**: Spark job has 199 tasks finish in 30 seconds and 1 task running for 40 minutes; Flink job has one subtask with much higher backpressure/state size than its peers; a `groupBy`/`join` stage is the pipeline's bottleneck stage.

**Diagnosis:**
```python
# Check key distribution before committing to a join/groupBy strategy
df.groupBy("customer_id").count().orderBy(col("count").desc()).show(20)
```
- Spark UI: look at the stage's task duration distribution — a long tail (max >> median) on a shuffle stage is the signature of skew.
- Common real-world skew sources: a "null"/"unknown" key bucket that's actually a huge fraction of rows, a small number of power-user/whale accounts, a default/placeholder foreign key.

**Fixes:**
| Technique | How |
|---|---|
| **Salting** | Add a random suffix to the skewed key for the shuffle, join, then strip the suffix: `df.withColumn("salted_key", concat(col("key"), lit("_"), (rand()*10).cast("int")))` — spreads one hot key across N reducer tasks |
| **Broadcast join** | If one side of a skewed join is small enough (< ~10GB depending on cluster), broadcast it (`spark.sql.autoBroadcastJoinThreshold` or explicit `broadcast(df)`) to avoid shuffling the large skewed side entirely |
| **Isolate and handle the hot key separately** | Filter out the top N hot keys, process them with a different strategy (e.g., pre-aggregate), union back with the long tail processed normally |
| **AQE skew join optimization** | Enable Spark Adaptive Query Execution: `spark.sql.adaptive.enabled=true` + `spark.sql.adaptive.skewJoin.enabled=true` — Spark 3.x automatically splits skewed partitions at runtime |
| **Repartition before an expensive stage** | `df.repartition(200, "key")` only helps if the *key itself* isn't skewed; if the key is skewed, repartitioning by that key doesn't fix it — salting or broadcast is required instead |

## 5. Small Files Problem

**Symptom**: Slow reads on a table despite modest total data size; `OPTIMIZE`/compaction jobs take a long time; listing a partition directory is itself slow.

**Diagnosis:**
```sql
-- Delta: check file count vs data size
DESCRIBE DETAIL gold.fact_orders;  -- numFiles vs sizeInBytes
```
If `numFiles` is in the tens of thousands for a table that's only a few GB, you have a small-files problem — usually caused by frequent low-volume streaming micro-batch writes or over-partitioning (see data_modeling_patterns.md §6).

**Fixes:**
- Delta: `OPTIMIZE gold.fact_orders` (bin-packing compaction), scheduled periodically, plus `ZORDER BY` on the common filter column.
- Reduce write frequency/increase micro-batch size (`trigger(processingTime="5 minutes")` instead of `"10 seconds"`) if near-real-time isn't actually required at that granularity.
- Repartition before write: `df.repartition(numPartitions).write...` to control output file count directly, or use `df.coalesce(n)` when reducing partition count without a full shuffle.

## 6. Airflow-Specific Failure Modes

| Symptom | Likely cause | Fix |
|---|---|---|
| DAG doesn't show up in UI | Import error, top-level exception in DAG file | `airflow dags list-import-errors`; move heavy imports/API calls out of top-level DAG file code (runs on every scheduler parse cycle) |
| Task stuck in `queued` forever | No available worker slots, pool exhausted, or scheduler heartbeat issue | Check `airflow.cfg` pool sizes, executor concurrency, and scheduler logs |
| Task succeeds but downstream data is wrong | Non-idempotent task re-ran (retry or backfill) and double-appended instead of overwriting/merging | Audit task for `INSERT` vs `MERGE`/`INSERT OVERWRITE`; see workflows.md idempotency section |
| SLA misses with no clear cause | Upstream sensor polling inefficiently, or resource contention (too many concurrent DAG runs sharing a pool) | Convert polling sensors to deferrable operators (`deferrable=True`) to free worker slots while waiting; use Datasets for event-driven triggering instead of time-based polling |

## 7. Data Quality Failures Caught Too Late

**Symptom**: Bad data reaches a BI dashboard or ML model before anyone notices — the pipeline "succeeded" but the output is wrong.

**Root cause**: quality gates exist only at the end (or not at all) rather than between each layer.

**Fix**: apply the layered testing strategy from dataops_best_practices.md — Great Expectations at ingestion, dbt tests between silver and gold — and make test failures **block promotion** (fail the DAG task, don't just log a warning) so bad data physically cannot reach `gold`/BI schemas. Pair with row-count/volume anomaly monitoring (§ Observability in dataops_best_practices.md) as a second, independent line of defense for quality issues that pass explicit tests but are still anomalous (e.g., valid-looking data that's simply the wrong day's data reloaded).
