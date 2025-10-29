# Example Code Reviews

This directory contains real-world examples of code reviews performed using the Modular Code Review System.

## 📄 Available Examples

### [sample-multi-tenant-saas-review.md](./sample-multi-tenant-saas-review.md)

**Type:** Full Security & Performance Audit
**Architecture:** Multi-tenant SaaS application
**Tech Stack:** Nuxt 4 + Supabase
**Modules Used:** 8 modules (~18K tokens)

**What it demonstrates:**
- ✅ Comprehensive multi-tenant RLS security audit
- ✅ Authentication & authorization review
- ✅ AI prompt injection protection analysis
- ✅ API security assessment
- ✅ Performance optimization opportunities
- ✅ SSRF and BOLA vulnerability detection

**Key Findings:**
- 0 P0 (Critical) issues
- 3 P1 (Important) issues found:
  - Missing rate limiting
  - Missing caching layer
  - Webhook URL validation needed
- 2 P2 (Recommended) improvements

**Security Score:** 93/100

**Review Time:** ~30 minutes
**Coverage:** 93% of security areas

---

## 🎯 How to Use These Examples

### For Learning
Study the examples to understand:
- How to apply modules to real codebases
- What findings look like in practice
- How to prioritize security issues (P0/P1/P2)
- How to write actionable recommendations

### For Comparison
Use these examples to:
- Benchmark your own review findings
- Understand expected coverage levels
- See how different modules complement each other
- Validate your review methodology

### For LinkedIn/Blog Posts
These anonymized examples can be:
- Shared in professional posts (already anonymized)
- Used in case studies
- Referenced in security discussions
- Cited as methodology examples

---

## 🔒 Note on Anonymization

All examples in this directory have been **anonymized** to protect proprietary business logic:
- Specific feature names generalized (e.g., "content generation" vs specific types)
- Migration timestamps removed
- Service provider names genericized where appropriate
- Internal file paths kept generic

The **security patterns and findings remain accurate** and representative of real-world audits.

---

## 🤝 Contributing Your Own Examples

Have a great code review example you'd like to share?

1. **Anonymize** your review output (remove proprietary details)
2. **Add context** at the top (tech stack, modules used, findings summary)
3. **Submit a PR** with your example

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

**Most wanted examples:**
- Django + PostgreSQL reviews
- Ruby on Rails applications
- Express + MongoDB stacks
- Laravel PHP applications
- Flask + PostgreSQL
- Spring Boot + Java
- GraphQL API security reviews
- WebSocket application reviews
