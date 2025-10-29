# Quick Security Checklist

**Token Estimate:** ~800 tokens
**Purpose:** Rapid security scan checklist
**Use when:** Quick review, pre-deployment check, security audit speedrun

---

## How to Use

1. Work through each section sequentially
2. Check ✅ if verified, ❌ if issue found
3. Note line numbers/files for any ❌ items
4. For detailed guidance, see full modules referenced below

---

## Critical (P0) - Fix Immediately

### Authentication & Authorization
- [ ] All API endpoints require authentication
- [ ] User cannot access other users' resources (test: try accessing `/api/projects/{other_user_id}`)
- [ ] Admin endpoints verify admin role server-side
- [ ] Passwords hashed with bcrypt/argon2 (work factor ≥ 12)
- [ ] Session/JWT secret is strong and in environment variables
- [ ] Service role/admin keys never in client-side code

### Database Security
- [ ] RLS enabled on all tenant-scoped tables (if using PostgreSQL/Supabase)
- [ ] All foreign keys have indexes
- [ ] No SQL injection: all queries use parameterization (no template literals with `${userInput}`)
- [ ] Service role usage has manual tenant checks

### Secrets & Environment
- [ ] No secrets hardcoded in code
- [ ] `.env` in `.gitignore`
- [ ] Production secrets different from development
- [ ] API keys have sufficient entropy (≥128 bits)

### Input Validation
- [ ] All user input validated server-side
- [ ] File uploads: type, size, and content validated
- [ ] Path traversal prevented (no `../` in file paths)
- [ ] Command injection prevented (no user input in shell commands)

### SSRF & External Requests
- [ ] User-provided URLs validated against whitelist
- [ ] Internal IPs blocked (127.*, 169.254.*, 10.*, 192.168.*)
- [ ] Cloud metadata endpoints blocked (169.254.169.254)

---

## Important (P1) - Fix Soon

### API Security
- [ ] Rate limiting on authentication endpoints (login, signup, password reset)
- [ ] Rate limiting on expensive operations (AI generation, file uploads)
- [ ] CORS properly configured (not `*` in production)
- [ ] Security headers set (X-Frame-Options, X-Content-Type-Options, HSTS)
- [ ] Request size limits enforced
- [ ] Generic error messages (no stack traces in production)

### Session & Token Management
- [ ] Sessions regenerated on login
- [ ] Sessions invalidated on logout
- [ ] Session timeout configured (< 24 hours)
- [ ] Cookies have httpOnly, secure, sameSite flags
- [ ] JWTs verify signature, check expiration
- [ ] CSRF protection enabled

### Input Validation
- [ ] XSS prevention: output encoding/escaping
- [ ] No `dangerouslySetInnerHTML` / `v-html` with user content
- [ ] NoSQL injection prevented (validate types)
- [ ] Schema validation on all endpoints (Zod, Yup, Joi)
- [ ] Email validation uses proper regex
- [ ] Phone numbers, URLs validated

### Authorization
- [ ] Object-level authorization on all [id] endpoints
- [ ] Nested resources verify parent ownership
- [ ] Users cannot modify their own role
- [ ] Mass assignment prevented (whitelist allowed fields)

### AI/LLM Security (if applicable)
- [ ] User input sanitized before prompts (delimiters, special tokens)
- [ ] Prompt injection patterns detected
- [ ] AI output validated
- [ ] Prompt templates are admin-only
- [ ] Token limits enforced

---

## Recommended (P2) - Improve When Possible

### Performance
- [ ] No N+1 queries (check loops with `await`)
- [ ] Pagination on all list endpoints
- [ ] SELECT only needed columns (not `SELECT *`)
- [ ] Caching on configuration/pricing data
- [ ] Database connection pooling configured

### Logging & Monitoring
- [ ] Security events logged (failed login, injection attempts)
- [ ] Audit trail for admin actions
- [ ] Error logging configured
- [ ] No sensitive data in logs (passwords, API keys)

### Code Quality
- [ ] Linting enabled and passing
- [ ] Type checking enabled (TypeScript/mypy/etc.)
- [ ] Tests for security-critical code
- [ ] RLS tests passing (if applicable)

### Dependencies
- [ ] No critical vulnerabilities in dependencies (`npm audit`, `pip check`)
- [ ] Dependencies up to date (or pinned with reason)
- [ ] Only necessary packages installed

---

## Quick Tests to Run

### Authentication:
```bash
# Test without auth token
curl https://api.example.com/projects

# Test with another user's ID
curl -H "Authorization: Bearer USER_A_TOKEN" \
  https://api.example.com/projects/USER_B_PROJECT_ID
```

### SQL Injection:
```bash
# Test SQL injection in parameters
curl "https://api.example.com/users?id=1' OR '1'='1"
curl "https://api.example.com/search?q='; DROP TABLE users--"
```

### Path Traversal:
```bash
# Test file access
curl "https://api.example.com/download?file=../../../../etc/passwd"
```

### SSRF:
```bash
# Test metadata endpoint access
curl -X POST https://api.example.com/fetch-url \
  -d '{"url": "http://169.254.169.254/latest/meta-data"}'
```

### Rate Limiting:
```bash
# Rapid requests
for i in {1..100}; do
  curl https://api.example.com/login \
    -d '{"email":"test@test.com","password":"wrong"}' &
done
```

---

## Priority Matrix

| Severity | Timeline | Examples |
|----------|----------|----------|
| **P0** | Fix today | SQL injection, auth bypass, RLS disabled |
| **P1** | Fix this week | Missing rate limiting, XSS, weak passwords |
| **P2** | Fix this sprint | No caching, SELECT *, unused indexes |

---

## Scoring Guide

Count your ✅ checks:

- **90-100%**: Excellent security posture
- **75-89%**: Good, minor improvements needed
- **60-74%**: Moderate risk, address P0/P1 items
- **< 60%**: High risk, immediate action required

---

## See Full Modules For Details

- **Authentication:** `core/02_authentication_authorization.md`
- **Input Validation:** `core/03_input_validation.md`
- **API Security:** `core/04_api_security.md`
- **RLS (Multi-tenant):** `specialized/multi_tenant_rls.md`
- **AI Security:** `specialized/ai_security.md`
- **SSRF & BOLA:** `specialized/ssrf_bola.md`
- **Command Injection:** `specialized/command_injection_redos.md`

---

## Emergency Response

If you find a P0 issue:

1. **Document:** Take screenshots, note reproduction steps
2. **Verify:** Test in production (if safe) or staging
3. **Fix:** Apply secure pattern from relevant module
4. **Test:** Verify vulnerability is closed
5. **Deploy:** Push fix immediately
6. **Review:** Check for similar issues elsewhere
7. **Post-mortem:** Document how it was missed

---

**Remember:** This is a quick scan. For comprehensive review, use full modules.
