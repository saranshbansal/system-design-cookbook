# Database Indexing

Indexes make reads fast by allowing the database to locate rows without scanning the entire table. They are fundamental to query performance, but they come with storage and maintenance costs — every index slows writes (INSERT/UPDATE/DELETE) and adds complexity to schema management. In interviews, you should be ready to explain index types, trade-offs, how the optimizer uses indexes, and how you'd design indexes for specific queries and workloads.

---

## Key index types 🔧

- **B-tree (balanced tree)** — The default for relational databases. Supports exact lookups, ordered scans, and range queries (e.g., BETWEEN, >, <) and can speed up ORDER BY when the index ordering matches the query.
- **Hash** — Very fast for equality lookups (WHERE col = X) but doesn't support range queries or ordering. Good for specific workloads, but less flexible than B-trees.
- **Bitmap** — Efficient for low-cardinality columns (few distinct values) and often used in OLAP engines for complex ad-hoc queries.
- **Inverted / Full-text** — For text search (words→documents). Implemented by Elasticsearch, Lucene, or DB extensions (e.g., Postgres `GIN`/`tsvector`).
- **Geospatial (R-tree / GiST / SP-GiST)** — Spatial queries (distance, bounding boxes, nearest neighbors) use specialized spatial indexes like PostGIS.
- **Function / Expression / Partial / Covering indexes** — Indexes on an expression (e.g., `lower(email)`), a subset of rows (partial index), or with included columns (covering index) to support index-only scans.
- **Clustered vs Non-clustered** — Clustered (e.g., primary key in InnoDB) dictates physical row order; non-clustered stores pointers to rows.

---

## How indexes are used and what affects their usefulness 📊

- **Selectivity & cardinality**: High-cardinality columns (many distinct values) are good index candidates because they filter out many rows. Low-cardinality columns (e.g., boolean) rarely benefit by themselves.
- **Statistics & optimizer**: The query planner uses column statistics to decide between index scan and sequential scan. Stale statistics can lead to poor plans.
- **Index-only scans / covering indexes**: If the index contains all columns a query needs (including via "included" columns), the database can satisfy the query from the index alone—much faster because it avoids accessing table rows.
- **Left-prefix rule (multi-column indexes)**: A composite index on (A, B, C) can be used for queries filtering A, (A, B), or (A, B, C) but not for queries that filter only B and C (except when the DB supports index skip-scan or intersection).
- **Ordering and pagination**: An index can avoid sorting when the index order matches ORDER BY; an index on (user_id, created_at DESC) helps `ORDER BY created_at DESC LIMIT 10` per user.
- **Index intersection**: Some DBs can combine multiple single-column indexes; this is less efficient than a well-chosen composite index but useful in certain cases.

---

## Interview: how to reason about indexes (step-by-step) ✅

1. **Ask about workload**: read vs write ratio, latency SLO, data growth, cardinality, and whether queries are OLTP or analytical.
2. **Examine query patterns**: filters (WHERE), joins (ON), sorts (ORDER BY), groupings (GROUP BY), and projections (SELECT columns).
3. **Choose index type & columns**: prefer composite index covering WHERE → JOIN → ORDER BY left-prefix; include extra columns to make it a covering index when beneficial.
4. **Consider trade-offs**: each index uses storage and slows writes; propose partial or time-based indexes for large tables.
5. **Validate**: run `EXPLAIN`/`EXPLAIN ANALYZE`, check cardinality estimates and whether index-only scans occur.
6. **Operational concerns**: index build time, online vs offline rebuilds, fragmentation, monitoring and alerting for slow queries.

Example: query

```sql
SELECT id, total
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 10;
```

Good index: `CREATE INDEX ON orders (user_id, created_at DESC) INCLUDE (total)` — supports the filter, the sort, and is a covering index for `total`.

---

## Practical tips, pitfalls & trade-offs ⚠️

- **Don't over-index**: Too many indexes slow writes and increase storage. Index only what's necessary.
- **Beware low-cardinality columns**: Indexes on boolean or tiny-enum fields rarely help unless combined with other predicates or used in partial indexes.
- **Order matters in composite indexes**: Follow the left-prefix rule — design column order to match queries.
- **Updates create write amplification**: Every indexed column modified increases work for updates.
- **Stale stats and misconceptions**: Use `ANALYZE` / `VACUUM` (Postgres) or equivalent to keep stats fresh so the optimizer makes good choices.
- **Fragmentation & maintenance**: Reindex or rebuild when fragmentation hurts performance; consider `CONCURRENT`/`ONLINE` index builds where supported.

---

## Scaling & distributed systems 🧭

- **Sharding**: Local indexes live within a shard; global indexes are harder and may require additional infrastructure or GSI (global secondary index) support (e.g., DynamoDB, Cassandra). Consider query routing to reduce cross-shard accesses.
- **External search systems**: For rich full-text search, complex ranking, or geo-search at scale, use Elasticsearch, OpenSearch, or specialized DB extensions. Keep in mind CDC-based syncing and eventual consistency.
- **Materialized views & pre-aggregation**: For expensive analytical queries, precompute and index materialized results.

---

## Diagnostics & monitoring 🔍

- Use `EXPLAIN` / `EXPLAIN ANALYZE` to inspect query plans and time breakdown.
- Monitor slow query logs and per-index usage statistics (`pg_stat_user_indexes`, `performance_schema` in MySQL).
- Track index size, bloat, and read/write amplification.

---

## Quick checklist for interviews (copyable) ✍️

- What are the query patterns and SLAs? ✅
- Read/write ratio and tolerance for write amplification? ✅
- Which columns are used in WHERE/JOIN/ORDER BY/GROUP BY? ✅
- Cardinality & selectivity of candidates? ✅
- Can we use a composite or covering index? ✅
- Would a partial or function index reduce size/cost? ✅
- How will the index affect writes and storage? ✅
- How to validate: EXPLAIN + benchmark + monitor? ✅

---

## Useful SQL examples

- Single-column index
```sql
CREATE INDEX idx_users_email ON users (email);
```

- Composite index with include (Postgres 11+, SQL Server)
```sql
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC) INCLUDE (total);
```

- Partial index (Postgres)
```sql
CREATE INDEX idx_active_users_email ON users (email) WHERE active = true;
```

- Function index (Postgres)
```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
```

---

## When not to index

- Tiny tables that fit in memory (table scan is cheap)
- Very low-cardinality fields used alone
- Columns that change frequently and don't improve query performance

---

### TL;DR 💡
Indexing is about trade-offs: speed up reads, slow down writes, and use storage. In interviews, focus on the workload, query patterns, and clear justification for any index you recommend — show how you'd validate it with `EXPLAIN` and monitoring.

For specialized needs beyond your primary DB, use external systems like Elasticsearch for full-text search or PostGIS for advanced geospatial queries — accept eventual consistency when syncing via CDC.

---

## Senior Q&A — interview-style (principled, short answers) 🎯

1. **Q: How do you design indexes for a table with heavy writes but occasional reads (hot write table)?**
   A: Minimize indexes — keep only critical ones used by the hottest reads. Use partial indexes for the hot subset, partitioning by time/tenant, and consider read-optimized replicas or an asynchronous denormalized read table to offload queries.

2. **Q: In a sharded system, how do you provide global queryability (e.g., find user by email) efficiently?**
   A: Prefer routing queries to the shard using a deterministic shard key (email hash) or maintain a small global lookup service/table mapping key→shard. Global secondary indexes are complex — if required, implement a separate index service with eventual consistency and a retry/validation strategy.

3. **Q: When would you choose materialized views or pre-aggregation over adding more indexes?**
   A: Use MVs when queries involve expensive aggregations or joins that indexes cannot accelerate. MVs trade storage and refresh complexity for fast reads; choose incremental/refresh policies that match your freshness SLAs.

4. **Q: What's your strategy for building/rebuilding indexes on very large tables in production?**
   A: Use online/concurrent index builds where supported; build on replicas and cut over, or perform rolling reindexes with minimal locking. Monitor IO/latency, throttle builders, and schedule during low-traffic windows.

5. **Q: An index exists but the planner isn't using it — what do you check?**
   A: Run `EXPLAIN`/`EXPLAIN ANALYZE`, validate stats (`ANALYZE`), check selectivity, column order, function usage (e.g., `lower(col)` vs `col`), implicit casts, and whether the index is covering the needed columns.

6. **Q: What are the operational risks of using external search systems (Elasticsearch) as indexes and how do you mitigate them?**
   A: Risks: eventual consistency, data drift, schema mapping issues, and operational overhead. Mitigate with robust CDC pipelines, idempotent writes, monitoring, reconciliation jobs, and fallbacks to DB queries for critical reads.

7. **Q: How do you keep index bloat under control (Postgres example)?**
   A: Monitor bloat and index usage (`pg_stat_user_indexes`), use `VACUUM`/`REINDEX` when needed, tune `fillfactor` to reduce page splits, avoid extremely wide indexes, and consider partitioning or partial indexes to limit index size.

8. **Q: Composite vs multiple single-column indexes — how do you decide?**
   A: Prefer a composite index when queries filter/sort on multiple columns together (left-prefix). Rely on index intersection only if composite isn't practical; composite indexes reduce I/O and avoid extra lookups.
