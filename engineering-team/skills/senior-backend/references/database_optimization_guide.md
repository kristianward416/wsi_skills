# Database Optimization Guide

Reference for the Database Optimization Workflow in `senior-backend`, used alongside `scripts/database_migration_tool.py --analyze` and `--suggest-indexes`. Examples are Postgres-flavored (the default backing store for all four profiles in SKILL.md — `node-express`, `fastapi-python`, `django-monolith`, and the extracted-service profile before it moves to a purpose-built store).

---

## 1. Indexing Strategy

### Index types and when to use them

| Index type | Postgres syntax | Use when |
|---|---|---|
| B-tree (default) | `CREATE INDEX idx ON t(col)` | Equality and range queries (`=`, `<`, `>`, `BETWEEN`), sorting |
| Composite | `CREATE INDEX idx ON t(col_a, col_b)` | Queries filtering/sorting on multiple columns together — column order must match query order (leftmost-prefix rule) |
| Partial | `CREATE INDEX idx ON t(col) WHERE status='active'` | Queries that always filter a low-cardinality flag; keeps index small and fast |
| Covering (`INCLUDE`) | `CREATE INDEX idx ON t(a) INCLUDE (b)` | Turns an index scan into an index-only scan by embedding extra columns the query selects but doesn't filter on |
| GIN | `CREATE INDEX idx ON t USING gin(tags)` | Array containment, full-text search (`tsvector`), JSONB key lookups |
| BRIN | `CREATE INDEX idx ON t USING brin(created_at)` | Very large, naturally-ordered tables (append-only time series) — tiny index size, coarse granularity |
| Hash | `CREATE INDEX idx ON t USING hash(col)` | Pure equality only, rarely worth it over B-tree in Postgres ≥ 10 |

### The leftmost-prefix rule
A composite index on `(user_id, status, created_at)` serves:
- `WHERE user_id = ?` ✓
- `WHERE user_id = ? AND status = ?` ✓
- `WHERE status = ?` ✗ (can't skip the leftmost column)
- `WHERE user_id = ? AND created_at > ?` — uses the index for `user_id` only, then filters `created_at` in-memory; still helpful but not as tight as if `created_at` were second.

Column order rule of thumb: equality columns first, then the single range/sort column last. This matches the exact examples in SKILL.md's Database Index Strategy section (`idx_orders_user_status ON orders(user_id, status)`).

### Cost of over-indexing
Every index slows every `INSERT`/`UPDATE`/`DELETE` on that table (each write must update every index) and consumes memory competing with your buffer cache. Rule of thumb: audit indexes quarterly with
```sql
SELECT indexrelid::regclass AS index, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelid NOT IN (SELECT conindid FROM pg_constraint);
```
Unused, non-constraint indexes with `idx_scan = 0` after a full traffic cycle (at least one full week, ideally a month) are candidates for removal.

---

## 2. N+1 Query Detection and Resolution

The N+1 pattern: one query to fetch a list of N parents, then N additional queries to fetch each parent's related data. It's the single most common backend performance bug, and it's the primary reason GraphQL resolvers need extra care (see `api_design_patterns.md` §1).

**Detecting it:**
- Symptom: request latency scales linearly with result-set size (10 orders = 50ms, 100 orders = 400ms) even though the "logical" query is one list fetch.
- Enable query logging with counts per request (`pg_stat_statements`, or an ORM query counter middleware) and alert when a single HTTP request issues more than, say, 20 queries — that's almost always N+1, not legitimate complexity.
- In Postgres, `log_min_duration_statement = 0` temporarily in a staging environment plus grouping by request ID surfaces the pattern immediately.

**Fixing it — three patterns, pick by context:**

1. **Eager loading / JOIN.** Fetch parent and children in one query.
```sql
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending';
```
ORM equivalent (Sequelize): `Order.findAll({ where: { status: 'pending' }, include: [User] })`. Best when the fan-out ratio is small and bounded (each order has exactly one user).

2. **Batch loading (DataLoader pattern).** For GraphQL resolvers or any case where eager-loading isn't feasible because the "child fetch" decision is made per-field, batch all keys requested within a single event-loop tick into one `WHERE id IN (...)` query, then fan the results back out:
```typescript
const userLoader = new DataLoader(async (userIds: string[]) => {
  const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [userIds]);
  const byId = new Map(users.map(u => [u.id, u]));
  return userIds.map(id => byId.get(id));  // must preserve input order
});

// Each resolver call for a different order in the same tick:
await userLoader.load(order.userId);  // batched into one query, not N queries
```

3. **Denormalize / cache the derived value.** When the "N" side is read far more often than the parent changes (e.g., `order_count` on a `users` row), maintain it with a trigger or application-level counter instead of computing it via join/subquery on every read.

**Decision rule:** if the child collection size is bounded and always needed → JOIN/eager-load. If it's needed conditionally per-field in a graph → DataLoader batching. If it's read orders of magnitude more than it's written → denormalize.

---

## 3. Query Plan Analysis

`EXPLAIN ANALYZE` is the primary diagnostic tool — this is the exact command referenced in SKILL.md's Database Optimization Workflow Step 2:
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 10;
```

**What to look for, worst to best:**

| Plan node | Meaning | Action |
|---|---|---|
| `Seq Scan` | Full table scan | Add an index on the filter column, unless the table is tiny (< ~1000 rows) or the query returns most of the table anyway |
| `Index Scan` | Uses index, then fetches matching rows from the heap | Good; if this is still slow, consider a covering index (`INCLUDE`) to make it index-only |
| `Index Only Scan` | Satisfies the query entirely from the index, no heap fetch | Ideal — this is the target state for hot-path queries |
| `Bitmap Heap Scan` | Combines multiple index matches before hitting the heap | Fine for medium-selectivity queries; if it dominates cost, the selectivity estimate may be off — check `ANALYZE` stats are current |
| `Nested Loop` with a high row count on the inner side | Executes inner query once per outer row | The query-level version of N+1 — fine for small inputs, disastrous if the planner misjudges cardinality; check `rows` estimated vs `actual` |
| `Sort` with `external merge` in `BUFFERS` output | Sort spilled to disk | Increase `work_mem` for the session, or add an index that avoids the sort entirely (index on the `ORDER BY` column matching direction) |

**Reading cost numbers:** `cost=0.42..8.44` — first number is startup cost (before the first row is returned), second is total cost to return all rows, in arbitrary planner cost units (not ms). Compare `actual time` between plan nodes, not the cost estimates, when analyzing a real bottleneck — cost is only used by the planner to choose a plan.

**Stale statistics are a common false alarm:** if the planner picks a seq scan when an index clearly exists and should be selective, run `ANALYZE tablename` first — Postgres may be working off outdated row-count/distribution estimates (`pg_stat_user_tables.last_analyze`).

---

## 4. Connection Pooling Math

Every backend connects to Postgres through a pool (PgBouncer, or the app-level pool in `pg`/Sequelize/SQLAlchemy). Getting the pool size wrong is a top-3 cause of "random" latency spikes under load.

**The formula (from PostgreSQL's own connection-pooling guidance, generalized from Little's Law):**
```
connections_needed ≈ ((core_count * 2) + effective_spindle_count)
```
For SSD-backed cloud databases (spindle count ≈ 0 as a practical matter), this simplifies to roughly `2× vCPU count` **per database instance**, not per application instance. A db.r6g.xlarge (4 vCPU) postgres instance wants roughly 8-16 *active* connections doing real work at once — not hundreds.

**Why "just increase max_connections" backfires:** each Postgres connection is a full OS process (~5-10MB RAM plus its own query working memory) and the planner/executor contend for the same finite CPU and buffer cache. Beyond the sweet spot, throughput *drops* as connection count rises, because of context-switching and lock-contention overhead — this is the classic pgbench "throughput peaks then falls" curve.

**Correct pattern:** many app instances → PgBouncer (transaction-pooling mode) → a small number of real Postgres connections.
```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction     # reuse connection across transactions, not per-session
max_client_conn = 1000       # app-side connections PgBouncer accepts
default_pool_size = 20       # actual connections PgBouncer opens to Postgres, per database/user pair
```
- `pool_mode = transaction` is required to get the multiplexing benefit — `session` mode ties one client to one server connection for the whole session, defeating the point.
- Set each app instance's own pool (`pg.Pool({max: N})`) small too — e.g., `max: 10` per instance × 10 instances = 100 client connections into PgBouncer, which then multiplexes down to `default_pool_size: 20` real Postgres connections.
- Watch `pg_stat_activity` count and `wait_event_type = 'Lock'` under load; if backends are queuing on PgBouncer (visible via `SHOW POOLS` — `cl_waiting > 0` sustained), that's a genuine capacity signal, not just a config nudge — it feeds directly into the read/write ratio + QPS forecast in Forcing Question 1.

---

## 5. Sharding and Partitioning Tradeoffs

These are different tools solving different scaling axes — don't reach for sharding when partitioning solves the actual problem.

### Partitioning (single instance, split by range/list/hash)
```sql
CREATE TABLE orders (
  id bigint, created_at timestamptz, user_id bigint, status text
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_01 PARTITION OF orders
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```
- Solves: table bloat, slow vacuum/index maintenance, and lets you drop old partitions instantly (`DROP TABLE orders_2020_01` instead of a slow `DELETE`).
- Doesn't solve: single-instance CPU/IO ceiling — all partitions still live on one Postgres instance.
- Use when a single table has grown past roughly 50-100M rows and index maintenance/vacuum is visibly degrading, but you're not yet write-throughput-bound on one instance.

### Sharding (horizontal split across instances)
- Solves: write throughput and total dataset size exceeding what one instance's CPU/IO/RAM can serve — this is the ≥1000 QPS territory of the `go-or-rust-microservice` profile in SKILL.md, not the < 50 QPS territory of the monolith profiles.
- Shard key choice is the whole game: pick a key that (a) matches your dominant query pattern (usually `tenant_id` or `user_id` for multi-tenant systems) so most queries stay single-shard, and (b) distributes writes evenly (avoid a shard key correlated with time, like `created_at`, which creates a hot "current" shard).
- Cross-shard joins and transactions become application-level problems — you either denormalize to avoid them, or accept eventual consistency and a saga/outbook pattern for cross-shard writes.
- **Don't shard prematurely.** Sharding is a one-way architectural commitment that multiplies operational complexity (migrations, backups, and rebalancing all become per-shard problems). It should be a direct, evidenced answer to Forcing Question 1 (read/write ratio + p99 QPS), not a defensive default. Most systems on the `node-express`/`fastapi-python`/`django-monolith` profiles never need it — a well-indexed single Postgres instance with read replicas comfortably serves tens of thousands of QPS for read-heavy workloads.

### Read replicas (the usual first move, before either of the above)
- Solves: read-heavy load (the common case — most systems are read-dominant per Forcing Question 1's ratio).
- Route reporting/analytics and GET-heavy traffic to replicas; keep writes and read-your-own-write flows on the primary.
- Replication lag is the tradeoff — for any flow where a user must immediately see their own write (e.g., "create order" → "show order confirmation"), read from the primary or pass a "read your own write" hint, don't blindly round-robin.

**Escalation order:** add indexes → tune queries/N+1 → add a read replica → partition large tables → shard. Each step is cheaper and more reversible than the next; skipping straight to sharding without exhausting the earlier steps is the most common over-engineering mistake this skill's decision engine is designed to catch (see the `runner-up tradeoff` output of `backend_decision_engine.py` in SKILL.md).
