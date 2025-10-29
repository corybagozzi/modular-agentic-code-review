# Contributing to Modular Code Review System

Thank you for your interest in contributing! This repository thrives on community contributions of new modules, improvements, and real-world testing feedback.

---

## 🎯 Ways to Contribute

### 1. Create New Modules

**Most Wanted:**
- Tech stack modules (Django, Rails, Laravel, Flask, Spring Boot, etc.)
- Specialized security modules (Payment processing, Healthcare/HIPAA, GraphQL, WebSocket, Blockchain/Web3)
- Industry-specific compliance modules (GDPR, SOC 2, PCI-DSS)
- Performance modules (Frontend optimization, CDN strategies, Database sharding)

### 2. Improve Existing Modules

- Add more code examples
- Update patterns for new framework versions
- Fix inaccuracies or outdated information
- Improve clarity and readability

### 3. Share Real-World Results

- Submit case studies of vulnerabilities found
- Share token usage metrics
- Report effectiveness in different scenarios

### 4. Report Issues

- Bugs in module logic
- Broken examples
- Missing patterns
- Documentation errors

---

## 📝 Module Contribution Guidelines

### Structure Requirements

Every module must include:

1. **Header with metadata:**
```markdown
# [Module Title]

**Token Estimate:** ~X,XXX tokens
**Purpose:** Brief description
**Applies to:** Technologies/frameworks
```

2. **Usage instructions:**
```markdown
## Usage

```bash
cat core/00_overview.md \
    [this module]
```
```

3. **Clear sections with:**
   - **Vulnerable code examples** (marked with ❌)
   - **Secure code examples** (marked with ✅)
   - **Severity indicators** (P0, P1, P2, P3)
   - **Actionable recommendations**

4. **Checklist at the end:**
```markdown
## Checklist

- [ ] Item 1
- [ ] Item 2
- [ ] Item 3
```

### Code Example Format

Always use this pattern:

```markdown
### X.X [Issue Name]

**Common vulnerability pattern:**

```language
# ❌ VULNERABLE - [Why this is bad]
[bad code example]

# ✅ SECURE - [Why this is good]
[good code example]
```

**Flag [Priority] if:**
- Specific condition 1
- Specific condition 2
```

### Priority Definitions

Use these consistently:

- **P0 (Critical):** Immediate security/data breach risk
- **P1 (Important):** High risk of exploitation or data leak
- **P2 (Recommended):** Defense-in-depth, reduces attack surface
- **P3 (Optional):** Best practices, maintainability

---

## 🔧 Tech Stack Module Guidelines

When creating a framework/language-specific module:

### 1. Focus on Top 5-10 Patterns

Don't try to be exhaustive. Focus on:
- Most common vulnerabilities in that framework
- Framework-specific security features (and how to use them)
- Most dangerous misconfigurations
- Most impactful performance issues

### 2. Required Sections

**Minimum viable tech stack module:**

1. **Authentication/Route Protection**
   - How framework handles auth
   - Common bypass patterns

2. **Query Security**
   - ORM-specific SQL injection risks
   - Parameterization examples

3. **Environment Configuration**
   - How framework loads secrets
   - Common misconfigurations

4. **Performance Patterns**
   - N+1 query detection
   - Framework-specific optimizations

5. **Checklist**
   - Top 10 security items
   - Top 5 performance items

### 3. Keep It Practical

```markdown
# ❌ TOO ABSTRACT
"Ensure proper input validation on all user inputs"

# ✅ SPECIFIC AND ACTIONABLE
**Check:** All controller methods validate params

```ruby
# ❌ VULNERABLE - No strong params
def update
  @user.update(params[:user])
end

# ✅ SECURE - Strong params
def update
  @user.update(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email)
end
```
```

### 4. Test Your Module

Before submitting:

1. **Use it on real code** in that framework
2. **Verify it catches known issues** (create test file with deliberate vulnerabilities)
3. **Check token count** (should be 2-4K tokens)
4. **Get feedback** from someone who uses that framework

---

## 🚀 Submission Process

### Step 1: Fork & Clone

```bash
# Fork repository on GitHub
git clone https://github.com/YOUR_USERNAME/code-review-modules.git
cd code-review-modules
```

### Step 2: Create Module

```bash
# For tech stack module
cp tech_stacks/template.md tech_stacks/your_stack.md

# For specialized module
touch specialized/your_module.md
```

### Step 3: Follow Template

Use existing modules as reference:
- **Tech stacks:** See `tech_stacks/nuxt_supabase.md`
- **Specialized:** See `specialized/multi_tenant_rls.md`
- **Core:** See `core/02_authentication_authorization.md`

### Step 4: Test Locally

```bash
# Test your module
cat core/00_overview.md \
    tech_stacks/your_stack.md \
| wc -w  # Should be ~2K-4K tokens (words * 1.3 ≈ tokens)

# Use on real code
cat core/00_overview.md your_module.md | claude-code review ./sample-project
```

### Step 5: Update README

Add your module to the appropriate table in `README.md`:

```markdown
| **Your Framework** | `tech_stacks/your_stack.md` (~3K tokens) |
```

### Step 6: Submit PR

```bash
git checkout -b add-module-your-stack
git add tech_stacks/your_stack.md README.md
git commit -m "Add tech stack module for [Framework]"
git push origin add-module-your-stack
```

Then create a Pull Request with:

**Title:** `Add [Framework/Topic] module`

**Description:**
```markdown
## Module Details

**Type:** Tech Stack / Specialized / Core
**Framework/Topic:** [Name]
**Token Count:** ~X,XXX tokens
**Tested On:** [Framework version, sample projects]

## What It Covers

- Pattern 1
- Pattern 2
- Pattern 3

## Testing Results

[Optional: Share any vulnerabilities found, token usage, effectiveness metrics]

## Checklist

- [ ] Follows module structure guidelines
- [ ] Includes vulnerable + secure code examples
- [ ] Has clear priority indicators (P0/P1/P2/P3)
- [ ] Tested on real code
- [ ] Token count in 2-4K range
- [ ] Updated README with new module
```

---

## 🧪 Testing Guidelines

### For New Modules

Create a test file with **deliberate vulnerabilities** that your module should catch:

```python
# test_my_module.py

# Your module should flag this as P0
def admin_panel(request):  # ❌ No authentication
    return render(request, 'admin.html')

# Your module should flag this as P1
def search(request):  # ❌ SQL injection
    query = f"SELECT * FROM users WHERE name = '{request.GET['q']}'"
    return User.objects.raw(query)
```

Run your module and verify it catches all test issues.

### For Module Updates

Ensure changes don't:
- Break existing examples
- Reduce coverage
- Significantly increase token count (>20%)

---

## 📏 Code Quality Standards

### 1. Clarity

```markdown
# ❌ UNCLEAR
"Check for proper authentication"

# ✅ CLEAR
**Verify:** All admin routes use `@require_admin` decorator

**Check files:** Glob: `routes/admin/**/*.py`

**Flag P0 if:** Admin route lacks `@require_admin` decorator
```

### 2. Completeness

Every pattern must have:
- **Why it matters** (impact)
- **How to find it** (grep/glob patterns)
- **What to fix** (secure example)
- **When to escalate** (severity criteria)

### 3. Consistency

Use consistent terminology:
- "Flag P0 if..." (not "Mark as critical if...")
- "Vulnerable" vs "Secure" (not "Bad" vs "Good")
- "Check:" vs "Verify:" (pick one per module)

---

## 🎨 Style Guide

### Markdown Formatting

```markdown
### X.X Pattern Name

**Description of issue:**

```language
# ❌ VULNERABLE - Specific reason
[code]

# ✅ SECURE - Specific improvement
[code]
```

**Flag [Priority] if:**
- Condition 1
- Condition 2

**Recommendation:**
[Actionable fix]
```

### Priority Indicators

Always use emoji for visibility:
- ✅ Secure/Good
- ❌ Vulnerable/Bad
- ⚠️ Warning/Caution
- 🔍 Check/Review

### Code Comments

In code examples:
```python
# ✅ SECURE - Bcrypt with work factor 12
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# ❌ VULNERABLE - MD5 is cryptographically broken
password_hash = hashlib.md5(password.encode()).hexdigest()
```

---

## 🤝 Community Guidelines

### Be Respectful

- Constructive feedback on PRs
- Assume good intent
- Help newcomers learn

### Be Accurate

- Test examples before submitting
- Cite sources for security claims
- Update outdated information

### Be Collaborative

- Respond to PR feedback
- Help review other contributions
- Share knowledge openly

---

## 📚 Resources

### Security References

- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **CWE Database:** https://cwe.mitre.org/
- **OWASP Cheat Sheets:** https://cheatsheetseries.owasp.org/

### Framework Security Docs

- **Django:** https://docs.djangoproject.com/en/stable/topics/security/
- **Rails:** https://guides.rubyonrails.org/security.html
- **Express:** https://expressjs.com/en/advanced/best-practice-security.html
- **Spring:** https://spring.io/projects/spring-security

### Module Examples

- **Tech Stack:** See `tech_stacks/nuxt_supabase.md`
- **Specialized:** See `specialized/ai_security.md`
- **Template:** Use `tech_stacks/template.md`

---

## 💬 Getting Help

**Questions about contributing?**

1. **Check existing modules** for examples
2. **Read** [CREATE_TECH_STACK_MODULE.md](./docs/CREATE_TECH_STACK_MODULE.md)
3. **Open an issue** with "Contribution Question" label
4. **Ask in discussions** (if enabled)

**Need help creating a module?**

Open an issue:
```markdown
Title: "Help creating [Framework] module"

I want to create a module for [Framework] but need guidance on:
- [Specific question 1]
- [Specific question 2]
```

We'll help identify key patterns and structure!

---

## 🏆 Recognition

Contributors are recognized in:
- **README.md** - Contributors section
- **Module headers** - "Created by @username"
- **Release notes** - New module announcements

Significant contributions (multiple modules, major improvements) may be recognized as **Module Maintainers** with commit access.

---

## 📝 License

By contributing, you agree that your contributions will be licensed under the same license as this project (MIT License).

---

**Thank you for making code reviews better for everyone!** 🙏
