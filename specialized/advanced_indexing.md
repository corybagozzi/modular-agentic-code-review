# Specialized Module: Advanced Database Indexing

**Token Estimate:** ~3,000 tokens
**Purpose:** Optimize database performance through strategic indexing
**Dependencies:** core/05_performance_basics.md
**Best For:** Database-heavy applications with complex queries

---

## Objective

Identify missing indexes, optimize existing ones, and implement advanced indexing strategies to reduce query times from seconds to milliseconds.

---

## 1. Index Audit

### 1.1 Find All Database Queries

```
Grep patterns:
- ORM queries: "\\.findMany\\(|\\.where\\(|\\.select\\("
- Raw SQL: "SELECT.*FROM|query\\(.*SELECT"
- Joins: "JOIN|include:|relations:"
```

**For each query, identify:**
- WHERE clause columns
- ORDER BY columns
- JOIN conditions
- GROUP BY columns

---

### 1.2 Check Existing Indexes

**PostgreSQL:**
```sql
-- List all indexes
SELECT
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- Find missing indexes on foreign keys
SELECT
  tc.table_name,
  kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1 FROM pg_indexes
    WHERE tablename = tc.table_name
    AND indexdef LIKE '%' || kcu.column_name || '%'
  );
```

**Flag P1 if:**
- Foreign keys without indexes
- WHERE clause columns without indexes
- ORDER BY columns without indexes

---

## 2. Index Types & When to Use

### 2.1 B-Tree Index (Default)

**Best for:**
- Equality comparisons (`=`, `!=`)
- Range queries (`<`, `>`, `BETWEEN`)
- Sorting (`ORDER BY`)
- Pattern matching with left-anchored patterns (`LIKE 'prefix%'`)

```sql
-- ✅ Standard B-tree index
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_projects_created_at ON projects(created_at);
```

---

### 2.2 Composite (Multi-Column) Indexes

**Column order matters!**

```sql
-- Query pattern: WHERE org_id = ? ORDER BY created_at DESC
CREATE INDEX idx_projects_org_created ON projects(organization_id, created_at DESC);

-- ✅ USES INDEX - Filters by org_id, then sorts by created_at
SELECT * FROM projects
WHERE organization_id = '123'
ORDER BY created_at DESC;

-- ✅ USES INDEX - Prefix matches (org_id only)
SELECT * FROM projects WHERE organization_id = '123';

-- ❌ DOESN'T USE INDEX - Not left-anchored
SELECT * FROM projects WHERE created_at > '2024-01-01';
```

**Index column order rules:**
1. **Equality first** (`=`) - Most selective
2. **Range second** (`<`, `>`, `BETWEEN`)
3. **Sort last** (`ORDER BY`)

**Example:**
```sql
-- Query: WHERE status = 'pending' AND created_at > X ORDER BY priority DESC
CREATE INDEX idx_jobs_status_created_priority
ON jobs(status, created_at DESC, priority DESC);
```

**Flag P2 if:**
- Separate indexes instead of composite (e.g., `idx_status` + `idx_created` instead of combined)

---

### 2.3 Partial Indexes

**Best for filtering on subset of data:**

```sql
-- ❌ WASTEFUL - Indexes all jobs
CREATE INDEX idx_jobs_created ON jobs(created_at);

-- ✅ EFFICIENT - Indexes only pending jobs
CREATE INDEX idx_pending_jobs_created
ON jobs(created_at)
WHERE status = 'pending';

-- Query benefits:
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 10;
```

**Common use cases:**
```sql
-- Active records only
CREATE INDEX idx_active_users ON users(last_login)
WHERE deleted_at IS NULL;

-- Failed jobs for retry
CREATE INDEX idx_failed_jobs ON jobs(created_at)
WHERE status = 'failed';

-- Unpaid invoices
CREATE INDEX idx_unpaid_invoices ON invoices(due_date)
WHERE paid_at IS NULL;
```

**ROI:** 50-90% smaller index size, faster writes, same query speed

---

### 2.4 Covering Indexes (Index-Only Scans)

**Include all queried columns in index:**

```sql
-- Query: SELECT id, name, email FROM users WHERE organization_id = ?

-- ❌ PARTIAL COVERAGE - Still needs table lookup
CREATE INDEX idx_users_org ON users(organization_id);

-- ✅ COVERING INDEX - No table lookup needed
CREATE INDEX idx_users_org_covering
ON users(organization_id)
INCLUDE (name, email);

-- PostgreSQL executes "Index Only Scan" (faster!)
```

**Verify with EXPLAIN:**
```sql
EXPLAIN ANALYZE
SELECT id, name, email
FROM users
WHERE organization_id = '123';

-- Look for: "Index Only Scan" instead of "Index Scan"
```

**ROI:** 2-5x faster for frequently-run queries

---

### 2.5 Expression Indexes

**For computed/transformed columns:**

```sql
-- Query: WHERE LOWER(email) = 'user@example.com'

-- ❌ DOESN'T USE INDEX - Function on column
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ✅ USES INDEX - Index on expression
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

**Other examples:**
```sql
-- JSONB field extraction
CREATE INDEX idx_metadata_type ON jobs((metadata->>'type'));

-- Date truncation
CREATE INDEX idx_jobs_created_date ON jobs(DATE(created_at));

-- Computed column
CREATE INDEX idx_users_full_name ON users((first_name || ' ' || last_name));
```

---

### 2.6 GIN Index (Full-Text Search, JSONB, Arrays)

```sql
-- JSONB queries
CREATE INDEX idx_metadata_gin ON jobs USING GIN (metadata);

-- Query:
SELECT * FROM jobs WHERE metadata @> '{"status": "active"}';

-- Array membership
CREATE INDEX idx_tags_gin ON posts USING GIN (tags);

-- Query:
SELECT * FROM posts WHERE tags @> ARRAY['typescript', 'security'];

-- Full-text search
CREATE INDEX idx_posts_search ON posts USING GIN (to_tsvector('english', content));

-- Query:
SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('security & performance');
```

**Best for:**
- JSONB queries (`@>`, `?`, `?&`, `?|`)
- Array operations
- Full-text search

---

### 2.7 GiST Index (Geospatial, Range Types)

```sql
-- PostGIS geospatial queries
CREATE INDEX idx_locations_geo ON locations USING GIST (coordinates);

-- Query:
SELECT * FROM locations
WHERE ST_DWithin(coordinates, ST_MakePoint(lat, lng), 1000);

-- Range types
CREATE INDEX idx_bookings_period ON bookings USING GIST (booking_period);

-- Query:
SELECT * FROM bookings WHERE booking_period && '[2024-01-01, 2024-01-31]';
```

---

## 3. Query-Specific Index Strategies

### 3.1 Pagination Queries

```sql
-- ❌ SLOW - No index on sort column
SELECT * FROM jobs
WHERE project_id = '123'
ORDER BY created_at DESC
LIMIT 20 OFFSET 100;

-- ✅ FAST - Composite index on filter + sort
CREATE INDEX idx_jobs_project_created ON jobs(project_id, created_at DESC);
```

**Cursor-based pagination (even faster):**
```sql
-- Index includes cursor column
CREATE INDEX idx_jobs_project_id_created ON jobs(project_id, id, created_at DESC);

-- Query:
SELECT * FROM jobs
WHERE project_id = '123'
  AND id > '999'  -- Cursor
ORDER BY id ASC
LIMIT 20;
```

---

### 3.2 Join Optimization

```sql
-- Query with join:
SELECT p.*, j.*
FROM projects p
JOIN jobs j ON j.project_id = p.id
WHERE p.organization_id = '123'
ORDER BY j.created_at DESC;

-- Required indexes:
CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_jobs_project_created ON jobs(project_id, created_at DESC);
```

**Flag P1 if:**
- Foreign keys used in JOINs without indexes
- Join queries take > 100ms

---

### 3.3 Aggregation Queries

```sql
-- Query: COUNT(*) with GROUP BY
SELECT status, COUNT(*)
FROM jobs
WHERE project_id = '123'
GROUP BY status;

-- ✅ OPTIMIZED INDEX
CREATE INDEX idx_jobs_project_status ON jobs(project_id, status);
```

**For complex aggregations, consider:**
```sql
-- Materialized view (pre-computed)
CREATE MATERIALIZED VIEW job_stats AS
SELECT
  project_id,
  status,
  COUNT(*) as count,
  AVG(processing_time) as avg_time
FROM jobs
GROUP BY project_id, status;

CREATE INDEX idx_job_stats_project ON job_stats(project_id);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY job_stats;
```

**ROI:** 100-1000x faster for complex aggregations

---

## 4. Index Maintenance

### 4.1 Identify Unused Indexes

**PostgreSQL:**
```sql
-- Find indexes never used
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Flag for removal if:**
- idx_scan = 0 (never used)
- Index size > 100MB
- Created > 30 days ago

**ROI:** Faster writes, less disk usage

---

### 4.2 Bloated Indexes

**Check for index bloat:**
```sql
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
  idx_scan,
  idx_tup_read / NULLIF(idx_scan, 0) as avg_rows_per_scan
FROM pg_stat_user_indexes
WHERE idx_scan > 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Rebuild bloated indexes:**
```sql
-- PostgreSQL
REINDEX INDEX CONCURRENTLY idx_jobs_created;

-- Or rebuild entire table
REINDEX TABLE CONCURRENTLY jobs;
```

**Schedule regular maintenance:**
```sql
-- Weekly during low traffic
REINDEX DATABASE myapp CONCURRENTLY;
```

---

## 5. Index Monitoring with EXPLAIN

### 5.1 EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT * FROM jobs
WHERE project_id = '123'
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;
```

**What to look for:**

```
✅ GOOD:
- "Index Scan" or "Index Only Scan"
- cost=0.43..8.45 (low cost)
- rows=10 (accurate estimate)
- actual time=0.123..0.145 ms (fast)

❌ BAD:
- "Seq Scan" on large tables
- cost=1000..50000 (high cost)
- rows=1000000 (large scan)
- actual time=234.567..890.123 ms (slow)
```

**Common issues:**
- **Seq Scan on large table** → Add index
- **Index Scan but slow** → Wrong index, need composite
- **rows estimate off by 10x** → Run ANALYZE on table

---

### 5.2 Query Performance Targets

| Query Type | Target Time | Action if Slower |
|------------|-------------|------------------|
| Lookup by ID | < 1ms | Check primary key index |
| Filtered list | < 10ms | Add composite index |
| Paginated list | < 20ms | Index on sort column |
| Simple join | < 50ms | Index foreign keys |
| Complex join | < 100ms | Denormalize or cache |
| Aggregation | < 100ms | Materialized view |

---

## 6. Index Design Patterns

### 6.1 Multi-Tenant Tables

```sql
-- All tenant queries include organization_id
-- ALWAYS put it first in composite indexes

-- ✅ CORRECT ORDER
CREATE INDEX idx_projects_org_created ON projects(organization_id, created_at DESC);
CREATE INDEX idx_jobs_org_status ON jobs(organization_id, status);

-- ❌ WRONG ORDER - Won't help tenant filtering
CREATE INDEX idx_projects_created_org ON projects(created_at DESC, organization_id);
```

---

### 6.2 Polymorphic Associations

```sql
-- jobs table with job_type (polymorphic)
CREATE TABLE jobs (
  id UUID PRIMARY KEY,
  job_type VARCHAR(50),  -- 'avatar', 'map', etc.
  ...
);

-- ✅ OPTIMIZED - Partial index per type
CREATE INDEX idx_avatar_jobs_created
ON jobs(created_at DESC)
WHERE job_type = 'avatar';

CREATE INDEX idx_map_jobs_created
ON jobs(created_at DESC)
WHERE job_type = 'map';
```

---

### 6.3 Time-Series Data

```sql
-- Logs table with timestamps
CREATE TABLE logs (
  id BIGSERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL,
  level VARCHAR(20),
  message TEXT
);

-- ✅ OPTIMIZED - BRIN index for time-series
CREATE INDEX idx_logs_created_brin ON logs USING BRIN (created_at);
-- BRIN = Block Range Index (tiny size, great for time-series)

-- For recent logs (hot partition)
CREATE INDEX idx_recent_logs ON logs(created_at DESC)
WHERE created_at > NOW() - INTERVAL '7 days';
```

**BRIN ROI:** 100x smaller than B-tree, 10x faster inserts

---

## 7. Common Index Anti-Patterns

### 7.1 Over-Indexing

```sql
-- ❌ TOO MANY - 5 indexes on same table
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users(name);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_org ON users(organization_id);

-- ✅ STRATEGIC - 2 composite indexes
CREATE INDEX idx_users_org_created ON users(organization_id, created_at DESC);
CREATE INDEX idx_users_email ON users(email);  -- Unique constraint
```

**Cost of over-indexing:**
- Slower INSERTs/UPDATEs (must update all indexes)
- Wasted disk space
- Slower backups

**Rule of thumb:** ≤ 5 indexes per table

---

### 7.2 Indexing Low-Cardinality Columns

```sql
-- ❌ WASTEFUL - Only 2-3 distinct values
CREATE INDEX idx_jobs_status ON jobs(status);
-- status values: 'pending', 'processing', 'completed', 'failed'

-- ✅ BETTER - Partial index on specific value
CREATE INDEX idx_pending_jobs ON jobs(created_at)
WHERE status = 'pending';
```

**When to index low-cardinality:**
- When used with high-cardinality column (composite)
- As partial index filtering on specific value

---

## 8. Testing Index Effectiveness

### 8.1 Before/After Benchmarks

```typescript
// Benchmark query performance
async function benchmarkQuery() {
  const iterations = 100

  console.time('Query time')
  for (let i = 0; i < iterations; i++) {
    await db.query(`
      SELECT * FROM jobs
      WHERE project_id = $1
      ORDER BY created_at DESC
      LIMIT 20
    `, [testProjectId])
  }
  console.timeEnd('Query time')
}

// Before index: ~50ms per query
// After composite index: ~2ms per query
// 25x improvement ✅
```

---

### 8.2 Load Testing with Indexes

```bash
# Artillery load test
artillery run --target https://api.myapp.com \
  --payload projects.csv \
  --config load-test.yml

# Monitor query performance:
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE query LIKE '%jobs%'
ORDER BY mean_exec_time DESC;
```

---

## Advanced Indexing Checklist

- [ ] All foreign keys indexed
- [ ] Composite indexes for multi-column WHERE clauses
- [ ] Partial indexes for filtered queries (status = 'pending')
- [ ] Covering indexes for frequently-run SELECT queries
- [ ] Expression indexes for function calls (LOWER, DATE, etc.)
- [ ] GIN indexes for JSONB/array queries
- [ ] No unused indexes (idx_scan = 0)
- [ ] All queries use indexes (no Seq Scan on large tables)
- [ ] Query times meet targets (< 10ms for lists, < 50ms for joins)
- [ ] Index bloat monitored and managed
- [ ] Regular ANALYZE runs scheduled

---

## Index Optimization Workflow

1. **Identify slow queries** - Monitor logs, pg_stat_statements
2. **Run EXPLAIN ANALYZE** - Check for Seq Scan
3. **Identify filter/sort columns** - WHERE, ORDER BY, JOIN
4. **Design index** - Composite, partial, or covering
5. **Create index CONCURRENTLY** - No table lock
6. **Verify improvement** - Re-run EXPLAIN, benchmark
7. **Monitor production** - Check idx_scan, query times
8. **Remove old index** - If replaced by better one

---

**Expected ROI:**
- 10-100x faster queries with proper indexes
- 50-90% reduction in query time for filtered lists
- 100-1000x faster for complex aggregations (materialized views)
- 2-5x faster with covering indexes (index-only scans)

**Common Mistakes:**
1. No index on foreign keys
2. Wrong column order in composite indexes
3. Separate indexes instead of composite
4. Indexing low-cardinality columns alone
5. Not using partial indexes for filtered queries
6. Never removing unused indexes
7. No EXPLAIN ANALYZE before/after

**See Also:**
- core/05_performance_basics.md - N+1 queries, basic pagination
- specialized/strategic_caching.md - Caching to avoid queries entirely
