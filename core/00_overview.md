# Core Module: Overview & Execution Protocol

**Token Estimate:** ~2,500 tokens
**Purpose:** Foundational guidance for all code reviews
**Required:** Always use this module first

---

## Execution Protocol

### Critical Requirements

1. **Use TodoWrite tool** to track ALL review phases
2. **Provide file paths and line numbers** for EVERY finding
3. **Include code snippets** for vulnerabilities
4. **Assign severity** (P0/P1/P2/P3) to ALL findings
5. **Document context** - Explain why it's an issue, not just what

### Tool Usage Reference

**File Operations:**
- `Read` - Read file contents (use for specific files)
- `Glob` - Find files by pattern (e.g., `**/*.sql`, `server/api/**/*.ts`)
- `Grep` - Search file contents (supports regex, use for code patterns)

**Code Exploration:**
- `Task` with `subagent_type: Explore` - Broad codebase exploration

**Task Management:**
- `TodoWrite` - Track review progress (REQUIRED)

### Review Flow

```
1. Initialize TodoWrite with review phases
2. Run reconnaissance (core/01_reconnaissance.md)
3. Apply relevant security modules
4. Apply performance modules (if needed)
5. Apply tech-specific patterns
6. Generate findings report
7. Update TodoWrite as completed
```

---

## Severity Definitions

### P0 - Critical (Fix Immediately)

**Characteristics:**
- **Active vulnerability** allowing unauthorized access
- **Data breach risk** or system compromise
- **Compliance violation** (GDPR, SOC2, PCI)

**Examples:**
- RLS missing on tenant-scoped tables
- Hardcoded secrets in code
- SQL injection vulnerability
- Admin bypass vulnerability
- SSRF allowing internal network access
- Command injection allowing code execution

**Timeline:** Fix within 24 hours
**Impact:** Data breach, system compromise, regulatory fines

---

### P1 - High (Fix Before Production)

**Characteristics:**
- **Potential vulnerability** or weak security control
- **Performance degradation** affecting UX
- **Missing critical safeguards**

**Examples:**
- Missing rate limiting on expensive operations
- Weak input validation
- No audit logging on admin actions
- BOLA/IDOR vulnerabilities
- N+1 queries on hot paths
- Missing indexes on foreign keys
- No RLS tests for tenant-scoped tables

**Timeline:** Fix within 1 week
**Impact:** Security incident likely, poor UX, system instability

---

### P2 - Medium (Fix Within 30 Days)

**Characteristics:**
- **Maintainability issues**
- **Performance concerns** (not critical)
- **Technical debt** accumulation

**Examples:**
- Code duplication
- Missing tests for non-critical features
- `SELECT *` on large tables
- Missing pagination
- No caching on static data
- Complex regex patterns (ReDoS risk)
- Excessive use of `any` type in TypeScript

**Timeline:** Fix within 30 days
**Impact:** Technical debt, slower development, gradual degradation

---

### P3 - Low (Backlog)

**Characteristics:**
- **Nice-to-have improvements**
- **Minor optimizations**
- **Non-critical suggestions**

**Examples:**
- Bundle size optimization
- Image format improvements
- Code style inconsistencies
- Minor performance tweaks
- Documentation improvements

**Timeline:** Backlog priority
**Impact:** Minimal current impact, future optimization opportunity

---

## Finding Report Format

For each finding, provide:

```markdown
---
**[P{0-3} - {Category}] {Brief Title}**

**File:** `path/to/file.ts:line_number`

**Issue:** {Clear description of the problem}

**Risk Impact:** {What could go wrong? Business/security/performance impact}

**Code:**
\`\`\`typescript
// ❌ VULNERABLE (current)
{show vulnerable code}

// ✅ SECURE (recommended)
{show secure code}
\`\`\`

**Fix:**
{Step-by-step fix instructions}

**Testing:**
{How to verify the fix works}

**Recommendation:** {Priority and next steps}
---
```

---

## Common Patterns to Flag

### Security Red Flags

```typescript
// ❌ No input validation
app.post('/api/user', (req, res) => {
  db.query(`SELECT * FROM users WHERE id = ${req.body.id}`)
})

// ❌ Hardcoded secrets
const API_KEY = 'sk-abc123xyz'

// ❌ No authentication check
app.get('/api/admin/users', async (req, res) => {
  return db.query('SELECT * FROM users')
})

// ❌ Client-controlled security decisions
if (req.body.isAdmin) {
  // Grant admin access
}

// ❌ No RLS on table
CREATE TABLE sensitive_data (...)
-- Missing: ENABLE ROW LEVEL SECURITY
```

### Performance Red Flags

```typescript
// ❌ N+1 query
for (const user of users) {
  const posts = await db.query('SELECT * FROM posts WHERE user_id = ?', user.id)
}

// ❌ No pagination
const allRecords = await db.query('SELECT * FROM large_table')

// ❌ Missing index
// Query: WHERE status = 'pending' AND created_at > ?
// Index: None on status or created_at

// ❌ SELECT * on large tables
const data = await db.query('SELECT * FROM jobs') // Returns megabytes
```

---

## Categories for Findings

**Security:**
- Authentication & Authorization
- Input Validation & Injection
- Data Protection & Privacy
- API Security
- Secrets Management
- Cryptography

**Performance:**
- Database Optimization
- Query Performance
- Caching Strategy
- Frontend Performance
- API Response Time

**Quality:**
- Code Maintainability
- Test Coverage
- Error Handling
- Type Safety
- Documentation

**Infrastructure:**
- CI/CD Security
- Deployment Configuration
- Dependency Management
- Environment Configuration

---

## Review Checklist Template

Use this at the start of every review:

```markdown
## Review Setup

**Project Type:** [SaaS / E-commerce / AI Application / etc.]
**Tech Stack:** [Framework + Database + Hosting]
**Review Scope:** [Full / Security Only / Performance Only / Feature Branch]
**Review Depth:** [Quick / Standard / Comprehensive]

## Modules Selected

- [ ] core/00_overview.md (this file)
- [ ] core/01_reconnaissance.md
- [ ] core/02_authentication_authorization.md
- [ ] core/03_input_validation.md
- [ ] core/04_api_security.md
- [ ] core/05_performance_basics.md
- [ ] specialized/[...]
- [ ] tech_stacks/[...]

## Expected Token Usage

- Modules: ~{X}K tokens
- Available for code: ~{Y}K tokens
- Sufficient: [Yes / No]

## Review Phases

- [ ] Phase 1: Reconnaissance
- [ ] Phase 2: Security Deep Scan
- [ ] Phase 3: Performance Analysis
- [ ] Phase 4: Code Quality
- [ ] Phase 5: Generate Report
```

---

## Tips for Effective Reviews

**DO:**
- ✅ Start with reconnaissance to understand the codebase
- ✅ Prioritize P0 and P1 findings
- ✅ Provide concrete examples and fixes
- ✅ Include file paths and line numbers
- ✅ Explain the "why" not just the "what"
- ✅ Group related findings together
- ✅ Test your assumptions with grep/glob
- ✅ Track progress with TodoWrite

**DON'T:**
- ❌ Skip reconnaissance phase
- ❌ Report findings without code examples
- ❌ Mix different severity levels without clear separation
- ❌ Provide vague recommendations ("improve security")
- ❌ Ignore false positives (note them as acceptable risks)
- ❌ Overwhelm with P3 findings (focus on P0/P1)

---

## Metric Tracking

Track these metrics for each review:

```markdown
## Review Metrics

**Findings by Severity:**
- P0: {count}
- P1: {count}
- P2: {count}
- P3: {count}
- Total: {count}

**Findings by Category:**
- Security: {count}
- Performance: {count}
- Quality: {count}
- Infrastructure: {count}

**Coverage:**
- Files reviewed: {count}
- Lines of code: {estimate}
- Modules used: {list}

**Risk Assessment:**
- Overall Risk Level: [Critical / High / Medium / Low]
- Blocker Issues: {P0 + P1 count}
- Estimated Fix Time: {hours/days}
```

---

## Example Review Initialization

```typescript
// Use TodoWrite to initialize review tracking

{
  "todos": [
    {
      "content": "Phase 1: Reconnaissance & Architecture Mapping",
      "status": "in_progress",
      "activeForm": "Mapping architecture"
    },
    {
      "content": "Phase 2: Security Deep Scan",
      "status": "pending",
      "activeForm": "Scanning security"
    },
    {
      "content": "Phase 3: Performance Analysis",
      "status": "pending",
      "activeForm": "Analyzing performance"
    },
    {
      "content": "Phase 4: Code Quality Review",
      "status": "pending",
      "activeForm": "Reviewing code quality"
    },
    {
      "content": "Phase 5: Generate Final Report",
      "status": "pending",
      "activeForm": "Generating report"
    }
  ]
}
```

---

**Next Step:** Proceed to `core/01_reconnaissance.md` to begin codebase analysis.
