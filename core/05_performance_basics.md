# Core Module: Performance Basics

**Token Estimate:** ~2,500 tokens
**Purpose:** Identify common performance bottlenecks
**Dependencies:** core/00_overview.md

---

## Objective

Detect N+1 queries, missing indexes, lack of caching, and other common performance issues that degrade user experience.

---

## 1. N+1 Query Detection

```
Grep pattern: "for.*await.*from\(|\.map\(.*await"
```

**Flag P1 if:**
- Loop contains database queries
- Should be batched or use joins

**Example:**
```typescript
// ❌ N+1 QUERY - Slow
const users = await db.users.findMany()
for (const user of users) {
  const posts = await db.posts.findMany({ where: { userId: user.id } })
  user.posts = posts
}

// ✅ SINGLE QUERY - Fast
const users = await db.users.findMany({
  include: { posts: true }
})

// ✅ BATCH QUERY - Alternative
const users = await db.users.findMany()
const userIds = users.map(u => u.id)
const posts = await db.posts.findMany({
  where: { userId: { in: userIds } }
})
```

---

## 2. Database Indexes

```
Check migrations for indexes:
Grep pattern: "CREATE INDEX|add_index"
```

**Essential indexes:**
- ✅ Foreign keys (user_id, post_id, etc.)
- ✅ Frequently queried columns (status, created_at)
- ✅ WHERE clause columns
- ✅ ORDER BY columns

**Flag P1 if:**
- Foreign keys without indexes
- Common query patterns without indexes

**Example:**
```sql
-- ❌ MISSING INDEX - Slow query
SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC

-- ✅ ADD COMPOSITE INDEX
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
```

---

## 3. SELECT * Anti-Pattern

```
Grep pattern: "select\('\*'\)|SELECT \*"
```

**Flag P2 if:**
- Selecting all columns when only few needed
- Especially on large tables

**Example:**
```typescript
// ❌ BAD - Loads unnecessary data
const users = await db.query('SELECT * FROM users')

// ✅ GOOD - Only needed columns
const users = await db.query('SELECT id, name, email FROM users')
```

---

## 4. Pagination

```
Check list endpoints for pagination:
Grep pattern: "\.findMany\(|\.find\(|SELECT.*FROM"
```

**Verify:**
- ✅ All list queries have LIMIT
- ✅ Default limit ≤ 100
- ✅ Pagination implemented (offset or cursor)

**Flag P1 if:**
- No pagination on list endpoints
- Can load thousands of records

**Example:**
```typescript
// ❌ NO PAGINATION - Loads all records
app.get('/api/users', async (req, res) => {
  const users = await db.users.findMany()
  res.json(users)
})

// ✅ CURSOR PAGINATION - Efficient
app.get('/api/users', async (req, res) => {
  const cursor = req.query.cursor
  const users = await db.users.findMany({
    take: 50,
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1
    }),
    orderBy: { createdAt: 'desc' }
  })
  res.json(users)
})
```

---

## 5. Caching Opportunities

```
Grep pattern: "cache|redis|memo"
```

**Should cache:**
- ✅ Rarely-changing data (config, pricing)
- ✅ Expensive computations
- ✅ External API responses
- ✅ Database query results

**Flag P2 if:**
- No caching on static data
- Every request hits database

**Example:**
```typescript
// ❌ NO CACHE - Queries DB every request
app.get('/api/config', async (req, res) => {
  const config = await db.config.findMany()
  res.json(config)
})

// ✅ WITH CACHE - 100x faster
const cache = new Map()

app.get('/api/config', async (req, res) => {
  let config = cache.get('config')
  
  if (!config) {
    config = await db.config.findMany()
    cache.set('config', config)
    setTimeout(() => cache.delete('config'), 60000) // 1 min TTL
  }
  
  res.json(config)
})
```

---

## 6. Frontend Performance

### 6.1 Bundle Size

```
Check package.json for heavy dependencies:
- moment.js (use date-fns instead)
- lodash (use lodash-es or individual imports)
```

**Flag P2 if:**
- Bundle size > 500KB gzipped
- Using full lodash
- Using moment.js

### 6.2 Image Optimization

```
Grep pattern: "<img|background-image"
```

**Verify:**
- ✅ Lazy loading enabled
- ✅ Responsive images (srcset)
- ✅ Modern formats (WebP, AVIF)
- ✅ Proper dimensions specified

---

## 7. API Response Times

**Targets:**
- List endpoints: < 200ms
- Detail endpoints: < 100ms
- Create/Update: < 300ms

**Measure with:**
```typescript
app.use((req, res, next) => {
  const start = Date.now()
  res.on('finish', () => {
    const duration = Date.now() - start
    if (duration > 1000) {
      console.warn(`Slow request: ${req.path} took ${duration}ms`)
    }
  })
  next()
})
```

---

## 8. Database Connection Pooling

```
Check database client configuration
```

**Verify:**
- ✅ Connection pool configured
- ✅ Pool size appropriate for load
- ✅ Connections reused, not created per request

---

## 9. Async/Parallel Operations

**Example:**
```typescript
// ❌ SEQUENTIAL - Slow
const user = await db.users.findOne({ id })
const posts = await db.posts.findMany({ userId: id })
const comments = await db.comments.findMany({ userId: id })

// ✅ PARALLEL - 3x faster
const [user, posts, comments] = await Promise.all([
  db.users.findOne({ id }),
  db.posts.findMany({ userId: id }),
  db.comments.findMany({ userId: id })
])
```

---

## Performance Checklist

- [ ] No N+1 queries
- [ ] Foreign keys have indexes
- [ ] Pagination on list endpoints
- [ ] Caching on static data
- [ ] SELECT only needed columns
- [ ] Bundle size reasonable
- [ ] Images optimized
- [ ] Database connection pooling
- [ ] Parallel async operations where possible

---

**Severity Summary:**
- **P1:** N+1 queries on hot paths, missing indexes on FKs
- **P2:** Missing pagination, no caching, SELECT * on large tables

**Note:** For advanced optimization, see:
- specialized/strategic_caching.md
- specialized/advanced_indexing.md
