# Quick Performance Checklist

**Token Estimate:** ~800 tokens
**Purpose:** Rapid performance audit checklist
**Use when:** Performance review, optimization sprint, pre-launch check

---

## How to Use

1. Work through each section
2. Check ✅ if optimized, ❌ if needs improvement
3. Note slow queries/endpoints for investigation
4. For detailed strategies, see full modules below

---

## Critical Performance Issues

### Database Queries
- [ ] No N+1 queries (check: loops with `await db.query()`)
- [ ] All foreign keys have indexes
- [ ] WHERE clause columns indexed
- [ ] ORDER BY columns indexed
- [ ] Composite indexes for multi-column queries
- [ ] All queries complete in < 100ms (check with EXPLAIN ANALYZE)

### Pagination
- [ ] All list endpoints have pagination (LIMIT/OFFSET or cursor)
- [ ] Default limit ≤ 100 items
- [ ] No `SELECT * FROM table` without LIMIT

### SELECT Optimization
- [ ] SELECT only needed columns (not `SELECT *`)
- [ ] Covering indexes for frequent queries
- [ ] No unnecessary JOINs

---

## Caching Opportunities

### Application Cache
- [ ] Configuration data cached (pricing, settings)
- [ ] Rarely-changing data cached (< 1 change/hour)
- [ ] External API responses cached
- [ ] Cache invalidation strategy implemented
- [ ] Cache hit rate > 80% for static data

### Browser/CDN Cache
- [ ] Static assets cached (max-age: 1 year)
- [ ] Cache-Control headers set appropriately
- [ ] ETags for semi-static content
- [ ] CDN enabled for public assets/APIs

### Session/Auth Cache
- [ ] User sessions in Redis (not database)
- [ ] Auth checks cached (not DB query every request)

---

## API Response Times

### Targets:
- [ ] List endpoints: < 200ms
- [ ] Detail endpoints: < 100ms
- [ ] Create/Update: < 300ms
- [ ] Search: < 500ms

### Optimization:
- [ ] Slow queries identified (> 100ms)
- [ ] Parallel async operations where possible
- [ ] Database connection pooling configured

---

## Frontend Performance

### Bundle Size
- [ ] JS bundle < 500KB gzipped
- [ ] Not using full lodash (use lodash-es or individual imports)
- [ ] Not using moment.js (use date-fns or dayjs)
- [ ] Code splitting implemented
- [ ] Lazy loading for routes

### Images
- [ ] Images lazy-loaded
- [ ] Responsive images (srcset)
- [ ] Modern formats (WebP, AVIF)
- [ ] Dimensions specified (avoid layout shift)
- [ ] CDN for image delivery

### Rendering
- [ ] No blocking JS in `<head>`
- [ ] CSS optimized and minified
- [ ] Critical CSS inlined
- [ ] Font loading optimized (font-display: swap)

---

## Advanced Indexing

### Index Coverage
- [ ] All foreign keys indexed
- [ ] Composite indexes for common query patterns
- [ ] Partial indexes for filtered queries
- [ ] Expression indexes for computed columns (if applicable)

### Index Health
- [ ] No unused indexes (idx_scan = 0)
- [ ] Index bloat managed
- [ ] ANALYZE run regularly

### Special Indexes
- [ ] GIN indexes for JSONB/array queries (if applicable)
- [ ] Full-text search indexes (if applicable)
- [ ] GiST indexes for geospatial (if applicable)

---

## Strategic Caching

### Multi-Layer Strategy
- [ ] Browser cache (static assets)
- [ ] CDN cache (public APIs, images)
- [ ] Application cache (Redis for dynamic data)
- [ ] Database query cache

### Cache Patterns
- [ ] Time-based invalidation (TTL)
- [ ] Event-based invalidation (on update)
- [ ] Cache stampede prevention
- [ ] Cache hit rate monitored

---

## External Services

### API Calls
- [ ] External API responses cached
- [ ] Timeout configured (< 10s)
- [ ] Retry logic with exponential backoff
- [ ] Parallel requests for independent data

### Third-Party Scripts
- [ ] Analytics loaded asynchronously
- [ ] Social widgets lazy-loaded
- [ ] Only essential third-party scripts

---

## Monitoring

### Metrics Tracked:
- [ ] API response times (p50, p95, p99)
- [ ] Database query times
- [ ] Cache hit rates
- [ ] Error rates
- [ ] Throughput (requests/second)

### Alerting:
- [ ] Alerts for slow queries (> 1s)
- [ ] Alerts for high error rates (> 1%)
- [ ] Alerts for low cache hit rate (< 70%)

---

## Quick Benchmarks

### Database:
```sql
-- Check slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Check missing indexes
SELECT schemaname, tablename, attname
FROM pg_stats
WHERE n_distinct > 100 AND correlation < 0.5
  AND schemaname = 'public';

-- Check table/index sizes
SELECT
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### API Response Times:
```bash
# Measure endpoint performance
time curl https://api.example.com/projects

# Load test
artillery quick --count 100 --num 10 https://api.example.com/projects

# Monitor during load
watch -n 1 'curl -s https://api.example.com/health | jq'
```

### Cache Hit Rate:
```bash
# Redis stats
redis-cli info stats | grep keyspace_hits
redis-cli info stats | grep keyspace_misses

# Calculate hit rate
# hit_rate = hits / (hits + misses) * 100
```

---

## Performance Targets by Endpoint Type

| Endpoint Type | Target | P95 Target | Action if Slower |
|---------------|--------|------------|------------------|
| Static asset | < 50ms | < 100ms | CDN + long cache |
| List (paginated) | < 200ms | < 400ms | Add index, cache |
| Detail (by ID) | < 100ms | < 200ms | Index PK, cache |
| Search | < 500ms | < 1s | Full-text index |
| Create/Update | < 300ms | < 600ms | Optimize writes |
| Aggregation | < 100ms | < 200ms | Materialized view |

---

## ROI Estimates

### High Impact (10-100x improvement):
- Adding missing indexes
- Implementing caching on config/pricing
- Fixing N+1 queries
- CDN for static assets

### Medium Impact (2-10x improvement):
- Query optimization (SELECT specific columns)
- Composite indexes
- Database connection pooling
- Compression enabled

### Low Impact (10-50% improvement):
- Frontend bundle optimization
- Image optimization
- Minor query tweaks

---

## Priority Actions

### If response time > 1s:
1. Check database query time (EXPLAIN ANALYZE)
2. Look for N+1 queries
3. Check for missing indexes
4. Consider caching

### If response time 500ms-1s:
1. Optimize query (SELECT, JOIN)
2. Add composite indexes
3. Enable caching
4. Check external API calls

### If response time 200-500ms:
1. Fine-tune indexes
2. Implement query result caching
3. Optimize frontend bundle

---

## Scoring Guide

Count your ✅ checks:

- **90-100%**: Excellent performance
- **75-89%**: Good, room for optimization
- **60-74%**: Moderate issues, address caching/indexing
- **< 60%**: Performance problems, immediate attention needed

---

## See Full Modules For Details

- **Performance Basics:** `core/05_performance_basics.md`
- **Strategic Caching:** `specialized/strategic_caching.md`
- **Advanced Indexing:** `specialized/advanced_indexing.md`

---

## Quick Wins (Implement in < 1 hour)

1. **Add cache headers** to static assets (1 line change)
2. **Add index** on most-queried foreign key (1 SQL command)
3. **Implement Redis cache** for config data (10 lines)
4. **Add pagination** to unbounded list endpoint (5 lines)
5. **Enable compression** (1 config change)

---

## Performance Optimization Workflow

1. **Measure:** Identify slow endpoints/queries
2. **Profile:** EXPLAIN ANALYZE on slow queries
3. **Optimize:** Add indexes, caching, or query improvements
4. **Measure again:** Verify improvement
5. **Monitor:** Track over time
6. **Repeat:** Continuous optimization

---

**Remember:** Premature optimization is the root of all evil. Optimize based on data, not assumptions.
