# Tech Stack Overlay: Nuxt + Supabase

**Token Estimate:** ~2,000 tokens
**Purpose:** Tech-specific patterns for Nuxt 3/4 + Supabase applications
**Applies to:** Nuxt 3+, Vue 3, Supabase (PostgreSQL + Auth + Storage)

---

## Usage

Combine with core modules for complete review:

```bash
cat core/00_overview.md \
    core/01_reconnaissance.md \
    core/02_authentication_authorization.md \
    tech_stacks/nuxt_supabase.md \
    specialized/multi_tenant_rls.md
```

---

## 1. Nuxt-Specific Security Patterns

### 1.1 Server Route Protection

```
Find all API routes:
Glob: server/api/**/*.{ts,js}
```

**Verify authentication:**

```typescript
// ❌ VULNERABLE - No auth check
export default defineEventHandler(async (event) => {
  const projects = await serverSupabaseClient(event)
    .from('projects')
    .select('*')
  return projects
})

// ✅ SECURE - Requires authentication
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  if (!user) {
    throw createError({
      statusCode: 401,
      message: 'Unauthorized'
    })
  }

  const supabase = await serverSupabaseClient(event)
  const { data } = await supabase
    .from('projects')
    .select('*')

  return data
})
```

**Flag P0 if:**
- Server routes without `serverSupabaseUser()` check
- Sensitive data returned without auth

---

### 1.2 Environment Variable Security

```
Check: nuxt.config.ts, .env
```

**Verify proper exposure:**

```typescript
// nuxt.config.ts

export default defineNuxtConfig({
  runtimeConfig: {
    // ✅ SECURE - Server-only (never exposed to client)
    supabaseServiceRoleKey: process.env.SUPABASE_SERVICE_ROLE_KEY,
    openaiApiKey: process.env.OPENAI_API_KEY,

    public: {
      // ✅ PUBLIC - Safe to expose (anon key)
      supabaseUrl: process.env.NUXT_PUBLIC_SUPABASE_URL,
      supabaseKey: process.env.NUXT_PUBLIC_SUPABASE_KEY,
    }
  }
})
```

**Flag P0 if:**
- Service role key in `public` config
- API keys in client-side code
- Secrets hardcoded in config files

**Check client exposure:**
```typescript
// ❌ WRONG - Using server-only config in client
const config = useRuntimeConfig()
console.log(config.supabaseServiceRoleKey)  // undefined in client ✅

// ✅ CORRECT - Public config only
console.log(config.public.supabaseUrl)  // Works in client ✅
```

---

### 1.3 Middleware Security

```
Find middleware:
Glob: middleware/**/*.{ts,js}
```

**Verify route protection:**

```typescript
// middleware/auth.ts

// ✅ SECURE - Global auth middleware
export default defineNuxtRouteMiddleware(async (to, from) => {
  const user = useSupabaseUser()

  // Public routes
  const publicRoutes = ['/auth/login', '/auth/signup']
  if (publicRoutes.includes(to.path)) {
    return
  }

  // Require auth for all other routes
  if (!user.value) {
    return navigateTo('/auth/login')
  }
})
```

**Admin middleware:**
```typescript
// middleware/admin.ts

export default defineNuxtRouteMiddleware(async (to, from) => {
  const user = useSupabaseUser()

  if (!user.value) {
    return navigateTo('/auth/login')
  }

  // Check admin role (server-side)
  const { data } = await useFetch('/api/admin/check-role')

  if (!data.value?.isAdmin) {
    throw createError({
      statusCode: 403,
      message: 'Admin access required'
    })
  }
})
```

---

## 2. Supabase-Specific Patterns

### 2.1 Row-Level Security Verification

**Critical: All tables MUST have RLS enabled**

```sql
-- Check RLS status
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
  AND rowsecurity = false;
```

**Flag P0 if:**
- Any tenant-scoped table has `rowsecurity = false`
- See `specialized/multi_tenant_rls.md` for detailed audit

---

### 2.2 Service Role Usage

```
Grep patterns:
- "serverSupabaseServiceRole|serviceRole|SUPABASE_SERVICE_ROLE_KEY"
```

**Verify proper usage:**

```typescript
// ❌ DANGEROUS - Service role bypasses RLS without checks
export default defineEventHandler(async (event) => {
  const supabase = serverSupabaseServiceRole(event)
  const { data } = await supabase
    .from('projects')
    .select('*')
    .eq('id', getRouterParam(event, 'id'))
  // No ownership check! ❌
  return data
})

// ✅ SECURE - Manual tenant check even with service role
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  if (!user) throw createError({ statusCode: 401 })

  const supabase = serverSupabaseServiceRole(event)
  const { data } = await supabase
    .from('projects')
    .select('*')
    .eq('id', getRouterParam(event, 'id'))
    .eq('organization_id', user.user_metadata.organization_id)  // Explicit check ✅

  if (!data) throw createError({ statusCode: 404 })
  return data
})
```

**Flag P0 if:**
- Service role used without tenant validation
- Service role in client-side code

---

### 2.3 Supabase Storage Security

```
Check storage configuration:
- Storage buckets in Supabase dashboard
- RLS policies on storage.objects
```

**Storage bucket policies:**

```sql
-- ✅ SECURE - Users can only read own organization's files
CREATE POLICY "Users can view own org files"
ON storage.objects FOR SELECT
USING (
  bucket_id = 'avatars' AND
  (storage.foldername(name))[1] IN (
    SELECT p.id::text
    FROM projects p
    JOIN memberships m ON p.organization_id = m.organization_id
    WHERE m.user_id = auth.uid()
  )
);

-- ✅ SECURE - Only authenticated users can upload
CREATE POLICY "Authenticated users can upload"
ON storage.objects FOR INSERT
WITH CHECK (
  bucket_id = 'avatars' AND
  auth.role() = 'authenticated'
);
```

**Upload validation:**
```typescript
// ✅ SECURE - Server-side upload with validation
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  if (!user) throw createError({ statusCode: 401 })

  const body = await readMultipartFormData(event)
  const file = body?.find(item => item.name === 'file')

  // Validate file
  if (!file || file.data.length > 5 * 1024 * 1024) {
    throw createError({ statusCode: 400, message: 'Invalid file' })
  }

  // Upload with service role (RLS still applies)
  const supabase = serverSupabaseServiceRole(event)
  const path = `${user.id}/${crypto.randomUUID()}`

  const { data, error } = await supabase.storage
    .from('avatars')
    .upload(path, file.data, {
      contentType: file.type,
      upsert: false
    })

  if (error) throw createError({ statusCode: 500 })
  return { path: data.path }
})
```

---

### 2.4 Realtime Subscriptions Security

```
Check for realtime usage:
Grep: "supabase\\.channel|supabase\\.from.*subscribe"
```

**Verify RLS applies to subscriptions:**

```typescript
// ✅ SECURE - RLS filters subscription data
const supabase = useSupabaseClient()

// Only receives updates for user's own organization (via RLS)
supabase
  .channel('jobs')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'jobs'
  }, (payload) => {
    console.log('New job:', payload.new)
  })
  .subscribe()
```

**Note:** RLS policies apply to realtime subscriptions automatically

---

## 3. Nuxt Performance Patterns

### 3.1 Server-Side Data Fetching

```typescript
// ❌ SLOW - Client-side fetch (2 round trips)
const { data } = await useFetch('/api/projects')

// ✅ FASTER - Inlined during SSR
const { data } = await useAsyncData('projects', async () => {
  const supabase = useSupabaseClient()
  const { data } = await supabase.from('projects').select('*')
  return data
})
```

**useFetch vs useAsyncData:**
- `useFetch` - HTTP request (use for external APIs)
- `useAsyncData` - Direct function call (use for Supabase queries)

---

### 3.2 Composable Caching

```typescript
// ✅ CACHED - Shares data across components
export const useProjects = () => {
  return useAsyncData('projects', async () => {
    const supabase = useSupabaseClient()
    const { data } = await supabase
      .from('projects')
      .select('*')
    return data
  }, {
    getCachedData: (key) => useNuxtApp().static.data[key]
  })
}

// Multiple components call useProjects() - only fetches once ✅
```

---

### 3.3 Route Caching

```typescript
// nuxt.config.ts

export default defineNuxtConfig({
  routeRules: {
    // ✅ Cache static pages
    '/docs/**': { swr: 3600 },  // 1 hour stale-while-revalidate

    // ✅ Cache API responses
    '/api/pricing': { swr: 600 },  // 10 minutes

    // ❌ Never cache auth endpoints
    '/api/auth/**': { cache: false },

    // ✅ Prerender public pages
    '/': { prerender: true }
  }
})
```

---

## 4. Common Nuxt + Supabase Issues

### 4.1 Auth State Management

```typescript
// ❌ WRONG - Doesn't handle initial auth state
const user = useSupabaseUser()
console.log(user.value)  // null on first load!

// ✅ CORRECT - Wait for auth to initialize
const user = useSupabaseUser()

onMounted(() => {
  if (!user.value) {
    navigateTo('/auth/login')
  }
})

// Or use middleware (preferred)
definePageMeta({
  middleware: 'auth'
})
```

---

### 4.2 Client vs Server Supabase Instances

```typescript
// CLIENT-SIDE (respects RLS as current user)
const supabase = useSupabaseClient()
const { data } = await supabase.from('projects').select('*')

// SERVER-SIDE (respects RLS as request user)
const supabase = await serverSupabaseClient(event)
const { data } = await supabase.from('projects').select('*')

// SERVER-SIDE (bypasses RLS - use carefully!)
const supabase = serverSupabaseServiceRole(event)
const { data } = await supabase.from('projects').select('*')
```

**Rule:** Never use service role without manual tenant checks

---

### 4.3 Migration Management

```
Check migrations:
Glob: supabase/migrations/*.sql
```

**Verify:**
- ✅ Migrations numbered sequentially
- ✅ All RLS policies in migrations
- ✅ No `IF NOT EXISTS` overuse (hides errors)
- ✅ Indexes for foreign keys

**Testing migrations:**
```bash
# Apply locally first
npx supabase migration up

# Run RLS tests
npm run test:rls

# Verify with prod
npx supabase db diff --linked

# Only then push to production
npx supabase db push
```

---

## 5. Nuxt + Supabase Security Checklist

### Authentication:
- [ ] All server routes use `serverSupabaseUser()` check
- [ ] Middleware protects authenticated routes
- [ ] Admin routes have role verification
- [ ] Service role key never in client config
- [ ] Auth state properly handled on client

### Database:
- [ ] All tables have RLS enabled
- [ ] RLS policies tested (see `specialized/multi_tenant_rls.md`)
- [ ] Service role usage has manual tenant checks
- [ ] Migrations tested locally before production

### Storage:
- [ ] Storage buckets have RLS policies
- [ ] File uploads validated server-side
- [ ] File paths include user/org identifier
- [ ] Storage URLs use signed URLs for private files

### Performance:
- [ ] useAsyncData for Supabase queries (not useFetch)
- [ ] Route caching configured for static pages
- [ ] Composables cache shared data
- [ ] SSR enabled for SEO-critical pages

### Environment:
- [ ] Secrets in server-only runtimeConfig
- [ ] .env in .gitignore
- [ ] Production uses different Supabase project

---

## Common Patterns Reference

### Protected API Route:
```typescript
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  if (!user) throw createError({ statusCode: 401 })

  const supabase = await serverSupabaseClient(event)
  // Query automatically scoped by RLS ✅
  const { data } = await supabase.from('table').select('*')

  return data
})
```

### Admin-Only Route:
```typescript
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  if (!user) throw createError({ statusCode: 401 })

  const supabase = serverSupabaseServiceRole(event)
  const { data: admin } = await supabase
    .from('admin_roles')
    .select('*')
    .eq('user_id', user.id)
    .eq('role', 'super_admin')
    .single()

  if (!admin) throw createError({ statusCode: 403 })

  // Admin logic here
})
```

### Secure File Upload:
```typescript
export default defineEventHandler(async (event) => {
  const user = await serverSupabaseUser(event)
  const form = await readMultipartFormData(event)
  const file = form?.find(f => f.name === 'file')

  // Validate
  if (!file || file.data.length > 5_000_000) {
    throw createError({ statusCode: 400 })
  }

  // Upload with user path
  const supabase = serverSupabaseServiceRole(event)
  const path = `${user.id}/${Date.now()}`
  await supabase.storage.from('bucket').upload(path, file.data)

  return { path }
})
```

---

**See Also:**
- core/02_authentication_authorization.md - General auth patterns
- specialized/multi_tenant_rls.md - Supabase RLS deep dive
- core/05_performance_basics.md - Performance optimization
