# Core Module: API Security

**Token Estimate:** ~2,500 tokens
**Purpose:** Secure API endpoints and prevent common API vulnerabilities
**Dependencies:** core/00_overview.md

---

## Objective

Ensure API endpoints are properly authenticated, rate-limited, and protected against common API attacks (CORS, CSRF, rate limiting, etc.).

---

## 1. Rate Limiting

```
Grep patterns:
- "rate.*limit|throttle|rateLimit"
- "express-rate-limit|rate-limiter-flexible"
```

**Verify:**
- ✅ Rate limiting on authentication endpoints (login, signup, password reset)
- ✅ Rate limiting on expensive operations (AI generation, file uploads)
- ✅ Per-user/IP/API key limits
- ✅ Appropriate limits per endpoint type

**Flag P1 if:**
- No rate limiting on auth endpoints
- No rate limiting on resource-intensive operations

**Example:**
```typescript
// ✅ SECURE - Rate limiting
import rateLimit from 'express-rate-limit'

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts'
})

app.post('/api/login', loginLimiter, async (req, res) => {
  // Login logic
})
```

---

## 2. CORS Configuration

```
Grep pattern: "cors|Access-Control-Allow"
```

**Verify:**
- ✅ CORS origins whitelisted (not `*` in production)
- ✅ Credentials properly configured
- ✅ Methods restricted to needed only
- ✅ Headers minimized

**Flag P1 if:**
- CORS allows `*` origin in production
- Credentials allowed with `*` origin

**Example:**
```typescript
// ❌ INSECURE - Allows any origin
app.use(cors({ origin: '*', credentials: true }))

// ✅ SECURE - Whitelist specific origins
app.use(cors({
  origin: ['https://myapp.com', 'https://app.myapp.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}))
```

---

## 3. Security Headers

```
Check for security headers:
Grep pattern: "helmet|securityHeaders|setHeader"
```

**Required headers:**
- ✅ `X-Content-Type-Options: nosniff`
- ✅ `X-Frame-Options: DENY` or `SAMEORIGIN`
- ✅ `X-XSS-Protection: 1; mode=block`
- ✅ `Strict-Transport-Security: max-age=31536000`
- ✅ `Content-Security-Policy` (if applicable)
- ✅ Remove `X-Powered-By` header

**Example:**
```typescript
// ✅ Use helmet for security headers
import helmet from 'helmet'
app.use(helmet())

// Or manually:
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff')
  res.setHeader('X-Frame-Options', 'DENY')
  res.setHeader('X-XSS-Protection', '1; mode=block')
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains')
  res.removeHeader('X-Powered-By')
  next()
})
```

---

## 4. Request Size Limits

```
Check body parser configuration:
Grep pattern: "bodyParser|express\.json|body.*limit"
```

**Verify:**
- ✅ Request size limits configured
- ✅ Different limits per endpoint type
- ✅ Protection against memory exhaustion

**Example:**
```typescript
// ✅ SECURE - Size limits
app.use(express.json({ limit: '1mb' })) // Default
app.use('/api/upload', express.json({ limit: '10mb' })) // File uploads
```

---

## 5. API Versioning

```
Check API structure:
Glob: **/api/v{1,2,3}/**/*
```

**Verify:**
- ✅ APIs versioned (e.g., /api/v1/)
- ✅ Old versions deprecated gracefully
- ✅ Breaking changes in new versions only

---

## 6. Error Handling

```
Grep pattern: "createError|throw new Error|catch"
```

**Verify:**
- ✅ Generic error messages to clients
- ✅ No stack traces in production
- ✅ No internal paths disclosed
- ✅ Errors logged server-side with context

**Example:**
```typescript
// ❌ BAD - Leaks implementation details
catch (error) {
  res.status(500).json({
    error: error.message, // "Connection to database failed at 192.168.1.5"
    stack: error.stack
  })
}

// ✅ GOOD - Generic error, log details
catch (error) {
  console.error('API Error:', error) // Log full error
  res.status(500).json({
    error: 'Internal server error' // Generic to client
  })
}
```

---

## 7. Content Negotiation

```
Check Accept header handling:
Grep pattern: "accept|content-type"
```

**Verify:**
- ✅ Only expected content types accepted
- ✅ Proper content type in responses
- ✅ No automatic XML parsing (XXE risk)

---

## 8. API Documentation Security

```
Check for API docs:
Grep pattern: "swagger|openapi|api.*docs"
```

**Verify:**
- ✅ Docs protected in production (or public if intended)
- ✅ No sensitive endpoints exposed
- ✅ Example requests don't contain real credentials

---

## 9. Webhook Security

```
Find webhook endpoints:
Grep pattern: "webhook|callback|notify"
```

**Verify:**
- ✅ Signature verification (HMAC-SHA256)
- ✅ Timestamp validation (prevent replay)
- ✅ Idempotency handling
- ✅ Rate limiting

**Example:**
```typescript
// ✅ SECURE - Webhook verification
function verifyWebhook(req) {
  const signature = req.headers['x-webhook-signature']
  const timestamp = req.headers['x-webhook-timestamp']
  
  // Check timestamp (5 minute window)
  if (Math.abs(Date.now() - timestamp) > 5 * 60 * 1000) {
    return false
  }
  
  // Verify HMAC
  const payload = timestamp + JSON.stringify(req.body)
  const expected = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(payload)
    .digest('hex')
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  )
}
```

---

## 10. GraphQL Security (If Applicable)

```
Check GraphQL setup:
Grep pattern: "graphql|apollo"
```

**Verify:**
- ✅ Query depth limiting
- ✅ Query complexity limits
- ✅ Introspection disabled in production
- ✅ Proper authentication on resolvers

---

## 11. REST Best Practices

**Endpoint Security:**
- ✅ Use HTTPS only
- ✅ Proper HTTP methods (GET/POST/PUT/DELETE)
- ✅ Idempotency for non-GET requests
- ✅ Pagination on list endpoints
- ✅ Filtering/sorting server-side only

**Response Structure:**
```typescript
// ✅ Consistent response format
{
  "data": {...},
  "meta": {
    "page": 1,
    "total": 100
  },
  "error": null
}
```

---

## API Security Checklist

- [ ] Rate limiting on all public endpoints
- [ ] CORS properly configured
- [ ] Security headers set
- [ ] Request size limits enforced
- [ ] Generic error messages
- [ ] HTTPS enforced
- [ ] API keys/tokens validated
- [ ] Proper authentication on all endpoints
- [ ] Audit logging on sensitive operations
- [ ] Webhooks verify signatures

---

**Severity Summary:**
- **P0:** No authentication on sensitive endpoints
- **P1:** Missing rate limiting, open CORS, no security headers
- **P2:** Missing pagination, no API versioning
