# Specialized Module: SSRF & BOLA Detection

**Token Estimate:** ~3,000 tokens
**Purpose:** Prevent Server-Side Request Forgery and Broken Object Level Authorization
**Dependencies:** core/02_authentication_authorization.md, core/04_api_security.md
**Best For:** API-heavy applications with external integrations

---

## Objective

Detect and prevent SSRF (Server-Side Request Forgery) attacks and BOLA/IDOR (Broken Object Level Authorization / Insecure Direct Object Reference) vulnerabilities.

---

## Part 1: SSRF Prevention

## 1. External Request Identification

### 1.1 Find All Outbound HTTP Requests

```
Grep patterns:
- "fetch\\(|axios\\.|http\\.get|https\\.get|request\\("
- "URL\\(|new URL|url\\.parse"
- "curl|wget" (if shell commands used)
```

**Categorize requests:**
- **User-controlled URLs** - CRITICAL risk
- **Partially user-controlled** (path/query only) - HIGH risk
- **Fixed URLs** (hardcoded) - LOW risk

---

## 2. URL Validation & Sanitization

### 2.1 Whitelist Approach

```typescript
// ❌ VULNERABLE - No validation
app.post('/api/fetch-avatar', async (req, res) => {
  const response = await fetch(req.body.url)  // ANY URL!
  return res.json(await response.json())
})
// Attacker: { url: "http://169.254.169.254/latest/meta-data/iam/security-credentials" }
// ^ AWS metadata endpoint - leaks credentials!

// ✅ SECURE - Whitelist domains
const ALLOWED_DOMAINS = [
  'api.example.com',
  'cdn.example.com',
  's3.amazonaws.com'
]

async function fetchExternalResource(url: string) {
  const parsed = new URL(url)

  // Check protocol
  if (!['https:', 'http:'].includes(parsed.protocol)) {
    throw new Error('Invalid protocol')
  }

  // Check domain whitelist
  if (!ALLOWED_DOMAINS.some(domain => parsed.hostname.endsWith(domain))) {
    throw new Error('Domain not allowed')
  }

  // Block internal IPs (even if DNS resolves to them)
  await validateNotInternalIP(parsed.hostname)

  return fetch(url, {
    timeout: 5000,
    redirect: 'manual'  // Don't follow redirects automatically
  })
}
```

**Flag P0 if:**
- User-provided URLs fetched without validation
- No domain whitelist
- No internal IP blocking

---

### 2.2 Internal IP Blocking

```typescript
import { isIP } from 'net'
import dns from 'dns/promises'

// ✅ SECURE - Blocks internal IPs
async function validateNotInternalIP(hostname: string): Promise<void> {
  // Resolve hostname to IP
  let ips: string[]
  try {
    ips = await dns.resolve4(hostname)
  } catch {
    // If DNS fails, block
    throw new Error('Could not resolve hostname')
  }

  // Check each resolved IP
  for (const ip of ips) {
    if (isInternalIP(ip)) {
      throw new Error('Internal IP address not allowed')
    }
  }
}

function isInternalIP(ip: string): boolean {
  // Private IPv4 ranges
  const patterns = [
    /^10\./,                    // 10.0.0.0/8
    /^172\.(1[6-9]|2[0-9]|3[0-1])\./, // 172.16.0.0/12
    /^192\.168\./,              // 192.168.0.0/16
    /^127\./,                   // 127.0.0.0/8 (localhost)
    /^169\.254\./,              // 169.254.0.0/16 (link-local)
    /^0\./,                     // 0.0.0.0/8
    /^::1$/,                    // IPv6 localhost
    /^fe80:/,                   // IPv6 link-local
    /^fc00:/,                   // IPv6 private
  ]

  return patterns.some(pattern => pattern.test(ip))
}
```

**Flag P0 if:**
- No DNS resolution check
- Internal IPs (127.0.0.1, 169.254.*, 10.*, 192.168.*) accessible
- Localhost aliases (localhost, 0.0.0.0) not blocked

---

### 2.3 SSRF via Redirect

```typescript
// ❌ VULNERABLE - Follows redirects to internal IPs
const response = await fetch(url, { redirect: 'follow' })
// Attacker URL: https://evil.com/redirect-to-metadata
// ^ Redirects to http://169.254.169.254/

// ✅ SECURE - Manually handle redirects
async function fetchWithRedirectValidation(url: string, maxRedirects = 3) {
  let currentUrl = url
  let redirectCount = 0

  while (redirectCount < maxRedirects) {
    // Validate each URL in chain
    await validateURL(currentUrl)

    const response = await fetch(currentUrl, { redirect: 'manual' })

    if (response.status >= 300 && response.status < 400) {
      const location = response.headers.get('location')
      if (!location) break

      // Validate redirect target
      currentUrl = new URL(location, currentUrl).href
      redirectCount++
    } else {
      return response
    }
  }

  throw new Error('Too many redirects')
}
```

**Flag P0 if:**
- `redirect: 'follow'` used with user URLs
- Redirect targets not validated

---

### 2.4 DNS Rebinding Protection

```typescript
// ✅ SECURE - Re-validates IP after initial check
async function fetchWithRebindingProtection(url: string) {
  const parsed = new URL(url)

  // First check
  await validateNotInternalIP(parsed.hostname)

  // Small delay to detect rebinding
  await sleep(100)

  // Second check before actual request
  await validateNotInternalIP(parsed.hostname)

  // Use short-lived connection
  return fetch(url, {
    timeout: 3000,
    // Consider using a proxy that validates again
  })
}
```

---

## 3. Cloud Metadata Endpoint Protection

### 3.1 Block Common Metadata URLs

```typescript
// ✅ SECURE - Blocks cloud metadata endpoints
const BLOCKED_URLS = [
  // AWS
  'http://169.254.169.254/latest/meta-data',
  'http://169.254.169.254/latest/user-data',
  'http://169.254.169.254/latest/dynamic',

  // Google Cloud
  'http://metadata.google.internal',
  'http://169.254.169.254/computeMetadata',

  // Azure
  'http://169.254.169.254/metadata',

  // DigitalOcean
  'http://169.254.169.254/metadata',

  // Kubernetes
  'https://kubernetes.default.svc',
]

function isMetadataURL(url: string): boolean {
  const normalized = url.toLowerCase()
  return BLOCKED_URLS.some(blocked =>
    normalized.startsWith(blocked.toLowerCase())
  )
}

if (isMetadataURL(url)) {
  throw new Error('Access to metadata endpoints blocked')
}
```

**Flag P0 if:**
- Cloud metadata endpoints accessible
- Running on AWS/GCP/Azure without metadata protection

---

## 4. Webhook Security (SSRF via Webhooks)

### 4.1 Validate Webhook URLs

```typescript
// ❌ VULNERABLE - User sets webhook to internal service
app.post('/api/webhooks', async (req, res) => {
  const webhook = await db.webhooks.create({
    url: req.body.url,  // ANY URL!
    project_id: req.project.id
  })
})
// Later: fetch(webhook.url) // SSRF!

// ✅ SECURE - Validate webhook URLs
async function createWebhook(url: string, projectId: string) {
  // Parse and validate
  const parsed = new URL(url)

  // Must be HTTPS
  if (parsed.protocol !== 'https:') {
    throw new Error('Webhooks must use HTTPS')
  }

  // Block internal IPs
  await validateNotInternalIP(parsed.hostname)

  // Block metadata endpoints
  if (isMetadataURL(url)) {
    throw new Error('Invalid webhook URL')
  }

  // Test webhook is reachable (with timeout)
  try {
    await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ test: true }),
      timeout: 5000,
      signal: AbortSignal.timeout(5000)
    })
  } catch {
    throw new Error('Webhook URL not reachable')
  }

  return db.webhooks.create({ url, project_id: projectId })
}
```

**Flag P0 if:**
- Webhook URLs not validated
- Can set webhook to internal services
- No HTTPS requirement

---

## 5. File Upload SSRF

### 5.1 URL-based File Uploads

```typescript
// ❌ VULNERABLE - Server fetches user-provided URL
app.post('/api/upload-from-url', async (req, res) => {
  const response = await fetch(req.body.imageUrl)
  const buffer = await response.buffer()
  // Save buffer...
})

// ✅ SECURE - Strict validation
async function uploadFromURL(imageUrl: string, userId: string) {
  // Validate URL
  const parsed = new URL(imageUrl)

  // Whitelist protocols
  if (parsed.protocol !== 'https:') {
    throw new Error('Only HTTPS URLs allowed')
  }

  // Block internal IPs
  await validateNotInternalIP(parsed.hostname)

  // Fetch with restrictions
  const response = await fetch(imageUrl, {
    method: 'GET',
    redirect: 'manual',
    timeout: 10000,
    headers: {
      'User-Agent': 'MyApp/1.0'
    }
  })

  // Validate content type
  const contentType = response.headers.get('content-type')
  if (!contentType?.startsWith('image/')) {
    throw new Error('URL must point to an image')
  }

  // Validate size
  const contentLength = response.headers.get('content-length')
  if (contentLength && parseInt(contentLength) > 5 * 1024 * 1024) {
    throw new Error('Image too large')
  }

  return response.buffer()
}
```

---

## Part 2: BOLA/IDOR Prevention

## 6. Object-Level Authorization Audit

### 6.1 Identify All [id] Endpoints

```
Find all parameterized routes:
Glob patterns:
- **/api/**/[id].{ts,js,py}
- **/api/**/[id]/**/*.{ts,js,py}
- **/api/**/:id/**/*.{ts,js,py}

Grep: "params\\.id|req\\.params|\\[id\\]"
```

**For EACH endpoint, verify authorization check:**

---

### 6.2 Authorization Check Pattern

```typescript
// ❌ BOLA/IDOR - No ownership check
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({ id: req.params.id })
  return res.json(project)
})
// ANY authenticated user can access ANY project!

// ✅ SECURE - Verifies ownership
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({
    id: req.params.id,
    organization_id: req.user.organizationId  // Ownership check
  })

  if (!project) {
    return res.status(404).json({ error: 'Not found' })
  }

  return res.json(project)
})

// ✅ BETTER - Use helper function
async function requireProjectAccess(projectId: string, userId: string) {
  const project = await db.projects.findOne({
    where: { id: projectId },
    include: {
      organization: {
        include: {
          memberships: {
            where: { user_id: userId }
          }
        }
      }
    }
  })

  if (!project || project.organization.memberships.length === 0) {
    throw new UnauthorizedError('Cannot access this project')
  }

  return project
}

app.get('/api/projects/:id', async (req, res) => {
  const project = await requireProjectAccess(req.params.id, req.user.id)
  return res.json(project)
})
```

**Flag P0 if:**
- Endpoint returns object by ID without ownership check
- Uses `findOne({ id })` without tenant/user filter
- Returns 404 for unauthorized instead of 403 (leaks existence)

---

### 6.3 Nested Resource Authorization

```typescript
// ❌ VULNERABLE - Only checks job, not project
app.get('/api/projects/:projectId/jobs/:jobId', async (req, res) => {
  const job = await db.jobs.findOne({ id: req.params.jobId })
  return res.json(job)
})
// User can access ANY job by guessing jobId!

// ✅ SECURE - Verifies full hierarchy
app.get('/api/projects/:projectId/jobs/:jobId', async (req, res) => {
  // Verify project access first
  await requireProjectAccess(req.params.projectId, req.user.id)

  // Then verify job belongs to that project
  const job = await db.jobs.findOne({
    id: req.params.jobId,
    project_id: req.params.projectId  // MUST match URL
  })

  if (!job) {
    return res.status(404).json({ error: 'Not found' })
  }

  return res.json(job)
})
```

**Flag P0 if:**
- Nested resources don't verify parent ownership
- Child resource queried without parent_id check

---

## 7. Mass Assignment & Parameter Tampering

### 7.1 Prevent Role Escalation via Updates

```typescript
// ❌ VULNERABLE - User can change own role
app.patch('/api/users/:id', async (req, res) => {
  await db.users.update({
    where: { id: req.params.id },
    data: req.body  // DANGER: Contains role: 'admin'!
  })
})

// ✅ SECURE - Whitelist allowed fields
const ALLOWED_USER_FIELDS = ['name', 'email', 'avatar_url']

app.patch('/api/users/:id', async (req, res) => {
  // Verify ownership
  if (req.params.id !== req.user.id && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Forbidden' })
  }

  // Filter to allowed fields only
  const updates = {}
  for (const field of ALLOWED_USER_FIELDS) {
    if (req.body[field] !== undefined) {
      updates[field] = req.body[field]
    }
  }

  await db.users.update({
    where: { id: req.params.id },
    data: updates
  })
})
```

**Flag P0 if:**
- Raw `req.body` used in updates
- No field whitelist
- Users can modify their own role/permissions

---

### 7.2 Hidden Field Manipulation

```typescript
// ❌ VULNERABLE - User changes pricing tier
app.post('/api/projects', async (req, res) => {
  const project = await db.projects.create({
    data: req.body  // Contains: pricing_tier: 'enterprise' !
  })
})

// ✅ SECURE - Set sensitive fields server-side
app.post('/api/projects', async (req, res) => {
  const project = await db.projects.create({
    data: {
      name: req.body.name,
      description: req.body.description,
      organization_id: req.user.organizationId,  // From auth
      pricing_tier: 'free',  // Default, not user-controlled
      created_by: req.user.id
    }
  })
})
```

---

## 8. Enumeration Prevention

### 8.1 Consistent Error Messages

```typescript
// ❌ LEAKS EXISTENCE - Different messages
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({ id: req.params.id })

  if (!project) {
    return res.status(404).json({ error: 'Project not found' })
  }

  if (project.organization_id !== req.user.organizationId) {
    return res.status(403).json({ error: 'Access denied' })
  }

  return res.json(project)
})
// Attacker knows project EXISTS if they get 403 instead of 404!

// ✅ SECURE - Consistent 404 for unauthorized
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({
    id: req.params.id,
    organization_id: req.user.organizationId
  })

  if (!project) {
    return res.status(404).json({ error: 'Not found' })
  }

  return res.json(project)
})
// Returns 404 whether project doesn't exist OR user doesn't have access
```

**Flag P1 if:**
- Different status codes reveal resource existence
- Error messages leak information ("Access denied" vs "Not found")

---

## 9. UUID vs Sequential IDs

### 9.1 Predictable ID Risk

```typescript
// ❌ HIGHER RISK - Sequential IDs easy to enumerate
// IDs: 1, 2, 3, 4, 5... (attacker can guess all)
id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true }

// ✅ LOWER RISK - UUIDs harder to enumerate
// IDs: a3f2c8d1-..., 9b7e4f2a-... (128-bit random)
id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4 }
```

**Note:** UUIDs reduce enumeration risk but **DO NOT replace authorization checks**

**Flag P2 if:**
- Sequential IDs used for sensitive resources
- No rate limiting on ID-based lookups

---

## 10. Testing Requirements

### 10.1 BOLA Test Suite

```typescript
describe('BOLA Prevention', () => {
  test('User A cannot access User B project', async () => {
    const projectB = await createProject({ orgId: orgB.id })

    const response = await request(app)
      .get(`/api/projects/${projectB.id}`)
      .set('Authorization', `Bearer ${userA.token}`)

    expect(response.status).toBe(404)  // Not 403!
  })

  test('Cannot modify resource by guessing ID', async () => {
    const projectB = await createProject({ orgId: orgB.id })

    const response = await request(app)
      .patch(`/api/projects/${projectB.id}`)
      .set('Authorization', `Bearer ${userA.token}`)
      .send({ name: 'Hacked' })

    expect(response.status).toBe(404)
  })

  test('Cannot escalate role via mass assignment', async () => {
    await request(app)
      .patch(`/api/users/${userA.id}`)
      .set('Authorization', `Bearer ${userA.token}`)
      .send({ role: 'admin' })

    const user = await db.users.findOne({ id: userA.id })
    expect(user.role).not.toBe('admin')
  })
})
```

---

## SSRF & BOLA Checklist

### SSRF Prevention:
- [ ] All outbound HTTP requests identified
- [ ] User-provided URLs validated against whitelist
- [ ] Internal IPs blocked (10.*, 192.168.*, 127.*, 169.254.*)
- [ ] Cloud metadata endpoints blocked
- [ ] Redirect validation implemented
- [ ] Webhook URLs validated before storing
- [ ] File upload URLs restricted to HTTPS + validated

### BOLA Prevention:
- [ ] All [id] endpoints audited
- [ ] Every endpoint has ownership check
- [ ] Nested resources verify parent ownership
- [ ] No mass assignment vulnerabilities
- [ ] Sensitive fields set server-side
- [ ] Consistent error messages (404, not 403)
- [ ] Test coverage for cross-user access attempts

---

**Severity Summary:**
- **P0:** SSRF to metadata endpoints, BOLA allowing cross-user access, privilege escalation via mass assignment
- **P1:** Missing URL validation, enumeration via error messages, weak authorization checks
- **P2:** Sequential IDs without rate limiting, inconsistent validation

**See Also:**
- core/02_authentication_authorization.md - Authentication patterns
- specialized/multi_tenant_rls.md - Database-level isolation
