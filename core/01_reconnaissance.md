# Core Module: Reconnaissance & Architecture Mapping

**Token Estimate:** ~2,000 tokens
**Purpose:** Understand codebase structure before deep review
**Dependencies:** core/00_overview.md

---

## Objective

Establish complete understanding of the codebase architecture, tech stack, and critical file locations before conducting deep security or performance reviews.

---

## Step 1: Initialize Review Tracking

Use TodoWrite to create review phase tracking (copy from core/00_overview.md).

---

## Step 2: Identify Tech Stack

```
Read: package.json, requirements.txt, Gemfile, go.mod (as applicable)
Read: Configuration files (nuxt.config.ts, next.config.js, etc.)
```

**Document:**
- **Frontend Framework:** [React / Vue / Angular / Svelte / etc.]
- **Backend Framework:** [Express / Fastify / Django / Rails / etc.]
- **Database:** [PostgreSQL / MySQL / MongoDB / etc.]
- **ORM/Query Builder:** [Prisma / Sequelize / Drizzle / TypeORM / etc.]
- **Authentication:** [Custom / Auth0 / Supabase Auth / NextAuth / etc.]
- **Hosting:** [Vercel / AWS / GCP / Azure / Docker / etc.]
- **Key Libraries:** [List critical dependencies]

---

## Step 3: Map Directory Structure

```
Use Task tool with Explore agent (thoroughness: "medium"):
"Explore the codebase to map the full directory structure. Identify all source code files,
database migrations, test files, and configuration files. Provide a summary of the project
organization, key directories, and file counts."
```

**Expected Output:**
```
project/
├── src/app/           # Application code (X files)
├── src/api/           # API routes (Y files)
├── src/lib/           # Utilities (Z files)
├── database/          # Migrations/schemas (N files)
├── tests/             # Test files (M files)
├── config/            # Configuration
└── public/            # Static assets
```

---

## Step 4: Critical File Inventory

Use Glob to catalog critical files by category:

### 4.1 Database Schemas & Migrations

```
Glob patterns:
- database/migrations/*.sql
- prisma/schema.prisma
- db/migrate/*.rb
- migrations/*.py
- *.sql
```

**Check:**
- Total migration count
- Recent changes (last 30 days)
- Table definitions
- Index definitions

### 4.2 API Routes by Security Zone

```
Glob patterns by framework:

Next.js:
- app/api/**/*.ts
- pages/api/**/*.ts

Nuxt:
- server/api/**/*.ts

Express:
- routes/**/*.ts
- api/**/*.ts

Django:
- */views.py
- */urls.py
```

**Categorize endpoints:**
- **Public API:** Requires API key or open
- **Internal API:** Requires user authentication
- **Admin API:** Requires elevated privileges
- **Webhooks:** Requires signature verification

### 4.3 Authentication & Authorization

```
Grep patterns:
- "auth|login|signup|session"
- "middleware.*auth"
- "requireAuth|requireAdmin|checkPermission"
```

**Find:**
- Authentication implementation files
- Session management
- Token validation
- Permission checking utilities

### 4.4 Security-Critical Utilities

```
Look for utility files containing:
- API key validation
- Token management
- Password hashing
- Encryption/decryption
- Rate limiting
- Input sanitization
```

### 4.5 Configuration Files

```
Glob:
- .env.example
- config/*.{ts,js,json}
- *.config.{ts,js}

Verify .env NOT in repository:
- Check .gitignore includes .env
```

### 4.6 Test Files

```
Glob patterns:
- **/*.test.{ts,js,py,rb}
- **/*.spec.{ts,js}
- tests/**/*
- spec/**/*
```

**Measure:**
- Test file count
- Coverage of security-critical code
- Integration vs unit test ratio

---

## Step 5: Identify Critical Data Flows

Use Grep to search for critical patterns:

### 5.1 Database Access Patterns

```
Grep patterns:
- Database queries: "\.query\(|\.from\(|\.select\(|\.findMany"
- Raw SQL: "raw.*sql|query.*\$\{|sql\`"
- ORM operations: "\.create\(|\.update\(|\.delete\("
```

### 5.2 Authentication Checkpoints

```
Grep patterns:
- Auth checks: "auth\.user|req\.user|session\."
- Permission checks: "can\(|ability\.|authorize"
- Role checks: "role.*==|hasRole|checkRole"
```

### 5.3 External API Calls

```
Grep patterns:
- HTTP clients: "fetch\(|axios\.|http\.get|request\("
- Webhooks: "webhook|callback|notify"
```

### 5.4 Environment Variables

```
Grep pattern: "process\.env|os\.getenv|ENV\["

Categorize:
- Secrets (API keys, database URLs)
- Configuration (URLs, feature flags)
- Public variables
```

---

## Step 6: Architecture Assessment

Document your findings:

```markdown
## Architecture Summary

### Tech Stack
- Frontend: [Framework + Version]
- Backend: [Framework + Version]
- Database: [Type + Version]
- Auth: [Method]
- Hosting: [Platform]

### Project Structure
- Monorepo: [Yes/No]
- Microservices: [Yes/No]
- API Style: [REST / GraphQL / gRPC]
- File Count: ~[X] files

### Security Model
- Authentication: [Method description]
- Authorization: [RBAC / ABAC / Custom]
- Multi-tenancy: [Yes/No - If yes, how isolated?]
- Secrets Management: [Env vars / Vault / etc.]

### Database
- Tables: [Count]
- Migrations: [Count]
- Row-Level Security: [Enabled/Disabled]
- Indexes: [Well-indexed / Needs work]

### API Surface
- Public Endpoints: [Count]
- Internal Endpoints: [Count]
- Admin Endpoints: [Count]
- Webhooks: [Count]

### Test Coverage
- Test Files: [Count]
- Framework: [Jest / Vitest / Pytest / etc.]
- Coverage: [% if available]

### Critical Files Identified
- Auth utilities: [List]
- API key validation: [List]
- Database models: [List]
- Security middleware: [List]
```

---

## Step 7: Risk Profile Assessment

Based on reconnaissance, assess initial risk areas:

**High Risk Areas (Investigate Further):**
- [ ] Multi-tenant system without RLS
- [ ] Custom authentication implementation
- [ ] Many admin endpoints without clear protection
- [ ] External API calls without input validation
- [ ] No test coverage for security features
- [ ] Raw SQL queries with string concatenation
- [ ] Secrets in configuration files

**Medium Risk Areas:**
- [ ] Complex authorization logic
- [ ] File upload functionality
- [ ] Payment processing
- [ ] Third-party integrations
- [ ] Cron jobs / background workers

**Questions to Answer:**
1. Is this a multi-tenant application?
2. What is the primary data security concern?
3. Are there any compliance requirements (GDPR, HIPAA, PCI)?
4. What is the most sensitive data stored?
5. What external services are integrated?

---

## Step 8: Select Review Modules

Based on reconnaissance, choose appropriate modules:

**For Multi-Tenant SaaS:**
```bash
cat core/02_authentication_authorization.md \
    specialized/multi_tenant_rls.md
```

**For API-Heavy Applications:**
```bash
cat core/03_input_validation.md \
    core/04_api_security.md \
    specialized/ssrf_bola.md
```

**For AI/ML Applications:**
```bash
cat core/03_input_validation.md \
    specialized/ai_security.md
```

**For Performance Reviews:**
```bash
cat core/05_performance_basics.md \
    specialized/strategic_caching.md \
    specialized/advanced_indexing.md
```

---

## Output Template

```markdown
# Reconnaissance Report

**Date:** [YYYY-MM-DD]
**Reviewer:** [Name]
**Repository:** [Name]

## Tech Stack
[From Step 6]

## File Inventory
- Source files: [count]
- API endpoints: [count]
- Database migrations: [count]
- Test files: [count]

## Architecture Type
[Monolith / Microservices / Serverless / etc.]

## Security Model
[Description from Step 6]

## Risk Assessment
**Overall Risk Level:** [Critical / High / Medium / Low]
**High Priority Areas:** [List]

## Recommended Review Modules
1. [Module name] - [Reason]
2. [Module name] - [Reason]
3. [Module name] - [Reason]

## Next Steps
1. Proceed with [selected modules]
2. Focus on [high-risk areas]
3. Track findings using TodoWrite
```

---

**Next Steps:**
- Update TodoWrite: Mark Phase 1 complete
- Proceed to selected security/performance modules
- Begin deep scan based on identified risks
