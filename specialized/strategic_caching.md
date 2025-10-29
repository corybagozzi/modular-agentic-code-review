# Specialized Module: Strategic Caching

**Token Estimate:** ~3,000 tokens
**Purpose:** Implement multi-layer caching for performance and cost reduction
**Dependencies:** core/05_performance_basics.md
**Best For:** High-traffic applications, expensive computations, external API calls

---

## Objective

Identify strategic caching opportunities to improve performance by 10-100x and reduce infrastructure costs by 50-90%.

---

## 1. Caching Layers Overview

### 1.1 Multi-Layer Strategy

```
Browser Cache (100-1000ms saved)
    ↓ (on miss)
CDN Cache (10-100ms saved)
    ↓ (on miss)
Application Cache / Redis (1-10ms saved)
    ↓ (on miss)
Database Query Cache (5-50ms saved)
    ↓ (on miss)
Database
```

**Performance impact by layer:**
- Browser: Eliminates network request entirely
- CDN: Serves from edge location (10-100ms vs 200-500ms)
- App/Redis: In-memory lookup (1-10ms vs 50-200ms DB query)
- DB Query Cache: Avoids disk I/O (5-50ms vs 50-200ms)

---

## 2. Layer 1: Browser Caching

### 2.1 HTTP Cache Headers

```
Identify cacheable assets:
Grep: "res\\.set|res\\.header|Cache-Control"
```

**Static assets (images, fonts, JS, CSS):**

```typescript
// ✅ SECURE - Long cache for immutable assets
app.use('/assets', express.static('public/assets', {
  maxAge: '365d',  // 1 year
  immutable: true,
  setHeaders: (res, path) => {
    if (path.endsWith('.js') || path.endsWith('.css')) {
      res.setHeader('Cache-Control', 'public, max-age=31536000, immutable')
    }
  }
}))
```

**Dynamic content with versioning:**

```typescript
// ✅ SECURE - Cache with ETag validation
app.get('/api/config', async (req, res) => {
  const config = await getConfig()
  const etag = generateETag(config)

  res.setHeader('Cache-Control', 'public, max-age=300')  // 5 minutes
  res.setHeader('ETag', etag)

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).send()  // Not Modified
  }

  res.json(config)
})
```

**Cache-Control strategies:**
```typescript
// Static assets (versioned filenames)
'public, max-age=31536000, immutable'

// Frequently updated (config, pricing)
'public, max-age=300, must-revalidate'  // 5 min

// User-specific data
'private, max-age=60'  // 1 min, not shared

// Never cache
'no-store, no-cache, must-revalidate'  // Auth endpoints
```

**Flag opportunity if:**
- Static assets without cache headers
- max-age = 0 on unchanging resources
- No ETags on semi-static content

**ROI:** 50-90% reduction in server requests for static content

---

## 3. Layer 2: CDN Caching

### 3.1 CDN Configuration

**Identify CDN usage:**
```
Check configuration for:
- Cloudflare
- Vercel Edge Network
- AWS CloudFront
- Fastly
```

**Optimal CDN strategy:**

```typescript
// ✅ SECURE - CDN-friendly headers
export default defineEventHandler(async (event) => {
  const pricing = await getPricingTiers()

  // Cache at CDN for 1 hour
  setResponseHeader(event, 'Cache-Control', 'public, s-maxage=3600, max-age=300')
  // s-maxage = CDN cache (1 hour)
  // max-age = browser cache (5 minutes)

  // Cache tag for purging
  setResponseHeader(event, 'Cache-Tag', 'pricing')

  return pricing
})

// When pricing changes, purge CDN:
await fetch('https://api.cloudflare.com/purge', {
  method: 'POST',
  body: JSON.stringify({ tags: ['pricing'] })
})
```

**What to cache at CDN:**
- ✅ API responses (pricing, config, public data)
- ✅ Generated images/assets
- ✅ Static pages
- ❌ User-specific data
- ❌ Authenticated endpoints

**Flag opportunity if:**
- Public APIs without CDN caching
- Generated images served directly from app
- No cache purging strategy

**ROI:** 10-50x faster response times globally (200-500ms → 20-50ms)

---

## 4. Layer 3: Application-Level Caching

### 4.1 In-Memory Cache (Small Scale)

```typescript
// ✅ Simple in-memory cache for config
const cache = new Map<string, { data: any, expires: number }>()

async function getCachedConfig(key: string, ttlMs = 60000) {
  const cached = cache.get(key)

  if (cached && cached.expires > Date.now()) {
    return cached.data
  }

  const data = await db.config.findMany()

  cache.set(key, {
    data,
    expires: Date.now() + ttlMs
  })

  // Cleanup old entries
  if (cache.size > 100) {
    for (const [k, v] of cache.entries()) {
      if (v.expires < Date.now()) cache.delete(k)
    }
  }

  return data
}
```

**Use for:**
- Configuration data
- Pricing tiers
- Feature flags
- Rarely-changing reference data

**Flag opportunity if:**
- Database queried every request for config
- Same data fetched repeatedly within request

**ROI:** 10-100x faster (50-200ms DB query → 1-5ms memory lookup)

---

### 4.2 Redis Cache (Production Scale)

```typescript
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

// ✅ SECURE - Redis with proper TTL
async function getCachedData<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds = 300
): Promise<T> {
  // Try cache first
  const cached = await redis.get(key)
  if (cached) {
    return JSON.parse(cached)
  }

  // Fetch from source
  const data = await fetcher()

  // Store in cache
  await redis.setex(key, ttlSeconds, JSON.stringify(data))

  return data
}

// Usage:
const pricing = await getCachedData(
  'pricing:tiers',
  () => db.pricingTiers.findMany(),
  3600  // 1 hour
)
```

**Cache invalidation patterns:**

```typescript
// Pattern 1: TTL (Time-based)
await redis.setex('config', 300, data)  // Expires in 5 min

// Pattern 2: Event-based invalidation
async function updatePricing(newPricing) {
  await db.pricingTiers.update(newPricing)
  await redis.del('pricing:tiers')  // Invalidate cache
}

// Pattern 3: Cache tags
await redis.set('user:123:profile', data)
await redis.sadd('cache:tag:user:123', 'user:123:profile')

async function invalidateUser(userId: string) {
  const keys = await redis.smembers(`cache:tag:user:${userId}`)
  if (keys.length > 0) {
    await redis.del(...keys)
  }
}
```

**Flag opportunity if:**
- Expensive queries executed every request
- External API calls not cached
- User sessions in database instead of Redis
- No cache invalidation strategy

**ROI:** 50-100x faster for expensive operations, 90% cost reduction on external APIs

---

## 5. Strategic Caching Patterns

### 5.1 Query Result Caching

```typescript
// ❌ NO CACHE - Hits DB every request
app.get('/api/pricing', async (req, res) => {
  const tiers = await db.pricingTiers.findMany()
  res.json(tiers)
})

// ✅ WITH CACHE - Hits DB every 1 hour
app.get('/api/pricing', async (req, res) => {
  const tiers = await getCachedData(
    'pricing:tiers',
    () => db.pricingTiers.findMany(),
    3600
  )
  res.json(tiers)
})
```

**What to cache:**
- Configuration tables (action_costs, pricing_tiers)
- Lookup data (countries, timezones)
- Aggregated analytics
- Rarely-changing user profiles

**What NOT to cache:**
- Real-time data (current balance, pending jobs)
- Personalized feeds
- Security-sensitive data

---

### 5.2 Computed Value Caching

```typescript
// ❌ EXPENSIVE - Computes every request
app.get('/api/dashboard', async (req, res) => {
  const stats = {
    totalJobs: await db.jobs.count(),
    completedJobs: await db.jobs.count({ where: { status: 'completed' } }),
    avgProcessingTime: await db.jobs.aggregate({ _avg: { processing_time: true } })
  }
  res.json(stats)
})

// ✅ CACHED - Computes every 5 minutes
app.get('/api/dashboard', async (req, res) => {
  const stats = await getCachedData(
    'stats:global',
    async () => ({
      totalJobs: await db.jobs.count(),
      completedJobs: await db.jobs.count({ where: { status: 'completed' } }),
      avgProcessingTime: await db.jobs.aggregate({ _avg: { processing_time: true } })
    }),
    300  // 5 minutes
  )
  res.json(stats)
})
```

**ROI:** 100x faster (3 separate queries → 1 cached object)

---

### 5.3 External API Response Caching

```typescript
// ❌ NO CACHE - Calls OpenAI every time
async function getModelPricing() {
  const response = await fetch('https://api.openai.com/v1/models')
  return response.json()
}

// ✅ CACHED - Calls OpenAI once per day
async function getModelPricing() {
  return getCachedData(
    'openai:models',
    async () => {
      const response = await fetch('https://api.openai.com/v1/models')
      return response.json()
    },
    86400  // 24 hours
  )
}
```

**ROI:** 90% reduction in external API costs + 10x faster response

---

### 5.4 User Session Caching

```typescript
// ❌ SLOW - Database lookup every request
app.use(async (req, res, next) => {
  const session = await db.sessions.findOne({ token: req.headers.authorization })
  req.user = await db.users.findOne({ id: session.userId })
  next()
})

// ✅ FAST - Redis session cache
app.use(async (req, res, next) => {
  const userId = await getCachedData(
    `session:${req.headers.authorization}`,
    async () => {
      const session = await db.sessions.findOne({ token: req.headers.authorization })
      return session.userId
    },
    3600
  )

  req.user = await getCachedData(
    `user:${userId}`,
    () => db.users.findOne({ id: userId }),
    600
  )

  next()
})
```

**ROI:** 10x faster auth (50ms DB → 5ms Redis)

---

## 6. Cache Invalidation Strategies

### 6.1 Proactive Invalidation

```typescript
// ✅ SECURE - Invalidate on update
async function updateActionCost(id: string, newCost: number) {
  // Update database
  await db.actionCosts.update({
    where: { id },
    data: { token_cost: newCost }
  })

  // Invalidate all related caches
  await redis.del('costs:all')
  await redis.del(`cost:${id}`)

  // Purge CDN cache
  await fetch('https://api.cloudflare.com/purge', {
    method: 'POST',
    body: JSON.stringify({ tags: ['costs'] })
  })
}
```

---

### 6.2 Cache Stampede Prevention

```typescript
// ❌ CACHE STAMPEDE - All requests refetch simultaneously
async function getExpensiveData(key: string) {
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)

  // 100 concurrent requests all hit DB at same time!
  const data = await expensiveDBQuery()
  await redis.setex(key, 3600, JSON.stringify(data))
  return data
}

// ✅ PREVENTS STAMPEDE - Lock during refetch
import Redlock from 'redlock'

const redlock = new Redlock([redis])

async function getExpensiveData(key: string) {
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)

  // Acquire lock
  const lock = await redlock.lock(`lock:${key}`, 5000)

  try {
    // Double-check cache (another request may have filled it)
    const cached2 = await redis.get(key)
    if (cached2) return JSON.parse(cached2)

    // Only ONE request executes this
    const data = await expensiveDBQuery()
    await redis.setex(key, 3600, JSON.stringify(data))
    return data
  } finally {
    await lock.unlock()
  }
}
```

---

## 7. Monitoring Cache Performance

### 7.1 Cache Hit Rate Tracking

```typescript
// ✅ SECURE - Monitor cache effectiveness
let cacheHits = 0
let cacheMisses = 0

async function getCachedDataWithMetrics<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl = 300
): Promise<T> {
  const cached = await redis.get(key)

  if (cached) {
    cacheHits++
    return JSON.parse(cached)
  }

  cacheMisses++

  const data = await fetcher()
  await redis.setex(key, ttl, JSON.stringify(data))
  return data
}

// Report metrics
app.get('/api/admin/cache-stats', requireAdmin, (req, res) => {
  const total = cacheHits + cacheMisses
  const hitRate = total > 0 ? (cacheHits / total * 100).toFixed(2) : 0

  res.json({
    hits: cacheHits,
    misses: cacheMisses,
    hit_rate: `${hitRate}%`,
    total_requests: total
  })
})
```

**Target metrics:**
- Hit rate > 80% for static data
- Hit rate > 50% for dynamic data
- Cache miss latency < 200ms

---

## 8. Security Considerations

### 8.1 Cache Poisoning Prevention

```typescript
// ❌ VULNERABLE - User can poison cache
app.get('/api/config', async (req, res) => {
  const locale = req.query.locale  // User-controlled!
  const config = await getCachedData(`config:${locale}`, fetchConfig)
  // Attacker: ?locale=en../../admin creates bad cache key
})

// ✅ SECURE - Sanitize cache keys
function sanitizeCacheKey(input: string): string {
  return input.replace(/[^a-zA-Z0-9:-]/g, '')
}

app.get('/api/config', async (req, res) => {
  const locale = sanitizeCacheKey(req.query.locale || 'en')
  const config = await getCachedData(`config:${locale}`, fetchConfig)
})
```

---

### 8.2 Sensitive Data in Cache

```typescript
// ❌ DANGEROUS - Caches user-specific data with shared key
const balance = await getCachedData('user:balance', () =>
  db.users.findOne({ id: userId })
)

// ✅ SECURE - User-specific cache keys
const balance = await getCachedData(`user:${userId}:balance`, () =>
  db.users.findOne({ id: userId })
)

// ✅ SECURE - Private cache control
res.setHeader('Cache-Control', 'private, max-age=60')
```

---

## Strategic Caching Checklist

- [ ] Static assets cached with long max-age (1 year)
- [ ] Semi-static data cached with ETags (config, pricing)
- [ ] CDN enabled for public APIs and assets
- [ ] Redis/Memcached for application cache
- [ ] Configuration data cached (not queried every request)
- [ ] External API responses cached
- [ ] User sessions in Redis, not database
- [ ] Cache invalidation strategy implemented
- [ ] Cache stampede prevention on expensive queries
- [ ] Cache hit rate monitored (target > 80%)
- [ ] Sensitive data uses user-specific cache keys
- [ ] Cache keys sanitized to prevent poisoning

---

## Cache Layer Selection Guide

| Data Type | Browser | CDN | Redis | DB Cache |
|-----------|---------|-----|-------|----------|
| Static assets | ✅ 1yr | ✅ 1yr | ❌ | ❌ |
| Pricing/Config | ✅ 5min | ✅ 1hr | ✅ 1hr | ❌ |
| User profile | ✅ 1min | ❌ | ✅ 10min | ✅ |
| Auth session | ❌ | ❌ | ✅ 1hr | ❌ |
| Real-time data | ❌ | ❌ | ❌ | ❌ |

---

**Expected ROI:**
- 50-90% reduction in database load
- 10-100x faster response times for cached data
- 90% reduction in external API costs
- 80%+ cache hit rate on static/config data

**Common Mistakes:**
1. Caching without invalidation strategy
2. Shared cache keys for user-specific data
3. No TTL on cache entries (memory leak)
4. Caching authenticated/personalized responses at CDN
5. Not monitoring cache hit rates

**See Also:**
- core/05_performance_basics.md - Basic performance patterns
- specialized/advanced_indexing.md - Database optimization
