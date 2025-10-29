# Modular Code Review System

A flexible, composable code review framework for Claude Code. Mix and match security and performance modules to create targeted reviews that fit your needs.

**📚 [See real-world examples →](./examples/)**

---

## ⚡ Quick Answer: "What if my tech stack isn't here?"

**No problem! You have 3 options:**

1. **Use core modules only** (tech-agnostic, works for any language/framework)
   ```bash
   cat core/00_overview.md core/02_authentication_authorization.md core/03_input_validation.md
   ```

2. **Create a basic tech stack module in 10 minutes** ([Step-by-step guide](./docs/CREATE_TECH_STACK_MODULE.md))
   ```bash
   cp tech_stacks/template.md tech_stacks/my_stack.md
   # Fill in top 5 framework-specific patterns
   ```

3. **Request a module** - Open an issue and we'll help create one for popular stacks

**Core modules work for:** Python, Ruby, PHP, Go, Java, C#, Rust, JavaScript, TypeScript, and any other language!

---

## 🚀 How to Use

### With Claude Code

1. **Clone this repository** next to your project or in a central location
2. **Compose modules** based on your review needs (see Quick Start below)
3. **Feed to Claude Code:**

```bash
# In your project directory
cat path/to/code-review-modules/core/00_overview.md \
    path/to/code-review-modules/core/02_authentication_authorization.md \
    path/to/code-review-modules/specialized/multi_tenant_rls.md
```

Then tell Claude Code:
```
"Review my codebase using the methodology above. Focus on RLS policies and authentication."
```

---

## 🎯 Quick Start

### Step 1: Choose Your Review Goal

<details>
<summary><b>🔒 Security Review (First-Time Audit)</b></summary>

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    core/03_input_validation.md \
    core/04_api_security.md \
    tech_stacks/[your-stack].md
```
**Finds:** Authentication issues, injection vulnerabilities, API security gaps
**Best for:** Pre-launch security audit, compliance checks
</details>

<details>
<summary><b>🏢 Multi-Tenant SaaS Audit</b></summary>

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    specialized/multi_tenant_rls.md \
    specialized/ssrf_bola.md \
    tech_stacks/nuxt_supabase.md
```
**Finds:** RLS policy issues, BOLA/IDOR vulnerabilities, tenant isolation gaps
**Best for:** Multi-tenant applications, SaaS platforms
</details>

<details>
<summary><b>🤖 AI/ML Application Security</b></summary>

```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    specialized/ai_security.md \
    specialized/strategic_caching.md
```
**Finds:** Prompt injection, data poisoning, cost optimization
**Best for:** Apps using LLM APIs
</details>

<details>
<summary><b>⚡ Performance Optimization</b></summary>

```bash
cat core/00_overview.md \
    core/05_performance_basics.md \
    specialized/strategic_caching.md \
    specialized/advanced_indexing.md
```
**Finds:** N+1 queries, missing indexes, caching opportunities
**Best for:** Slow applications, high database costs
</details>

<details>
<summary><b>✅ Quick PR Review (5 minutes)</b></summary>

```bash
cat checklists/security_quick.md \
    checklists/performance_quick.md
```
**Finds:** Critical P0/P1 issues only
**Best for:** Daily PR reviews, pre-commit checks
</details>

### Step 2: Run the Review

**Method 1: Direct Paste**
```bash
# Generate combined prompt
cat [modules above] > /tmp/review-prompt.md

# Copy to clipboard
cat /tmp/review-prompt.md | pbcopy  # macOS
cat /tmp/review-prompt.md | xclip   # Linux

# Paste into Claude Code and ask for review
```

**Method 2: Save as Project Template**
```bash
# Save your common combination
echo "cat core/00_overview.md \\" > .code-review.sh
echo "    core/02_authentication_authorization.md \\" >> .code-review.sh
echo "    tech_stacks/nuxt_supabase.md" >> .code-review.sh
chmod +x .code-review.sh

# Use anytime
./.code-review.sh | pbcopy  # macOS
./.code-review.sh | xclip   # Linux
```

---

## 📋 Module Selection Guide

### Decision Tree

```
START: What's your primary goal?

├─ 🔒 SECURITY REVIEW
│  ├─ Multi-tenant app? → Use specialized/multi_tenant_rls.md
│  ├─ AI/LLM integration? → Use specialized/ai_security.md
│  ├─ External API calls? → Use specialized/ssrf_bola.md
│  └─ General security? → Use core/02, 03, 04
│
├─ ⚡ PERFORMANCE REVIEW
│  ├─ Database slow? → Use specialized/advanced_indexing.md
│  ├─ API response slow? → Use specialized/strategic_caching.md
│  └─ General optimization? → Use core/05_performance_basics.md
│
├─ 🏗️ ARCHITECTURE REVIEW
│  └─ First-time codebase? → Use core/01_reconnaissance.md
│
└─ ✅ QUICK CHECK
   └─ PR review? → Use checklists/*.md
```

### By Tech Stack

| Your Stack | Recommended Modules |
|------------|-------------------|
| **Nuxt 3/4 + Supabase** | `tech_stacks/nuxt_supabase.md` + `specialized/multi_tenant_rls.md` |
| **Next.js + Vercel** | `tech_stacks/nextjs_vercel.md` + `core/04_api_security.md` |
| **Django + PostgreSQL** | [Create your own](./docs/CREATE_TECH_STACK_MODULE.md) + `specialized/multi_tenant_rls.md` |
| **Ruby on Rails** | [Create from template](./docs/CREATE_TECH_STACK_MODULE.md#ruby-on-rails--postgresql) |
| **Express + MongoDB** | [Create from template](./docs/CREATE_TECH_STACK_MODULE.md#express--mongodb) |
| **Laravel + MySQL** | [Create from template](./docs/CREATE_TECH_STACK_MODULE.md#laravel--mysql) |
| **Flask + PostgreSQL** | [Create from template](./docs/CREATE_TECH_STACK_MODULE.md#flask--postgresql) |
| **Spring Boot + Java** | [Create from template](./docs/CREATE_TECH_STACK_MODULE.md#spring-boot--postgresql) |
| **AI-powered (any stack)** | Use core modules + `specialized/ai_security.md` |

**Don't see your stack?** [Create a module in 10 minutes →](./docs/CREATE_TECH_STACK_MODULE.md)

---

## 📁 Module Structure

```
code-review-modules/
├── README.md                          # This file
├── core/                              # Tech-agnostic fundamentals
│   ├── 00_overview.md                 # ⭐ START HERE - Methodology & severity definitions
│   ├── 01_reconnaissance.md           # Architecture mapping (optional)
│   ├── 02_authentication_authorization.md  # Auth, sessions, RBAC, BOLA/IDOR
│   ├── 03_input_validation.md         # SQL injection, XSS, command injection
│   ├── 04_api_security.md             # Rate limiting, CORS, headers
│   └── 05_performance_basics.md       # N+1 queries, indexing, caching basics
├── specialized/                       # Deep dive modules
│   ├── multi_tenant_rls.md           # ⭐ Row-Level Security for multi-tenant apps
│   ├── ai_security.md                # ⭐ Prompt injection, data poisoning
│   ├── ssrf_bola.md                  # SSRF, BOLA/IDOR, webhook security
│   ├── command_injection_redos.md    # Command injection, ReDoS
│   ├── strategic_caching.md          # Multi-layer caching strategies
│   └── advanced_indexing.md          # Database optimization
├── tech_stacks/                      # Framework-specific patterns
│   ├── nuxt_supabase.md             # Nuxt 3/4 + Supabase patterns
│   ├── nextjs_vercel.md             # Next.js 13-15 + Vercel patterns
│   └── template.md                  # Template for creating new stacks
└── checklists/                       # Quick reference checklists
    ├── security_quick.md            # P0/P1 security checklist
    └── performance_quick.md         # Quick performance wins
```

**⭐ = Most frequently used modules**

---

## 🔧 Common Combinations

### By Project Phase

<details>
<summary><b>Daily PR Reviews</b></summary>

```bash
cat checklists/security_quick.md \
    checklists/performance_quick.md
```
**Time:** 5 minutes
**Focus:** Critical issues only
</details>

<details>
<summary><b>Feature Branch Review (Weekly)</b></summary>

```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    core/04_api_security.md \
    tech_stacks/[your-stack].md
```
**Time:** 15-20 minutes
**Focus:** Security + framework patterns
</details>

<details>
<summary><b>Release Review (Before Deploy)</b></summary>

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    core/03_input_validation.md \
    core/04_api_security.md \
    core/05_performance_basics.md \
    tech_stacks/[your-stack].md
```
**Time:** 30-40 minutes
**Focus:** Comprehensive security + performance
</details>

<details>
<summary><b>Comprehensive Audit (Quarterly)</b></summary>

```bash
cat core/*.md \
    specialized/*.md \
    tech_stacks/[your-stack].md
```
**Time:** 1-2 hours
**Focus:** Everything
</details>

### By Security Focus

**Authentication Audit:**
```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    specialized/multi_tenant_rls.md  # If multi-tenant
```

**API Security Review:**
```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    core/04_api_security.md \
    specialized/ssrf_bola.md
```

**AI/ML Security:**
```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    specialized/ai_security.md
```

### By Performance Focus

**Database Performance:**
```bash
cat core/00_overview.md \
    core/05_performance_basics.md \
    specialized/advanced_indexing.md
```

**Full Stack Performance:**
```bash
cat core/00_overview.md \
    core/05_performance_basics.md \
    specialized/strategic_caching.md \
    specialized/advanced_indexing.md
```

---

## 🎨 Customization

### Create Your Own Tech Stack Module

1. **Copy template:**
```bash
cp tech_stacks/template.md tech_stacks/my_stack.md
```

2. **Fill in framework-specific sections:**
   - Server route protection patterns
   - Environment variable handling
   - Authentication implementation
   - Common vulnerabilities for that stack
   - Performance optimization tips

3. **Example - Ruby on Rails:**
```markdown
# Tech Stack Overlay: Ruby on Rails

## 1. Controller Security

### 1.1 Strong Parameters

**Verify:** All controllers use strong parameters

```ruby
# ❌ VULNERABLE - Mass assignment
def create
  @user = User.create(params[:user])
end

# ✅ SECURE - Strong parameters
def create
  @user = User.create(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email)
end
```
```

4. **Use it:**
```bash
cat core/*.md tech_stacks/my_stack.md
```

### Create Project-Specific Override

For recurring project-specific patterns:

```bash
# Create custom module
cat > my_project_patterns.md << 'EOF'
# Project-Specific Patterns

## 1. Check Custom Auth Middleware

**Verify:** All `/api/admin/*` routes use `requireSuperAdmin()`

```typescript
// File: server/api/admin/**/*.ts
export default defineEventHandler(async (event) => {
  await requireSuperAdmin(event)  // ✅ Required
  // ... admin logic
})
```

## 2. Payment Endpoint Security

[Your project-specific checks]
EOF

# Use with standard modules
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    my_project_patterns.md
```

---

## 🆘 FAQ

<details>
<summary><b>Q: How is this different from automated security scanners?</b></summary>

**A:** This provides **contextual, logic-based review** that understands your business logic, not just pattern matching.

**Automated scanners** find:
- Hardcoded secrets
- Known vulnerable dependencies
- Simple regex patterns

**This modular system** finds:
- Business logic flaws (e.g., "User can access other user's data")
- Architecture issues (e.g., "Missing RLS on tenant table")
- Performance problems (e.g., "N+1 query in critical path")
- Context-specific vulnerabilities

**Best Practice:** Use both! Run automated scanners for quick wins, use this for deep review.
</details>

<details>
<summary><b>Q: Which modules should I start with?</b></summary>

**A:** Start with the quick checklist!

**Day 1:** `checklists/security_quick.md` (5 minutes)
- Gets you 80% of critical issues
- Builds muscle memory for what to check

**Week 1:** Add `core/02` and `core/03` (20 minutes)
- Covers authentication and input validation
- Most common vulnerability sources

**Month 1:** Add specialized modules based on your stack
- Multi-tenant? → `specialized/multi_tenant_rls.md`
- AI integration? → `specialized/ai_security.md`
</details>

<details>
<summary><b>Q: Can I use this for languages other than JavaScript/TypeScript?</b></summary>

**A:** Yes! Core modules are tech-agnostic.

**Supported via core modules:**
- Python, Ruby, PHP, Go, Java, C#, Rust, etc.

**Need framework-specific patterns?**
- Create a tech stack module using the template
- See `tech_stacks/template.md`

**Example:** For Django, focus on:
- `specialized/multi_tenant_rls.md` (if using PostgreSQL)
- `core/03_input_validation.md` (SQL injection via Django ORM)
- Create `tech_stacks/django_postgresql.md` for Django-specific patterns
</details>

<details>
<summary><b>Q: How often should I run reviews?</b></summary>

**A:** Depends on risk and velocity:

| Frequency | Use Case | Modules |
|-----------|----------|---------|
| **Every PR** | High-security apps | Quick checklists |
| **Weekly** | Most teams | Core modules |
| **Before releases** | All projects | Core + specialized |
| **Quarterly** | Compliance | Comprehensive audit |

**Recommendation:** Start with monthly, increase frequency as you find issues.
</details>

<details>
<summary><b>Q: What if I need a quicker review?</b></summary>

**A:** Use fewer modules or just the checklists:

1. **Use checklists only** - covers critical P0/P1 issues
2. **Review in chunks** - Run different modules on different days
3. **Focus on changed files** - Only review what's new

**Quick Review Example:**
```bash
cat checklists/security_quick.md
```
</details>

<details>
<summary><b>Q: Can I contribute my own modules?</b></summary>

**A:** Yes! Please do!

**How to contribute:**
1. Fork the repository
2. Create module in appropriate directory (`specialized/` or `tech_stacks/`)
3. Follow module format (see `tech_stacks/template.md`)
4. Include examples (vulnerable + secure code)
5. Submit PR with description of what it covers

**Most wanted modules:**
- Payment security (Stripe, PayPal patterns)
- Healthcare/HIPAA compliance
- GraphQL security
- WebSocket security
- Blockchain/Web3 patterns
- Mobile API security (iOS/Android)
</details>

<details>
<summary><b>Q: How do I know if the review was thorough enough?</b></summary>

**A:** Check coverage using the module checklists.

**Each module has a checklist at the end** - verify you've checked each item:

```markdown
## Security Checklist

- [x] All API endpoints require authentication
- [x] User cannot access other users' resources
- [x] Admin endpoints verify admin role
- [ ] Passwords hashed with bcrypt (work factor ≥ 12)  ← Need to check this
```

**Minimum for "thorough":**
- ✅ All P0 items checked
- ✅ >80% of P1 items checked
- ✅ Core modules applied
- ✅ Tech stack module applied (if available)
</details>

---

## 🎯 Best Practices

### 1. Start Small, Scale Up
```bash
Week 1:  checklists only (2K tokens)
Week 2:  + 2 core modules (6K tokens)
Week 3:  + 1 specialized module (9K tokens)
Week 4:  + tech stack module (12K tokens)
```

### 2. Create Project Templates
```bash
# Save your common combination
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    specialized/multi_tenant_rls.md \
    tech_stacks/nuxt_supabase.md \
    > .code-review-template.md

# Reuse anytime
cat .code-review-template.md | your-review-tool
```

### 3. Track Findings Over Time
```markdown
# FINDINGS.md

## 2025-01-28 - Security Review
**Modules:** core/02, core/03, specialized/ai_security
**Findings:** 0 P0, 2 P1, 3 P2
**Action Items:**
- [ ] Add rate limiting (P1)
- [ ] Implement caching (P1)
- [ ] Add security headers (P2)

## 2025-02-15 - Re-review
**Modules:** Same as above
**Findings:** 0 P0, 0 P1, 1 P2
**Status:** ✅ P1 issues resolved
```

### 4. Customize for Your Team
```bash
# Create team-specific module
cat > team_patterns.md << 'EOF'
# Team-Specific Patterns

## Our Authentication Pattern
All endpoints must use our custom auth:
```typescript
export default defineEventHandler(async (event) => {
  const user = await ourCustomAuth(event)  // ✅ Required
  // ...
})
```
EOF
```

### 5. Integrate with Workflow
```bash
# Add to package.json
{
  "scripts": {
    "review:security": "cat code-review-modules/checklists/security_quick.md",
    "review:performance": "cat code-review-modules/checklists/performance_quick.md",
    "review:full": "cat code-review-modules/core/*.md code-review-modules/specialized/*.md"
  }
}

# Use in CI
npm run review:security | review-tool
```

---

## 📚 Examples in Practice

### Startup SaaS (Nuxt + Supabase)

**Daily PR reviews:**
```bash
cat checklists/security_quick.md
```

**Pre-deployment:**
```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    core/03_input_validation.md \
    specialized/multi_tenant_rls.md \
    tech_stacks/nuxt_supabase.md
```

**Quarterly audit:**
```bash
cat core/*.md \
    specialized/multi_tenant_rls.md \
    specialized/ai_security.md \
    specialized/ssrf_bola.md \
    tech_stacks/nuxt_supabase.md
```

### E-commerce Platform

**Payment feature review:**
```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    core/04_api_security.md \
    specialized/payment_security.md  # Create custom module
```

**Performance optimization:**
```bash
cat core/00_overview.md \
    core/05_performance_basics.md \
    specialized/strategic_caching.md \
    specialized/advanced_indexing.md
```

### AI-Powered Application

**AI feature security:**
```bash
cat core/00_overview.md \
    core/03_input_validation.md \
    specialized/ai_security.md
```

**Cost optimization:**
```bash
cat core/05_performance_basics.md \
    specialized/strategic_caching.md  # Cache AI responses
```

---

## 🔄 Maintenance

### Updating Modules

**Individual module updates:**
```bash
# Update just one module without affecting others
git pull origin main -- specialized/ai_security.md
```

**Project-specific overrides:**
```bash
# Create override without modifying source
cp specialized/multi_tenant_rls.md my_project_rls.md
# Edit my_project_rls.md with project-specific patterns
# Use override instead:
cat core/00_overview.md my_project_rls.md
```

### Adding New Modules

1. **Identify gap in coverage**
   - Review findings: What did you miss?
   - Team feedback: What takes longest to review manually?

2. **Create focused module**
   ```bash
   cp specialized/template.md specialized/my_new_module.md
   ```

3. **Include:**
   - Objective statement
   - Grep patterns for finding issues
   - Code examples (vulnerable + secure)
   - Severity guidelines (P0/P1/P2)
   - Checklist at end

4. **Test it:**
   ```bash
   cat specialized/my_new_module.md | review-tool test-project/
   ```

5. **Share with community** (optional)
   - Submit PR to main repository
   - Help others benefit from your work

---

## 📞 Support & Contributing

### Getting Help

- **Questions?** Open an issue with `[Question]` tag
- **Bug in module?** Open issue with module name in title
- **Need new module?** Open issue with `[Module Request]` tag

### Contributing

**We welcome contributions!**

**How to contribute:**

1. **Fork the repository**
2. **Create your module:**
   - Use `tech_stacks/template.md` as starting point
   - Follow format of existing modules
   - Keep focused and concise
3. **Test it** on real code
4. **Submit PR** with:
   - Description of what it covers
   - Example usage

**Most wanted contributions:**
- [ ] Payment security module (Stripe, PayPal)
- [ ] GraphQL security patterns
- [ ] WebSocket security
- [ ] Mobile API security
- [ ] Blockchain/Web3 patterns
- [ ] Healthcare/HIPAA compliance
- [ ] PCI-DSS compliance
- [ ] More tech stack modules (Django, Rails, Laravel, etc.)

**Module Development Guidelines:**

1. **Keep Focused:** One topic per module
2. **Include Examples:** Show vulnerable + secure code
3. **Use Grep Patterns:** Make it actionable
4. **Provide Context:** Brief explanation of why it matters
5. **Set Severity:** Use P0/P1/P2/P3 consistently
6. **Stay Tech-Agnostic:** (for core modules only)

---

## 🚀 Quick Reference Card

**Print this for your desk:**

```
┌─────────────────────────────────────────────────────┐
│         MODULAR CODE REVIEW QUICK REFERENCE         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  DAILY PR (5 min):                                 │
│  cat checklists/security_quick.md                  │
│                                                     │
│  WEEKLY REVIEW (20 min):                           │
│  cat core/00 core/02 core/03 tech_stacks/[yours]  │
│                                                     │
│  PRE-DEPLOY (30 min):                              │
│  cat core/*.md specialized/[2-3 relevant]          │
│                                                     │
│  COMPREHENSIVE (1 hr):                             │
│  cat core/*.md specialized/*.md tech_stacks/[yours]│
│                                                     │
├─────────────────────────────────────────────────────┤
│  MODULE SELECTION GUIDE:                           │
│  • Multi-tenant? → specialized/multi_tenant_rls    │
│  • AI/LLM? → specialized/ai_security               │
│  • API-heavy? → specialized/ssrf_bola              │
│  • Slow DB? → specialized/advanced_indexing        │
│  • High costs? → specialized/strategic_caching     │
└─────────────────────────────────────────────────────┘
```

---

## 📄 License

MIT License - See [LICENSE](LICENSE) for details.

---

**⭐ Star this repo if it helps you catch security issues!**

**🤝 Contributions welcome** - Help us expand the module library!
