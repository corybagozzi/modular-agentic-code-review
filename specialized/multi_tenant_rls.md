# Specialized Module: Multi-Tenant RLS Security

**Token Estimate:** ~3,000 tokens
**Purpose:** Deep audit of Row-Level Security for multi-tenant applications
**Dependencies:** core/02_authentication_authorization.md
**Best For:** PostgreSQL/Supabase multi-tenant SaaS applications

---

## Objective

Ensure complete tenant isolation through Row-Level Security (RLS) policies, preventing cross-tenant data leakage in multi-tenant applications.

---

## 1. RLS Policy Inventory

### 1.1 Identify All Tenant-Scoped Tables

```
Find migration files:
Glob: database/migrations/*.sql, supabase/migrations/*.sql
Grep: "CREATE TABLE|CREATE POLICY|ENABLE ROW LEVEL SECURITY"
```

**For each table, document:**
- Table name
- Tenant identifier column (organization_id, workspace_id, etc.)
- RLS enabled? (Y/N)
- Policy count (SELECT, INSERT, UPDATE, DELETE)
- Policy uses auth context? (Y/N)

**Flag P0 if:**
- Tenant-scoped table WITHOUT RLS enabled
- Table has user/org data but no isolation

---

## 2. RLS Policy Coverage Matrix

### 2.1 Required Policies Per Table

Every tenant-scoped table MUST have 4 policies:

```sql
-- ✅ COMPLETE COVERAGE
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Policy 1: SELECT
CREATE POLICY "Users can view own organization projects"
ON projects FOR SELECT
USING (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid()
  )
);

-- Policy 2: INSERT
CREATE POLICY "Users can create projects in own organization"
ON projects FOR INSERT
WITH CHECK (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid()
  )
);

-- Policy 3: UPDATE
CREATE POLICY "Users can update own organization projects"
ON projects FOR UPDATE
USING (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid()
  )
);

-- Policy 4: DELETE
CREATE POLICY "Users can delete own organization projects"
ON projects FOR DELETE
USING (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid() AND role = 'owner'
  )
);
```

**Flag P0 if:**
- Missing any of the 4 policy types
- Policy exists but uses `true` or `1=1` (effectively disabled)

---

## 3. Auth Context Validation

### 3.1 Verify Proper Auth Functions

**PostgreSQL/Supabase:**
```sql
-- ✅ CORRECT - Uses auth.uid()
USING (user_id = auth.uid())

-- ✅ CORRECT - Uses JWT claims
USING (organization_id = (current_setting('request.jwt.claims', true)::json->>'organization_id'))

-- ❌ WRONG - Uses session variable (can be spoofed)
USING (organization_id = current_setting('app.current_org_id'))

-- ❌ WRONG - No auth check
USING (true)
```

**Flag P0 if:**
- Policy uses spoofable session variables
- No authentication context in policy
- Policy trusts client-provided values

---

## 4. Tenant Hierarchy Verification

### 4.1 Organization → Project → Resource Chain

For nested tenant structures, verify EACH level:

```sql
-- ❌ INCOMPLETE - Only checks direct parent
CREATE POLICY "View jobs" ON jobs FOR SELECT
USING (project_id IN (SELECT id FROM projects WHERE user_id = auth.uid()));

-- ✅ COMPLETE - Traverses full hierarchy
CREATE POLICY "View jobs" ON jobs FOR SELECT
USING (
  project_id IN (
    SELECT p.id FROM projects p
    JOIN memberships m ON p.organization_id = m.organization_id
    WHERE m.user_id = auth.uid()
  )
);
```

**Verify hierarchy:**
- User → Membership → Organization → Project → Job
- Each level enforces tenant isolation
- No shortcuts that bypass levels

---

## 5. Service Role Bypass Detection

### 5.1 Check for Unsafe Service Role Usage

```
Grep patterns:
- "service.*role|service_role_key|SUPABASE_SERVICE_ROLE_KEY"
- "role.*=.*service|bypass.*rls"
```

**Verify:**
- ✅ Service role used ONLY in backend code
- ✅ Never exposed to frontend
- ✅ Backend code still validates tenant context manually
- ✅ Admin operations log audit trail

**Example:**
```typescript
// ❌ DANGEROUS - Service role bypasses RLS with no checks
const { data } = await supabaseAdmin
  .from('projects')
  .select('*')
  .eq('id', req.params.id) // No org check!

// ✅ SECURE - Manual tenant check even with service role
const user = await supabaseAdmin.auth.getUser(token)
const { data } = await supabaseAdmin
  .from('projects')
  .select('*')
  .eq('id', req.params.id)
  .eq('organization_id', user.organization_id) // Explicit check
```

**Flag P0 if:**
- Service role key in frontend code
- Service role queries without tenant validation
- No audit logging for service role operations

---

## 6. Cross-Tenant Join Prevention

### 6.1 Verify Joins Respect RLS

```sql
-- ❌ VULNERABLE - Join might leak cross-tenant data
SELECT p.*, j.*
FROM projects p
LEFT JOIN jobs j ON j.project_id = p.id
-- If 'jobs' table has weak RLS, can see other orgs' jobs

-- ✅ SECURE - Both tables have proper RLS
SELECT p.*, j.*
FROM projects p
LEFT JOIN jobs j ON j.project_id = p.id
-- Both tables filter by organization_id via RLS
```

**Test:**
- Create test user in Org A
- Create test user in Org B
- Verify user A cannot query user B's data via any join

---

## 7. Policy Performance

### 7.1 Ensure Policies Don't Create N+1 Issues

**Check for subqueries in policies:**
```sql
-- ❌ SLOW - Subquery executes per row
CREATE POLICY "View projects" ON projects FOR SELECT
USING (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid()
  )
);

-- ✅ FASTER - Use function or indexed column
CREATE POLICY "View projects" ON projects FOR SELECT
USING (organization_id = get_user_org_id(auth.uid()));

-- Even better: Index on (organization_id, user_id) in memberships
```

**Verify:**
- ✅ Policies use indexed columns
- ✅ Complex policies use functions (cached)
- ✅ No Cartesian products in policy joins

---

## 8. Default Deny Verification

### 8.1 Test Unauthenticated Access

**For each tenant-scoped table:**
```sql
-- Test as unauthenticated user
SET LOCAL role = 'anon';
SELECT * FROM projects; -- Should return 0 rows

-- Test as wrong user
SET LOCAL role = 'authenticated';
SET LOCAL request.jwt.claims = '{"sub": "wrong-user-id"}';
SELECT * FROM projects WHERE organization_id = 'target-org';
-- Should return 0 rows
```

**Flag P0 if:**
- Unauthenticated users can read any tenant data
- Default is allow instead of deny

---

## 9. Shared/Public Data Handling

### 9.1 Tables Without Tenant Isolation

**Identify truly shared tables:**
- Configuration tables (action_costs, pricing_tiers)
- Public data (blog posts, documentation)
- System tables (migrations, internal state)

**Verify:**
```sql
-- ✅ CORRECT - Public data allows SELECT only
CREATE POLICY "Anyone can view pricing" ON pricing_tiers FOR SELECT
USING (true);

-- No INSERT/UPDATE/DELETE policies for public users

-- ✅ CORRECT - System tables deny all
CREATE POLICY "Deny all access" ON internal_jobs FOR ALL
USING (false);
-- Only service role can access
```

---

## 10. Membership/Role RLS

### 10.1 User-Organization Relationship

**Critical table: `memberships` or `user_organizations`**

```sql
-- ✅ SECURE - Users see only their own memberships
CREATE POLICY "View own memberships" ON memberships FOR SELECT
USING (user_id = auth.uid());

-- ✅ SECURE - Only org owners can add members
CREATE POLICY "Owners can add members" ON memberships FOR INSERT
WITH CHECK (
  organization_id IN (
    SELECT organization_id FROM memberships
    WHERE user_id = auth.uid() AND role = 'owner'
  )
);

-- ❌ VULNERABLE - Users can add themselves to any org
CREATE POLICY "Anyone can join" ON memberships FOR INSERT
WITH CHECK (user_id = auth.uid());
```

**Flag P0 if:**
- Users can insert themselves into any organization
- Users can modify their own role
- No owner requirement for member management

---

## 11. Soft Delete RLS

### 11.1 Ensure Deleted Records Respect RLS

```sql
-- ❌ INCOMPLETE - Deleted records visible
CREATE POLICY "View projects" ON projects FOR SELECT
USING (organization_id = get_user_org_id());

-- ✅ COMPLETE - Excludes soft-deleted
CREATE POLICY "View active projects" ON projects FOR SELECT
USING (
  organization_id = get_user_org_id() AND
  deleted_at IS NULL
);
```

---

## 12. Audit Logging for RLS

### 12.1 Track Policy Violations

**Create logging trigger:**
```sql
CREATE OR REPLACE FUNCTION log_rls_violation()
RETURNS TRIGGER AS $$
BEGIN
  IF NOT (NEW.organization_id = get_user_org_id()) THEN
    INSERT INTO security_audit_log (
      user_id,
      action,
      table_name,
      attempted_org_id,
      actual_org_id,
      timestamp
    ) VALUES (
      auth.uid(),
      TG_OP,
      TG_TABLE_NAME,
      NEW.organization_id,
      get_user_org_id(),
      NOW()
    );
    RAISE EXCEPTION 'Cross-tenant access attempt blocked';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_tenant_isolation
  BEFORE INSERT OR UPDATE ON projects
  FOR EACH ROW EXECUTE FUNCTION log_rls_violation();
```

---

## 13. RLS Testing Requirements

### 13.1 Automated Test Coverage

**Must have tests for EVERY table:**

```typescript
// Example test structure
describe('RLS: projects table', () => {
  test('User A cannot view User B projects', async () => {
    const projectB = await createProject({ orgId: 'org-b' })
    const { data } = await supabaseAsUserA
      .from('projects')
      .select('*')
      .eq('id', projectB.id)
    expect(data).toHaveLength(0)
  })

  test('User cannot insert into other org', async () => {
    await expect(
      supabaseAsUserA.from('projects').insert({
        name: 'Test',
        organization_id: 'org-b' // Different org
      })
    ).rejects.toThrow()
  })

  test('User cannot update other org resources', async () => {
    const projectB = await createProject({ orgId: 'org-b' })
    await expect(
      supabaseAsUserA.from('projects')
        .update({ name: 'Hacked' })
        .eq('id', projectB.id)
    ).rejects.toThrow()
  })

  test('User cannot delete other org resources', async () => {
    const projectB = await createProject({ orgId: 'org-b' })
    await expect(
      supabaseAsUserA.from('projects')
        .delete()
        .eq('id', projectB.id)
    ).rejects.toThrow()
  })
})
```

**Coverage target: 100% of tenant-scoped tables**

---

## 14. Common RLS Anti-Patterns

### 14.1 Dangerous Patterns to Flag

```sql
-- ❌ ANTI-PATTERN 1: Using SECURITY DEFINER functions
CREATE FUNCTION get_all_projects()
RETURNS SETOF projects
SECURITY DEFINER -- Bypasses RLS!
AS $$ SELECT * FROM projects $$;

-- ❌ ANTI-PATTERN 2: Permissive OR conditions
CREATE POLICY "View projects" ON projects FOR SELECT
USING (
  organization_id = get_user_org_id() OR
  is_public = true -- Can leak data if misconfigured
);

-- ❌ ANTI-PATTERN 3: Client-side filtering
-- Relying on WHERE clauses instead of RLS
const { data } = await supabase
  .from('projects')
  .select('*')
  .eq('organization_id', userOrgId) // RLS should enforce this!

-- ❌ ANTI-PATTERN 4: No policy = No access (but unclear)
-- Better to have explicit DENY policy
```

---

## 15. Migration Checklist

**Before deploying RLS changes:**

- [ ] All tenant-scoped tables identified
- [ ] RLS enabled on all tenant tables
- [ ] 4 policies (SELECT/INSERT/UPDATE/DELETE) per table
- [ ] Policies use auth.uid() or equivalent
- [ ] Service role usage audited
- [ ] Tests written for cross-tenant access prevention
- [ ] Tests passing (100% isolation verified)
- [ ] Performance impact measured
- [ ] Rollback plan ready
- [ ] Audit logging enabled

---

## 16. Emergency RLS Audit

**If breach suspected, run immediately:**

```sql
-- Find tables WITHOUT RLS
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename NOT IN (
    SELECT tablename FROM pg_policies
  )
  AND tablename NOT LIKE 'pg_%'
  AND tablename NOT LIKE 'sql_%';

-- Find policies with "true" (always allow)
SELECT schemaname, tablename, policyname, qual
FROM pg_policies
WHERE qual = 'true';

-- Find tables with RLS disabled
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND rowsecurity = false
  AND tablename NOT LIKE 'pg_%';
```

---

## Multi-Tenant RLS Checklist

- [ ] All tenant tables have RLS enabled
- [ ] All tables have 4 policies (SELECT/INSERT/UPDATE/DELETE)
- [ ] Policies use auth.uid() or secure JWT claims
- [ ] Service role usage is audited and logged
- [ ] Cross-tenant joins tested
- [ ] Membership table prevents self-promotion
- [ ] Soft deletes respect RLS
- [ ] 100% test coverage on RLS policies
- [ ] No SECURITY DEFINER functions bypass RLS
- [ ] Performance impact acceptable

---

**Severity Summary:**
- **P0:** Missing RLS on tenant table, cross-tenant access possible, service role bypass
- **P1:** Incomplete policy coverage, weak auth context, no testing
- **P2:** Performance issues, missing audit logs

**See Also:**
- core/02_authentication_authorization.md - RBAC and access control
- specialized/ssrf_bola.md - BOLA/IDOR prevention
