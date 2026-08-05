# DataOps Best Practices

Applying DevOps discipline — testing, CI/CD, observability, contracts — to data pipelines, so pipeline changes are as safe and reviewable as application code changes.

## 1. Data Testing Frameworks

### dbt tests
Two categories: **generic** (reusable, declared in YAML) and **singular** (custom SQL, one-off).

```yaml
# models/marts/schema.yml
models:
  - name: fact_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_key
        tests:
          - not_null
          - relationships:
              to: ref('dim_customer')
              field: customer_key
      - name: net_amount
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
```

Singular test (business-rule assertion):
```sql
-- tests/assert_no_negative_inventory.sql
select *
from {{ ref('fact_inventory') }}
where quantity_on_hand < 0
```
A dbt test "passes" when the query returns **zero rows** — this is the pattern to remember when writing custom tests.

`dbt build` runs models and tests in dependency order, failing fast on a broken upstream model rather than continuing to build downstream on bad data.

### Great Expectations
Better suited to validating raw/ingested data before it enters the warehouse, or validating pandas/Spark DataFrames mid-pipeline.

```python
import great_expectations as gx

context = gx.get_context()
validator = context.sources.pandas_default.read_csv("data/sales.csv")

validator.expect_column_values_to_not_be_null("order_id")
validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_be_between("net_amount", min_value=0, max_value=1_000_000)
validator.expect_column_values_to_match_regex("email", r"^[^@]+@[^@]+\.[^@]+$")

result = validator.validate()
if not result.success:
    raise ValueError("Data quality checks failed — see validation result for details")
```

Integrate as an Airflow task gating downstream publish:
```python
@task
def validate_sales_data():
    result = validator.validate()
    if not result.success:
        raise AirflowFailException("GE validation failed")
```

**Where to test what:**
| Layer | Test type | Tooling |
|---|---|---|
| Ingestion (bronze) | Schema conformance, null/type checks on raw | Great Expectations |
| Transform (silver) | Uniqueness, referential integrity, business rules | dbt tests |
| Marts (gold) | Metric reconciliation, row-count sanity vs. source | dbt tests + custom reconciliation job |

## 2. Data Contracts

A **data contract** is an explicit, versioned agreement between a data producer (source team/service) and consumers, specifying schema, semantics, SLAs, and change-management rules — turns "silent breaking change" into "contract violation caught before merge."

```yaml
# contracts/orders_v2.yaml
name: orders
version: 2.0.0
owner: checkout-team
schema:
  - name: order_id
    type: string
    required: true
  - name: net_amount
    type: decimal(12,2)
    required: true
  - name: currency
    type: string
    required: true
    added_in: 2.0.0
sla:
  freshness_minutes: 15
  completeness_pct: 99.5
breaking_change_policy: major_version_bump_and_dual_publish_90_days
```

Enforcement points:
- **Producer side**: CI check that a proposed schema change against the service/event doesn't violate the contract (e.g., `openapi`/Avro schema validated against schema registry compatibility rules — `BACKWARD`, `FORWARD`, or `FULL` compatibility mode in Confluent Schema Registry).
- **Consumer side**: pipeline validates incoming data against the contract schema at ingestion (reject/quarantine on violation rather than silently propagating).
- Dbt-native approximation: `contract: {enforced: true}` on a model forces the model's declared column types to match its `.sql`, failing the build if they drift:
  ```yaml
  models:
    - name: fact_orders
      config:
        contract:
          enforced: true
      columns:
        - name: order_id
          data_type: string
        - name: net_amount
          data_type: decimal(12,2)
  ```

## 3. CI/CD for Data Pipelines

**What belongs in CI for a dbt/Airflow project:**
```yaml
# .github/workflows/ci.yml
name: dbt-ci
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install dbt-snowflake
      - run: dbt deps
      - run: dbt compile --target ci        # catch Jinja/SQL syntax errors
      - run: dbt build --target ci --select state:modified+ --defer --state ./prod-manifest
        # slim CI: only build models changed vs. prod, deferring unchanged upstream to prod
      - run: sqlfluff lint models/ --dialect snowflake
```

Key CI/CD practices specific to data (beyond generic app CI):
- **Slim CI (`state:modified+`)**: never rebuild the entire warehouse per PR — build only changed models and their downstream dependents, deferring unchanged models to production state. This is the dbt-specific equivalent of not running the whole test suite on every app change.
- **Ephemeral/isolated schema per PR** (`dbt build --target ci` pointed at a PR-scoped schema) so parallel PRs don't clobber each other's test data.
- **Airflow DAG validation in CI**: `python -c "import dagbag; dagbag.get_dag('my_dag')"` or `airflow dags list-import-errors` to catch broken DAGs before merge, plus DAG-linting (no top-level heavy computation, no missing `catchup` settings).
- **Automated data diff on PR** (e.g., `dbt-checkpoint`, or tools like `data-diff`) comparing row counts/checksums of modified models between PR branch and prod before merging a transformation-logic change.
- **Progressive deployment**: deploy transform changes to a staging schema, run reconciliation against prod for N days on a shadow pipeline before cutting BI over — especially for changes to core fact tables.

## 4. Observability and Lineage

**Three pillars of pipeline observability**: freshness, volume, schema — plus quality (test pass rate).

| Signal | What it catches | How to monitor |
|---|---|---|
| Freshness | Upstream source went stale/stopped updating | `dbt source freshness`, Airflow SLA misses, Monte Carlo/Metaplane freshness monitors |
| Volume | Silent partial load, duplicate load | Row-count anomaly detection (e.g., alert if today's row count deviates >3σ from 28-day rolling average) |
| Schema | Upstream added/dropped/retyped a column | Schema registry compatibility checks, dbt contracts, Monte Carlo schema change detection |
| Quality | Business-rule violations | dbt test pass rate trend, Great Expectations validation results dashboard |

```yaml
# dbt source freshness config
sources:
  - name: erp
    tables:
      - name: orders
        loaded_at_field: updated_at
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 24, period: hour}
```

**Lineage**: use `dbt docs generate` (column-level lineage within dbt) plus **OpenLineage** (via Airflow's `openlineage-airflow` provider or Marquez/Datahub integration) to capture lineage *across* tools — Airflow task → Spark job → dbt model — not just within one tool. Column-level lineage is what lets you answer "if I change this source column, what dashboards break" before you change it, not after an incident.

**Monitoring stack mapping** (from the tech stack): Prometheus/Grafana for pipeline infra metrics (task duration, queue depth, executor slots), Datadog/PagerDuty for alert routing, Monte Carlo/Metaplane for data-specific anomaly detection (freshness/volume/schema/quality) layered on top of infra monitoring — infra health and data health are different signals and need different tools.

## 5. Incident Response for Data Pipelines

1. **Detect**: automated alert (freshness/volume/schema/test failure) fires before a human notices a wrong dashboard number.
2. **Triage**: identify blast radius via lineage graph — which downstream marts/dashboards/models consume the broken upstream table.
3. **Contain**: pause downstream consumers (or mark the table "stale" in the catalog) rather than let bad data propagate into gold/BI while investigating.
4. **Root cause**: check upstream schema changes, source system incidents, recent deploys of the transform code, and whether a backfill/reprocess is needed.
5. **Fix + backfill**: apply the code fix, then reprocess the affected partitions specifically (not a blanket full-history rerun) using the idempotent backfill pattern from workflows.md.
6. **Postmortem**: write up root cause, add a regression test (dbt test or GE expectation) that would have caught it, and consider whether a data contract would have prevented the upstream change from shipping at all.

Maintain a **data incident runbook** analogous to an application on-call runbook: who owns which pipeline, escalation path, and a status page/Slack channel convention (`#data-incidents`) so consumers know when to distrust a dashboard.
