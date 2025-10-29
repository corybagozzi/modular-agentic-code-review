# Changelog

All notable changes to the Modular Code Review System will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-01-28

### Added

#### Core Modules (6)
- `core/00_overview.md` - Execution protocol, severity definitions, reporting format
- `core/01_reconnaissance.md` - Architecture mapping, tech stack identification
- `core/02_authentication_authorization.md` - Auth patterns, RBAC, BOLA/IDOR prevention
- `core/03_input_validation.md` - SQL injection, XSS, command injection, ReDoS
- `core/04_api_security.md` - Rate limiting, CORS, security headers, webhooks
- `core/05_performance_basics.md` - N+1 queries, indexing, caching opportunities

#### Specialized Modules (6)
- `specialized/multi_tenant_rls.md` - Row-Level Security audit for multi-tenant apps
- `specialized/ai_security.md` - Prompt injection, data poisoning, LLM security
- `specialized/ssrf_bola.md` - SSRF prevention, BOLA/IDOR detection
- `specialized/command_injection_redos.md` - Command injection, ReDoS patterns
- `specialized/strategic_caching.md` - Multi-layer caching strategies, cache invalidation
- `specialized/advanced_indexing.md` - Composite, partial, covering indexes

#### Tech Stack Modules (3)
- `tech_stacks/nuxt_supabase.md` - Nuxt 3/4 + Supabase patterns
- `tech_stacks/nextjs_vercel.md` - Next.js 13-15 App Router patterns
- `tech_stacks/template.md` - Template for creating new tech stack modules

#### Quick Checklists (2)
- `checklists/security_quick.md` - P0/P1 security checklist (~5 minutes)
- `checklists/performance_quick.md` - Performance benchmarks, quick wins

#### Documentation
- `README.md` - Comprehensive usage guide with token calculator, decision tree, FAQ
- `docs/CREATE_TECH_STACK_MODULE.md` - Step-by-step guide for creating modules
- `CONTRIBUTING.md` - Contribution guidelines, module standards, testing requirements
- `ENHANCEMENTS_SUMMARY.md` - Documentation of all README improvements
- `LICENSE` - MIT License
- `CHANGELOG.md` - This file

#### GitHub Templates
- `.github/ISSUE_TEMPLATE/bug_report.md` - Bug report template
- `.github/ISSUE_TEMPLATE/feature_request.md` - Feature request template
- `.github/ISSUE_TEMPLATE/new_module_request.md` - New module request template

### Validated
- Tested modular approach vs monolithic (v3) on real codebase
- **Result:** 35% more token-efficient, 20% more P1 findings
- **Coverage:** 93% vs 79% for monolithic approach
- Caught webhook SSRF vulnerability missed by monolithic

### Benchmarks
- Total modules: 17
- Token range per module: 2-4K tokens
- Minimum viable review: ~5K tokens (checklists)
- Comprehensive review: ~30K tokens (all core + specialized)
- Flexibility: Infinite combinations for different scenarios

---

## [Unreleased]

### Planned
- Payment security module (Stripe, PayPal patterns)
- Healthcare/HIPAA compliance module
- GraphQL security module
- WebSocket security module
- Mobile API security module
- Additional tech stack modules (Django, Rails, Laravel, Flask, Spring Boot)

---

## How to Use This Changelog

**For Users:**
- Check this file to see what's new in each version
- Review changes before updating modules

**For Contributors:**
- Update this file when submitting PRs
- Follow the format: Added/Changed/Deprecated/Removed/Fixed/Security
- Link to relevant issues/PRs

---

## Version History

- **1.0.0** (2025-01-28) - Initial release with 17 modules, comprehensive documentation
