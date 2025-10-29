# Creating a Tech Stack Module - Step-by-Step Guide

**Don't see your tech stack?** Follow this 10-minute guide to create one!

---

## Quick Start (5 minutes)

```bash
# 1. Copy the template
cp tech_stacks/template.md tech_stacks/my_stack.md

# 2. Fill in the basics (see below)

# 3. Use it immediately
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    tech_stacks/my_stack.md
```

---

## Step-by-Step Instructions

### Step 1: Copy the Template

```bash
# Navigate to your code-review-modules directory
cd code-review-modules

# Copy template with your stack name (use lowercase, underscores)
cp tech_stacks/template.md tech_stacks/django_postgres.md
# or
cp tech_stacks/template.md tech_stacks/rails_heroku.md
# or
cp tech_stacks/template.md tech_stacks/express_mongo.md
```

### Step 2: Fill in the Header

Open your new file and update the top section:

```markdown
# Tech Stack Overlay: Django + PostgreSQL

**Token Estimate:** ~2,000 tokens
**Purpose:** Django-specific security and performance patterns
**Applies to:** Django 4.x+, Python 3.10+, PostgreSQL 14+

---

## Usage

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    tech_stacks/django_postgres.md
```
```

### Step 3: Add Your Framework's Key Patterns

**Focus on the top 3-5 security issues specific to your framework.**

#### Example: Django

```markdown
## 1. Django-Specific Security Patterns

### 1.1 View Protection

```
Find all views:
Glob: **/views.py
Grep: "def.*request"
```

**Verify authentication:**

```python
# ❌ VULNERABLE - No auth check
def user_profile(request, user_id):
    user = User.objects.get(id=user_id)
    return render(request, 'profile.html', {'user': user})

# ✅ SECURE - Requires login + ownership check
@login_required
def user_profile(request, user_id):
    user = get_object_or_404(User, id=user_id)

    # Check ownership
    if request.user.id != user.id and not request.user.is_staff:
        return HttpResponseForbidden()

    return render(request, 'profile.html', {'user': user})
```

**Flag P0 if:**
- Views without `@login_required` decorator
- No ownership check on user-specific data
```

### Step 4: Add Framework-Specific Vulnerabilities

**Check your framework's security documentation** for common issues:

#### Example: Django

```markdown
### 1.2 SQL Injection in Django ORM

**Common mistake: raw() with string formatting**

```python
# ❌ VULNERABLE - SQL injection via raw()
User.objects.raw(f"SELECT * FROM users WHERE name = '{user_input}'")

# ✅ SECURE - Parameterized raw query
User.objects.raw("SELECT * FROM users WHERE name = %s", [user_input])

# ✅ BETTER - Use ORM methods
User.objects.filter(name=user_input)
```

**Flag P0 if:**
- Using `.raw()` with string formatting
- Using `.extra()` with user input
```

### Step 5: Add Environment Variable Patterns

**How does your framework handle secrets?**

```markdown
## 3. Environment Configuration

### 3.1 Settings.py Security

```
Check: settings.py, .env
```

**Verify proper secret management:**

```python
# ❌ DANGEROUS - Secret in code
SECRET_KEY = 'hard-coded-secret-key-123'
DEBUG = True  # In production!

# ✅ SECURE - From environment
import os
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')
```

**Flag P0 if:**
- `SECRET_KEY` hardcoded
- `DEBUG = True` in production
- No `ALLOWED_HOSTS` restriction
```

### Step 6: Add Performance Patterns

**Framework-specific optimization tips:**

```markdown
## 4. Performance Patterns

### 4.1 Django Query Optimization

**Common N+1 query in Django:**

```python
# ❌ N+1 QUERY
for user in User.objects.all():
    print(user.profile.bio)  # Queries profile table N times

# ✅ OPTIMIZED - select_related for FK
for user in User.objects.select_related('profile'):
    print(user.profile.bio)  # Single JOIN query

# ✅ OPTIMIZED - prefetch_related for M2M
for user in User.objects.prefetch_related('groups'):
    print(user.groups.all())  # 2 queries total
```
```

### Step 7: Add a Checklist

**Make it scannable:**

```markdown
## Django + PostgreSQL Checklist

### Security:
- [ ] All views have `@login_required` or explicit permission checks
- [ ] No `.raw()` or `.extra()` with string formatting
- [ ] `SECRET_KEY` from environment, not hardcoded
- [ ] `DEBUG = False` in production
- [ ] `ALLOWED_HOSTS` configured
- [ ] CSRF protection enabled (default)
- [ ] SQL injection: ORM used properly

### Performance:
- [ ] `select_related()` for foreign keys
- [ ] `prefetch_related()` for M2M and reverse FKs
- [ ] Database indexes on filtered/sorted columns
- [ ] `django-debug-toolbar` shows no N+1 queries

---

**See Also:**
- core/02_authentication_authorization.md
- core/03_input_validation.md
- specialized/multi_tenant_rls.md (if using PostgreSQL)
```

---

## Common Tech Stacks - Quick Templates

### Ruby on Rails + PostgreSQL

**Top patterns to include:**
1. Strong Parameters (mass assignment prevention)
2. `before_action` for authentication
3. SQL injection via `where()` with string interpolation
4. N+1 queries (use `includes()` or `joins()`)
5. CSRF protection via `protect_from_forgery`

**Key files to check:**
```
Glob: app/controllers/**/*.rb, app/models/**/*.rb
```

### Express + MongoDB

**Top patterns to include:**
1. Route authentication middleware
2. NoSQL injection in query objects
3. Helmet.js for security headers
4. JWT validation
5. Rate limiting with `express-rate-limit`

**Key files to check:**
```
Glob: routes/**/*.js, controllers/**/*.js
```

### Laravel + MySQL

**Top patterns to include:**
1. Middleware authentication
2. Eloquent ORM query protection
3. CSRF token validation
4. Mass assignment protection (fillable/guarded)
5. SQL injection via DB::raw()

**Key files to check:**
```
Glob: app/Http/Controllers/**/*.php, routes/**/*.php
```

### Flask + PostgreSQL

**Top patterns to include:**
1. `@login_required` decorator
2. SQL injection via SQLAlchemy
3. CSRF protection (Flask-WTF)
4. Session management
5. Blueprint authentication

**Key files to check:**
```
Glob: app/**/*.py, routes/**/*.py
```

### Spring Boot + PostgreSQL

**Top patterns to include:**
1. `@PreAuthorize` annotations
2. SQL injection via JPQL
3. CSRF protection (Spring Security)
4. Bean validation
5. N+1 queries (use fetch joins)

**Key files to check:**
```
Glob: src/main/java/**/controllers/**/*.java, src/main/java/**/services/**/*.java
```

---

## Quick Reference: What to Include

### Minimum Viable Tech Stack Module

Include at least these 5 sections:

1. **Route/Controller Protection**
   - How to require authentication
   - Framework-specific decorator/middleware patterns

2. **Query Security**
   - ORM-specific SQL injection patterns
   - Parameterization examples

3. **Environment Variables**
   - How framework loads secrets
   - Common misconfigurations

4. **Performance Patterns**
   - N+1 query detection
   - Framework-specific optimization (eager loading, etc.)

5. **Checklist**
   - Top 10 security items
   - Top 5 performance items

### Optional (But Recommended)

6. **CSRF Protection**
   - How framework handles CSRF
   - Common bypass patterns

7. **Session Management**
   - How framework manages sessions
   - Security best practices

8. **File Upload Handling**
   - Framework's file upload mechanism
   - Validation patterns

---

## Testing Your Module

### 1. Use It on Sample Code

```bash
# Clone a sample project in your framework
git clone https://github.com/example/django-sample-app

# Run review with your module
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    tech_stacks/django_postgres.md \
| your-review-tool django-sample-app/
```

### 2. Verify It Finds Known Issues

Create a test file with deliberate vulnerabilities:

```python
# test_vulnerabilities.py

# This should be flagged by your module:

# 1. No authentication
def admin_panel(request):
    return render(request, 'admin.html')

# 2. SQL injection
def search_users(request):
    query = f"SELECT * FROM users WHERE name = '{request.GET['q']}'"
    return User.objects.raw(query)

# 3. Hardcoded secret
SECRET_KEY = 'django-insecure-hardcoded-key'
```

Run your module and verify it flags all three issues.

### 3. Refine Based on Results

- Did it miss any obvious issues? Add patterns.
- Too many false positives? Refine grep patterns.
- Unclear recommendations? Add better examples.

---

## Sharing Your Module

### Option 1: Add to Main Repository

1. **Fork** the main repository
2. **Add your module** to `tech_stacks/`
3. **Update README** with your stack in the table
4. **Submit PR** with:
   - Description of framework/version coverage
   - Example usage
   - Any dependencies or prerequisites

### Option 2: Keep It Private

```bash
# Add to your project's .code-review/ directory
mkdir -p .code-review/tech_stacks
cp tech_stacks/my_stack.md .code-review/tech_stacks/

# Use from project root
cat code-review-modules/core/*.md \
    .code-review/tech_stacks/my_stack.md
```

### Option 3: Publish Separately

Create your own repository:
```
my-code-review-modules/
├── README.md
└── tech_stacks/
    ├── laravel_mysql.md
    ├── wordpress_php.md
    └── symfony_postgres.md
```

Others can use it with:
```bash
cat code-review-modules/core/*.md \
    my-code-review-modules/tech_stacks/laravel_mysql.md
```

---

## Examples of Great Tech Stack Modules

### Minimal But Effective (Django)

```markdown
# Tech Stack Overlay: Django + PostgreSQL

## 1. View Protection
- Check: All views have `@login_required`
- Check: Ownership verified on user-specific data

## 2. Query Security
- Avoid: `.raw()` with string formatting
- Use: ORM methods or parameterized queries

## 3. Settings Security
- `SECRET_KEY` from environment
- `DEBUG = False` in production
- `ALLOWED_HOSTS` configured

## 4. Performance
- Use `select_related()` for FKs
- Use `prefetch_related()` for M2M
- Check for N+1 queries

## Checklist
- [ ] All views protected
- [ ] No SQL injection in raw queries
- [ ] Secrets in environment
- [ ] No N+1 queries
```

**Token count:** ~800 tokens
**Effectiveness:** Covers 80% of Django-specific issues

---

## Need Help?

**Can't figure out what to include?**

1. **Check framework security docs:**
   - Django: https://docs.djangoproject.com/en/stable/topics/security/
   - Rails: https://guides.rubyonrails.org/security.html
   - Express: https://expressjs.com/en/advanced/best-practice-security.html

2. **Look at OWASP guidance** for your framework

3. **Ask the community:**
   - Open an issue: "Help creating [Framework] module"
   - We'll help you identify the top patterns

4. **Start minimal:**
   - Include just 3-5 patterns
   - Expand based on real findings

---

**Remember:** Even a basic 800-token tech stack module is valuable! Start small, iterate based on real usage.
