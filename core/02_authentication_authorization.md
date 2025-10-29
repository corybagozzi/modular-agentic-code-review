# Core Module: Authentication & Authorization

**Token Estimate:** ~3,000 tokens
**Purpose:** Verify authentication and authorization security
**Dependencies:** core/00_overview.md, core/01_reconnaissance.md

---

## Objective

Ensure all authentication mechanisms are secure, sessions are properly managed, and authorization checks prevent privilege escalation and unauthorized access.

---

## 1. Authentication Mechanism Audit

### 1.1 Password Hashing

```
Grep patterns:
- "bcrypt|scrypt|argon2|pbkdf2"
- "password.*hash|hashPassword"
- "MD5|SHA1|SHA256" (flag as weak)
```

**Check:**
- ✅ Uses industry-standard hashing (bcrypt, argon2, scrypt)
- ✅ Work factor ≥ 12 for bcrypt
- ✅ Salt is unique per password (not global)
- ✅ No custom password hashing implementation

**Flag P0 if:**
- Uses MD5, SHA1, or unsalted SHA256
- Passwords stored in plaintext
- Custom crypto implementation

**Example:**
```typescript
// ❌ WEAK - SHA256 without salt
const hash = crypto.createHash('sha256').update(password).digest('hex')

// ✅ SECURE - bcrypt with proper work factor
const hash = await bcrypt.hash(password, 12)
```

### 1.2 Session Management

```
Grep patterns:
- "session|jwt|token"
- "cookie.*httpOnly|secure"
- "sessionSecret|jwtSecret"
```

**Verify:**
- ✅ Sessions regenerated on login (prevent fixation)
- ✅ Sessions invalidated on logout
- ✅ Session timeout configured (< 24 hours for sensitive apps)
- ✅ httpOnly and secure flags on session cookies
- ✅ CSRF protection enabled

**Flag P0 if:**
- Session IDs in URLs
- No session timeout
- Session fixation possible

**Flag P1 if:**
- Session timeout > 24 hours
- No CSRF protection
- Session cookies without httpOnly flag

### 1.3 JWT Validation

```
Grep patterns:
- "jwt\.verify|jwt\.decode"
- "jsonwebtoken|jose"
```

**Check:**
- ✅ JWT signature verified server-side
- ✅ Expiration checked (exp claim)
- ✅ Issuer validated (iss claim)
- ✅ Algorithm explicitly specified (no "none" algorithm)
- ✅ Secret stored securely (not hardcoded)

**Example:**
```typescript
// ❌ VULNERABLE - Doesn't verify signature
const decoded = jwt.decode(token) // No verification!

// ✅ SECURE - Verifies signature and expiration
const decoded = jwt.verify(token, secret, {
  algorithms: ['HS256'],  // Explicit algorithm
  issuer: 'myapp',
  maxAge: '1h'
})
```

---

## 2. Authorization & Access Control

### 2.1 Role-Based Access Control (RBAC)

```
Grep patterns:
- "role|permission|ability"
- "requireAdmin|requireRole|authorize"
- "canAccess|hasPermission"
```

**Verify:**
- ✅ Roles checked server-side (not trusted from client)
- ✅ Role hierarchy properly enforced
- ✅ Default deny (explicit permission required)
- ✅ Least privilege principle applied

**Flag P0 if:**
- Roles accepted from request body/headers
- No server-side role validation
- Client-side only authorization checks

**Example:**
```typescript
// ❌ VULNERABLE - Trusts client-provided role
if (req.body.isAdmin) {
  // Grant admin access
}

// ✅ SECURE - Checks database for actual role
const user = await db.users.findOne({ id: req.userId })
if (user.role === 'admin') {
  // Grant admin access
}
```

### 2.2 Object-Level Authorization

```
Check ALL endpoints with [id] parameters:
Glob: **/api/**/[id]*.{ts,js,py,rb}
```

**For each endpoint, verify:**
- ✅ User owns the resource OR has permission
- ✅ Organization/tenant membership checked
- ✅ Cannot access by guessing IDs

**Flag P0 if:**
- No ownership verification
- Direct ID access without checks

**Example:**
```typescript
// ❌ BOLA/IDOR - No ownership check
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({ id: req.params.id })
  return res.json(project) // Any authenticated user can access!
})

// ✅ SECURE - Verifies ownership
app.get('/api/projects/:id', async (req, res) => {
  const project = await db.projects.findOne({
    id: req.params.id,
    userId: req.user.id  // Must belong to requesting user
  })
  if (!project) return res.status(404).json({ error: 'Not found' })
  return res.json(project)
})
```

### 2.3 Privilege Escalation Prevention

```
Grep patterns:
- "admin.*role|grant.*permission|promote"
- "UPDATE.*users.*role|UPDATE.*permissions"
```

**Check:**
- ✅ Users cannot grant themselves admin role
- ✅ Permission changes require existing admin
- ✅ Audit logging for privilege changes
- ✅ Database constraints prevent direct updates

**Flag P0 if:**
- User can modify their own role
- No audit logging on role changes
- Direct database updates without validation

---

## 3. Authentication Endpoint Security

### 3.1 Login Endpoint

```
Find login/signin endpoints
Grep pattern: "login|signin|authenticate"
```

**Verify:**
- ✅ Rate limiting (prevent brute force)
- ✅ Account lockout after N failed attempts
- ✅ Generic error messages ("Invalid credentials", not "User not found")
- ✅ Password requirements enforced
- ✅ MFA support (especially for admin accounts)

**Flag P1 if:**
- No rate limiting
- Detailed error messages leak user existence
- No account lockout mechanism

### 3.2 Registration Endpoint

```
Find signup/register endpoints
Grep pattern: "register|signup|create.*user"
```

**Verify:**
- ✅ Email verification required
- ✅ Password strength requirements
- ✅ Rate limiting (prevent spam)
- ✅ CAPTCHA or similar bot protection
- ✅ No auto-admin role assignment

**Flag P1 if:**
- No email verification
- Weak password requirements
- No bot protection

### 3.3 Password Reset

```
Find password reset endpoints
Grep pattern: "reset.*password|forgot.*password"
```

**Verify:**
- ✅ Reset tokens are cryptographically random
- ✅ Tokens expire (< 1 hour)
- ✅ Tokens single-use only
- ✅ Rate limiting on reset requests
- ✅ Email verification before reset

**Flag P0 if:**
- Predictable reset tokens
- No token expiration
- Tokens can be reused

---

## 4. Multi-Tenant Isolation (If Applicable)

### 4.1 Tenant Context

```
Check how tenant/organization is determined
Grep patterns:
- "organization|tenant|workspace"
- "req\.org|ctx\.tenant|current.*organization"
```

**Verify:**
- ✅ Tenant determined from authenticated user
- ✅ Tenant cannot be overridden by client
- ✅ All queries scoped to current tenant
- ✅ No cross-tenant data leakage

**Flag P0 if:**
- Tenant ID accepted from request
- Queries not scoped to tenant
- Cross-tenant access possible

### 4.2 Row-Level Security (If using PostgreSQL/Supabase)

```
Check for RLS policies:
Grep pattern: "ENABLE ROW LEVEL SECURITY|CREATE POLICY"
```

**For each tenant-scoped table:**
- ✅ RLS enabled
- ✅ Policies for SELECT, INSERT, UPDATE, DELETE
- ✅ Policies use auth context (auth.uid(), current_user, etc.)
- ✅ Test coverage for RLS policies

**Flag P0 if:**
- Tenant-scoped tables without RLS
- RLS policies allow cross-tenant access

*Note: For comprehensive RLS audit, use specialized/multi_tenant_rls.md*

---

## 5. API Key Security (If Applicable)

```
Check for API key authentication:
Grep pattern: "api.*key|bearer.*token|x-api-key"
```

**Verify:**
- ✅ API keys are hashed (not plaintext)
- ✅ Keys use sufficient entropy (≥128 bits)
- ✅ Prefix lookup for performance
- ✅ Last used timestamp tracked
- ✅ Keys can be revoked

**Flag P0 if:**
- API keys stored in plaintext
- Weak key generation (< 128 bits)

**Example:**
```typescript
// ❌ WEAK - Predictable keys
const apiKey = `${userId}-${Date.now()}`

// ✅ SECURE - Cryptographically random
const apiKey = crypto.randomBytes(32).toString('base64url')
const hash = await bcrypt.hash(apiKey, 12)
// Store hash, return apiKey once
```

---

## 6. Third-Party Authentication

### 6.1 OAuth / Social Login

```
Check OAuth implementation:
Grep pattern: "oauth|passport|auth0|google.*auth|github.*auth"
```

**Verify:**
- ✅ State parameter used (CSRF protection)
- ✅ Redirect URI whitelist enforced
- ✅ Token exchange done server-side
- ✅ ID tokens validated

**Flag P1 if:**
- No state parameter
- Open redirect possible via redirect_uri

### 6.2 SAML / Enterprise SSO

```
Check SAML implementation:
Grep pattern: "saml|sso|idp"
```

**Verify:**
- ✅ Assertions validated (signature check)
- ✅ Audience restriction enforced
- ✅ Timestamp checked (NotBefore, NotOnOrAfter)
- ✅ SSL/TLS for metadata exchange

---

## 7. Testing Requirements

**Must have tests for:**
- [ ] Cannot access resources without authentication
- [ ] Cannot access other users' resources
- [ ] Cannot escalate own privileges
- [ ] Session timeout works
- [ ] Password reset flow secure
- [ ] Rate limiting prevents brute force
- [ ] Admin endpoints protected

---

## Common Vulnerabilities Checklist

- [ ] **Authentication Bypass:** Can access without credentials?
- [ ] **Session Fixation:** Session ID changes on login?
- [ ] **Privilege Escalation:** Can user grant themselves admin?
- [ ] **BOLA/IDOR:** Can access resources by guessing IDs?
- [ ] **Weak Passwords:** Are requirements enforced?
- [ ] **No MFA:** Is MFA available for sensitive accounts?
- [ ] **Token Leakage:** Are tokens in URLs or logs?
- [ ] **Insecure Cookies:** httpOnly, secure, sameSite set?

---

**Severity Summary:**
- **P0:** Authentication bypass, privilege escalation, password storage issues
- **P1:** Missing rate limiting, weak session management, no MFA for admins
- **P2:** Missing audit logs, weak password requirements
