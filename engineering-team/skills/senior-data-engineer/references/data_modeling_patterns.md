# Data Modeling Patterns

How to structure data for analytics, from OLTP-adjacent normalized models to analytics-optimized dimensional models, plus the Data Vault alternative for enterprise integration layers.

## 1. Normalization vs Denormalization Tradeoff

| | Normalized (3NF) | Denormalized (dimensional) |
|---|---|---|
| Write/update cost | Low (single place to update) | Higher (redundant copies) |
| Read/query cost (analytics) | High (many joins) | Low (few joins, wide tables) |
| Storage | Compact | Larger (acceptable — storage is cheap in columnar warehouses) |
| Best for | OLTP source systems | OLAP/BI consumption layer (gold layer) |

Rule: model **bronze/silver** close to source (normalized-ish, conformed), model **gold** for query patterns (denormalized, dimensional) — don't force one modeling philosophy across every layer.

## 2. Dimensional Modeling — Star Schema

A **fact table** (numeric measures, foreign keys to dimensions, grain explicitly defined) surrounded by **dimension tables** (descriptive attributes).

```
                dim_date
                    │
dim_customer ── fact_orders ── dim_product
                    │
                dim_store
```

```sql
CREATE TABLE fact_orders (
    order_id            BIGINT,
    order_date_key      INT REFERENCES dim_date(date_key),
    customer_key        INT REFERENCES dim_customer(customer_key),
    product_key         INT REFERENCES dim_product(product_key),
    store_key           INT REFERENCES dim_store(store_key),
    quantity            INT,
    net_amount          DECIMAL(12,2),
    discount_amount     DECIMAL(12,2)
    -- grain: one row per order line item
);
```

**Grain declaration is the single most important modeling decision** — write it explicitly ("one row per order line item, per day") before building the table; grain ambiguity is the #1 cause of double-counted metrics downstream.

**Additive vs semi-additive vs non-additive facts**: `net_amount` sums correctly across any dimension (additive). An `account_balance` snapshot fact only sums meaningfully across some dimensions, not time (semi-additive — use last-value-of-period, not SUM, when aggregating over date). Ratios/percentages are non-additive — never store them as facts; store numerator/denominator and compute the ratio at query time.

## 3. Snowflake Schema

Normalizes dimension tables further (e.g., `dim_product` → `dim_category` → `dim_department`) to eliminate redundancy in the dimension itself.

```
dim_department ── dim_category ── dim_product ── fact_orders
```

Tradeoff: saves storage/update-anomaly risk in slowly-changing hierarchies, but costs extra joins per query. **In modern columnar warehouses (Snowflake, BigQuery), prefer star over snowflake** — join cost isn't the bottleneck it was in the 1990s and analysts/BI tools navigate flat star schemas more easily. Snowflake schemas are still justified when a dimension is huge and highly reused with independent update cadence (e.g., a massive product catalog dimension shared across many facts).

## 4. Slowly Changing Dimensions (SCD)

| Type | Behavior | When to use |
|---|---|---|
| **Type 0** | Never update — original value retained forever | Immutable attributes (e.g., original signup date) |
| **Type 1** | Overwrite in place, no history | Correcting errors, attributes where history doesn't matter (e.g., typo fix in name) |
| **Type 2** | New row per change, with `effective_date`/`end_date`/`is_current` flag | Full history required — the classic dimensional-modeling default |
| **Type 3** | Add a new column for "previous value" | Need exactly one prior value, not full history (rare) |
| **Type 4** | Current value in dimension + full history in a separate mini-dimension/history table | High-frequency-changing attributes kept out of the main dimension for performance |
| **Type 6** | Hybrid: Type1 + Type2 + Type3 in one row (current value overwritten, history rows kept, "previous value" column also maintained) | Need both fast "current value" filtering and full history in the same table |

**Type 2 implementation (dbt snapshot):**
```yaml
# snapshots/customers_snapshot.yml
snapshots:
  - name: customers_snapshot
    relation: source('crm', 'customers')
    config:
      unique_key: customer_id
      strategy: timestamp
      updated_at: updated_at
```
This generates `dbt_valid_from`, `dbt_valid_to` columns automatically — query `WHERE dbt_valid_to IS NULL` for current rows, or join a fact's `order_date` between `dbt_valid_from` and `dbt_valid_to` for point-in-time-correct history.

**Type 2 implementation (raw SQL MERGE pattern):**
```sql
MERGE INTO dim_customer AS t
USING staged_customers AS s
ON t.customer_id = s.customer_id AND t.is_current = TRUE
WHEN MATCHED AND (t.email != s.email OR t.address != s.address) THEN
  UPDATE SET t.is_current = FALSE, t.end_date = CURRENT_DATE
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, address, effective_date, end_date, is_current)
  VALUES (s.customer_id, s.email, s.address, CURRENT_DATE, NULL, TRUE);
-- followed by a separate INSERT for the new current row when matched-and-changed
```

## 5. Data Vault Modeling

Designed for enterprise integration layers where auditability, traceability to source, and resilience to source-system change matter more than query convenience (BI sits on top, in a dimensional layer built from the vault).

Three core entity types:
- **Hub**: unique list of a business key + surrogate hash key + load metadata. No descriptive attributes.
  ```sql
  CREATE TABLE hub_customer (
      customer_hk   BINARY(32) PRIMARY KEY,  -- hash of business key
      customer_bk   VARCHAR,                  -- natural business key
      load_date     TIMESTAMP,
      record_source VARCHAR
  );
  ```
- **Link**: represents a relationship/transaction between hubs (many-to-many by design).
  ```sql
  CREATE TABLE link_customer_order (
      link_hk        BINARY(32) PRIMARY KEY,
      customer_hk    BINARY(32) REFERENCES hub_customer,
      order_hk       BINARY(32) REFERENCES hub_order,
      load_date      TIMESTAMP,
      record_source  VARCHAR
  );
  ```
- **Satellite**: descriptive attributes + history, attached to a hub or link, keyed by hash key + load timestamp (naturally gives you Type-2-like history for free).
  ```sql
  CREATE TABLE sat_customer_details (
      customer_hk   BINARY(32),
      load_date     TIMESTAMP,
      email         VARCHAR,
      address       VARCHAR,
      hash_diff     BINARY(32),  -- detects change, avoids no-op inserts
      PRIMARY KEY (customer_hk, load_date)
  );
  ```

**When to choose Data Vault over straight dimensional modeling**: multiple disparate source systems feeding one integration layer, frequent source-system schema churn, strict audit/traceability requirements (regulated industries), need to load hubs/links/satellites in parallel for scale. **Cost**: significantly more tables and joins; almost always requires a downstream dimensional ("information mart") layer built via views/dbt on top of the vault for BI consumption — Data Vault is not meant to be queried directly by analysts.

## 6. Partitioning and Clustering (Physical Modeling)

- **Partition by the column most queries filter on and that has bounded cardinality growth** — almost always a date (`order_date`), not a high-cardinality key like `customer_id` (too many small partitions = metadata overhead).
- **Clustering/sort keys** (Snowflake `CLUSTER BY`, BigQuery `CLUSTER BY`, Delta `ZORDER BY`) for the next-most-common filter/join column after the partition key:
  ```sql
  -- Snowflake
  ALTER TABLE fact_orders CLUSTER BY (order_date, customer_key);

  -- Delta Lake
  OPTIMIZE gold.fact_orders ZORDER BY (customer_key);
  ```
- **Avoid over-partitioning**: a common failure mode is partitioning by both date *and* a high-cardinality column, producing thousands of tiny files/partitions that slow down both writes (many small files) and reads (metadata scan overhead) — see troubleshooting.md for the small-files failure mode.

## 7. dbt Modeling Best Practices

- **Layered model structure**: `staging` (1:1 with source, light renaming/casting, no joins) → `intermediate` (joins/business logic building blocks) → `marts` (final star-schema fact/dim tables consumed by BI).
- **One staging model per source table**, materialized as a `view` (cheap, always fresh); **marts materialized as `table` or `incremental`** for query performance.
- **Incremental models keyed correctly**:
  ```sql
  {{ config(materialized='incremental', unique_key='order_id', incremental_strategy='merge') }}
  select * from {{ source('erp', 'orders') }}
  {% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
  {% endif %}
  ```
- **Surrogate keys** via `dbt_utils.generate_surrogate_key(['col1','col2'])` rather than trusting source systems to have stable natural keys.
- **Exposures and sources.yml** to make lineage from raw source to BI dashboard explicit and testable.
