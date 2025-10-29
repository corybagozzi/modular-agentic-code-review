# Tech Stack Overlay: [Framework Name]

**Token Estimate:** ~2,000 tokens
**Purpose:** Tech-specific patterns for [Framework/Stack description]
**Applies to:** [Framework version, language, deployment platform]

---

## Usage

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    tech_stacks/[your_stack].md
```

---

## 1. [Framework]-Specific Security Patterns

### 1.1 [Security Pattern Category]

```
Find relevant files:
Glob: [pattern]
Grep: [pattern]
```

**Description of what to check:**

```[language]
// ❌ VULNERABLE - [Explanation]
[vulnerable code example]

// ✅ SECURE - [Explanation]
[secure code example]
```

**Flag P0 if:**
- [Critical vulnerability pattern]
- [Another critical issue]

**Flag P1 if:**
- [Important vulnerability pattern]

---

### 1.2 [Another Security Pattern]

[Repeat structure from 1.1]

---

## 2. [Platform/Database]-Specific Patterns

### 2.1 [Pattern Category]

[Repeat structure]

---

## 3. Environment Configuration

### 3.1 Environment Variables

```
Check: [config files]
```

**Proper configuration:**

```[language]
// ❌ DANGEROUS - [Why it's dangerous]
[bad example]

// ✅ SECURE - [Why it's secure]
[good example]
```

**Flag P0 if:**
- Secrets exposed to client
- Production credentials in code

---

## 4. Performance Patterns

### 4.1 [Performance Optimization Category]

**Best practices:**

```[language]
// ❌ SLOW - [Why it's slow]
[slow example]

// ✅ FAST - [Why it's fast]
[optimized example]
```

**ROI:** [Expected performance improvement]

---

## 5. Common [Framework] Issues

### 5.1 [Common Issue Category]

**Problem:**
[Description of common mistake]

**Solution:**
```[language]
[Solution code]
```

**Flag P[0-2] if:**
- [Detection criteria]

---

## 6. Authentication Patterns

### 6.1 [Auth Implementation]

**Recommended pattern:**

```[language]
// ✅ SECURE - [Auth method]
[secure auth implementation]
```

**Flag P0 if:**
- No authentication on protected resources
- [Other auth issues]

---

## 7. [Framework] + [Platform] Checklist

### Framework-Specific:
- [ ] [Security check item]
- [ ] [Security check item]
- [ ] [Configuration check]

### Platform-Specific:
- [ ] [Platform security item]
- [ ] [Platform configuration]

### Performance:
- [ ] [Performance optimization item]
- [ ] [Caching strategy]

---

## Common Patterns Reference

### [Pattern Name]:
```[language]
// Common implementation pattern
[code example]
```

### [Another Pattern]:
```[language]
// Another common pattern
[code example]
```

---

## Instructions for Creating New Tech Stack Overlays

1. **Copy this template** to `tech_stacks/your_framework_name.md`

2. **Replace all [placeholders]** with framework-specific information

3. **Add 5-7 security pattern sections** covering:
   - Route/endpoint protection
   - Environment variable handling
   - Authentication implementation
   - Framework-specific vulnerabilities
   - Deployment platform security

4. **Include code examples** in the framework's language:
   - Always show ❌ vulnerable vs ✅ secure patterns
   - Use real, copy-paste-ready code
   - Add comments explaining why

5. **Add severity flags** (P0/P1/P2) for each pattern

6. **Include a checklist** at the end for quick reference

7. **Estimate tokens** - aim for ~2,000 tokens (1,000-3,000 words)

8. **Link to related modules**:
   - Always reference relevant core modules
   - Link to specialized modules when applicable

---

## Example Tech Stacks to Create

- `django_postgresql.md` - Django + PostgreSQL
- `rails_heroku.md` - Ruby on Rails + Heroku
- `laravel_aws.md` - Laravel + AWS
- `flask_docker.md` - Flask + Docker
- `express_mongo.md` - Express + MongoDB
- `fastapi_postgres.md` - FastAPI + PostgreSQL
- `spring_boot_gcp.md` - Spring Boot + GCP
- `aspnet_azure.md` - ASP.NET + Azure

---

**See Also:**
- `nuxt_supabase.md` - Example implementation
- `nextjs_vercel.md` - Another example
- core/00_overview.md - Review methodology
