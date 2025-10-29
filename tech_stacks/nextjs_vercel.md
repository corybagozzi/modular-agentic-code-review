# Tech Stack Overlay: Next.js + Vercel

**Token Estimate:** ~2,000 tokens
**Purpose:** Tech-specific patterns for Next.js 13-15 (App Router) + Vercel
**Applies to:** Next.js 13+, React, Vercel deployment, Edge Runtime

---

## Usage

```bash
cat core/00_overview.md \
    core/02_authentication_authorization.md \
    tech_stacks/nextjs_vercel.md \
    specialized/ssrf_bola.md
```

---

## 1. Next.js App Router Security

### 1.1 Server Actions Protection

```
Find Server Actions:
Grep: "use server|'use server'"
```

**Verify authentication:**

```typescript
// app/actions/projects.ts

'use server'

// ❌ VULNERABLE - No auth check
export async function createProject(formData: FormData) {
  const name = formData.get('name')
  await db.projects.create({ data: { name } })
}

// ✅ SECURE - Auth verification
import { auth } from '@/lib/auth'

export async function createProject(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  const name = formData.get('name') as string

  // Validate input
  if (!name || name.length > 100) {
    throw new Error('Invalid input')
  }

  await db.projects.create({
    data: {
      name,
      user_id: session.user.id
    }
  })

  revalidatePath('/projects')
}
```

**Flag P0 if:**
- Server Actions without auth checks
- Server Actions trust client-provided user IDs
- No input validation in Server Actions

---

### 1.2 Route Handler Security

```
Find Route Handlers:
Glob: app/api/**/route.{ts,tsx}
```

**Protected API routes:**

```typescript
// app/api/projects/route.ts

import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'

// ❌ VULNERABLE
export async function GET(request: NextRequest) {
  const projects = await db.projects.findMany()
  return NextResponse.json(projects)
}

// ✅ SECURE
export async function GET(request: NextRequest) {
  const session = await getServerSession(authOptions)

  if (!session) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }

  const projects = await db.projects.findMany({
    where: { user_id: session.user.id }
  })

  return NextResponse.json(projects)
}
```

---

### 1.3 Environment Variables

```
Check: .env.local, next.config.js
```

**Proper exposure:**

```typescript
// next.config.js

module.exports = {
  env: {
    // ❌ DANGEROUS - Exposes secret to client
    DATABASE_URL: process.env.DATABASE_URL,  // NEVER!
  }
}

// ✅ SECURE - Server-only variables
// No config needed - accessed via process.env in server components/API routes

// Client-side requires NEXT_PUBLIC_ prefix
const publicUrl = process.env.NEXT_PUBLIC_API_URL  // OK in client
const secret = process.env.DATABASE_URL  // undefined in client ✅
```

**Flag P0 if:**
- Secrets in `env` config option
- API keys without `NEXT_PUBLIC_` prefix in client code
- `.env.local` committed to git

---

## 2. Middleware Security

### 2.1 Authentication Middleware

```
Check: middleware.ts (root level)
```

```typescript
// middleware.ts

import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt'

export async function middleware(request: NextRequest) {
  const token = await getToken({ req: request })

  // Public paths
  const publicPaths = ['/login', '/signup', '/']
  if (publicPaths.includes(request.nextUrl.pathname)) {
    return NextResponse.next()
  }

  // Require auth for all other paths
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Admin paths
  if (request.nextUrl.pathname.startsWith('/admin')) {
    if (token.role !== 'admin') {
      return NextResponse.redirect(new URL('/', request.url))
    }
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)']
}
```

---

### 2.2 Rate Limiting Middleware

```typescript
// middleware.ts

import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'),
})

export async function middleware(request: NextRequest) {
  // Rate limit API routes
  if (request.nextUrl.pathname.startsWith('/api')) {
    const ip = request.ip ?? '127.0.0.1'
    const { success } = await ratelimit.limit(ip)

    if (!success) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429 }
      )
    }
  }

  return NextResponse.next()
}
```

---

## 3. Vercel-Specific Security

### 3.1 Edge Runtime Limitations

```
Check for Edge Runtime usage:
Grep: "export const runtime = 'edge'"
```

**Limitations to verify:**
- ✅ No file system access (`fs` module)
- ✅ No Node.js-specific APIs
- ✅ Limited npm packages
- ✅ No dynamic code evaluation

```typescript
// app/api/route.ts

// ❌ FAILS at edge runtime
export const runtime = 'edge'

export async function GET() {
  const fs = require('fs')  // Error: fs not available
  const data = fs.readFileSync('data.json')
  return Response.json(data)
}

// ✅ WORKS at edge
export const runtime = 'edge'

export async function GET() {
  const data = await fetch('https://api.example.com/data')
  return Response.json(await data.json())
}
```

**Flag P1 if:**
- Edge functions using Node.js APIs
- No fallback for unsupported packages

---

### 3.2 Environment Variable Security

**Vercel dashboard:**
- ✅ Secrets stored in Vercel dashboard (not .env.local)
- ✅ Different values for preview vs production
- ✅ Sensitive vars scoped to production only

**Flag P0 if:**
- Production secrets in .env files in git
- Same credentials for preview and production

---

### 3.3 Vercel Cron Jobs

```
Check: vercel.json
```

**Secure cron endpoints:**

```json
{
  "crons": [{
    "path": "/api/cron/cleanup",
    "schedule": "0 0 * * *"
  }]
}
```

```typescript
// app/api/cron/cleanup/route.ts

import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  // ✅ SECURE - Verify cron secret
  const authHeader = request.headers.get('authorization')

  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }

  // Cleanup logic
  await cleanupOldJobs()

  return NextResponse.json({ success: true })
}
```

**Flag P1 if:**
- Cron endpoints without authorization
- No secret verification

---

## 4. Next.js Performance Patterns

### 4.1 Server Components vs Client Components

```
Grep: "use client|'use client'"
```

**Default to Server Components:**

```tsx
// ✅ SERVER COMPONENT - Default, no 'use client'
async function ProjectList() {
  const projects = await db.projects.findMany()

  return (
    <div>
      {projects.map(p => <ProjectCard key={p.id} project={p} />)}
    </div>
  )
}

// ✅ CLIENT COMPONENT - Only when needed
'use client'

import { useState } from 'react'

function CreateProjectForm() {
  const [name, setName] = useState('')

  return <form>...</form>
}
```

**Flag P2 if:**
- 'use client' on components that don't need interactivity
- Fetching data in Client Components instead of Server Components

---

### 4.2 Caching Strategies

```typescript
// ✅ Static - Never revalidates (default for fetch)
const data = await fetch('https://api.example.com/static')

// ✅ Revalidate every 60 seconds
const data = await fetch('https://api.example.com/semi-static', {
  next: { revalidate: 60 }
})

// ✅ Never cache (user-specific)
const data = await fetch('https://api.example.com/user-data', {
  cache: 'no-store'
})

// ✅ On-demand revalidation
import { revalidatePath, revalidateTag } from 'next/cache'

async function updateProject() {
  await db.projects.update(...)
  revalidatePath('/projects')  // Revalidate page
  revalidateTag('projects')    // Revalidate tagged fetches
}
```

---

### 4.3 Route Segment Config

```typescript
// app/dashboard/page.tsx

// ✅ Dynamic (user-specific)
export const dynamic = 'force-dynamic'

// ✅ Static with revalidation
export const revalidate = 3600  // 1 hour

// ✅ Edge runtime for fast responses
export const runtime = 'edge'
```

---

## 5. Common Next.js + Vercel Issues

### 5.1 CORS Configuration

```typescript
// app/api/public/route.ts

export async function GET(request: NextRequest) {
  const data = { message: 'Hello' }

  return NextResponse.json(data, {
    headers: {
      'Access-Control-Allow-Origin': process.env.ALLOWED_ORIGIN || '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    }
  })
}

// Handle preflight
export async function OPTIONS(request: NextRequest) {
  return new NextResponse(null, {
    status: 200,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    }
  })
}
```

**Flag P1 if:**
- CORS allows `*` with credentials in production
- No OPTIONS handler for CORS preflight

---

### 5.2 Image Optimization Security

```typescript
// next.config.js

module.exports = {
  images: {
    // ✅ SECURE - Whitelist domains
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
      },
      {
        protocol: 'https',
        hostname: 's3.amazonaws.com',
        pathname: '/my-bucket/**',
      }
    ],
    // ❌ DANGEROUS - Allows any domain
    // domains: ['*']
  }
}
```

**Flag P0 if:**
- Image domains not whitelisted (SSRF risk)

---

### 5.3 Streaming and Suspense

```tsx
// ✅ SECURE - Streaming with error boundary
import { Suspense } from 'react'
import { ErrorBoundary } from '@/components/ErrorBoundary'

export default function Page() {
  return (
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <ProjectList />
      </Suspense>
    </ErrorBoundary>
  )
}

async function ProjectList() {
  const projects = await fetchProjects()  // Can throw
  return <div>{projects.map(...)}</div>
}
```

---

## 6. Authentication Patterns

### 6.1 NextAuth.js Setup

```typescript
// app/api/auth/[...nextauth]/route.ts

import NextAuth from 'next-auth'
import type { NextAuthOptions } from 'next-auth'

export const authOptions: NextAuthOptions = {
  providers: [
    // ✅ SECURE - OAuth providers
  ],
  callbacks: {
    async session({ session, token }) {
      session.user.id = token.sub
      session.user.role = token.role
      return session
    },
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role
      }
      return token
    }
  },
  pages: {
    signIn: '/login',
  },
  // ✅ SECURE - Custom secret
  secret: process.env.NEXTAUTH_SECRET,
}

const handler = NextAuth(authOptions)
export { handler as GET, handler as POST }
```

**Flag P0 if:**
- No NEXTAUTH_SECRET in production
- JWT secret hardcoded

---

## 7. Next.js + Vercel Security Checklist

### Next.js:
- [ ] Server Actions have auth checks
- [ ] Route Handlers verify authentication
- [ ] Secrets not in client-side config
- [ ] Middleware protects routes
- [ ] 'use client' only when necessary
- [ ] Image domains whitelisted

### Vercel:
- [ ] Environment variables in dashboard
- [ ] Different secrets for preview vs production
- [ ] Cron endpoints secured
- [ ] Edge runtime limitations respected
- [ ] CORS properly configured

### Performance:
- [ ] Server Components used by default
- [ ] Appropriate caching strategies
- [ ] Static pages prerendered
- [ ] Dynamic routes use dynamic rendering

---

## Common Patterns

### Protected Page:
```tsx
import { getServerSession } from 'next-auth'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const session = await getServerSession()

  if (!session) {
    redirect('/login')
  }

  return <div>Dashboard for {session.user.name}</div>
}
```

### Protected API:
```typescript
export async function POST(request: NextRequest) {
  const session = await getServerSession()
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  // Handle request
}
```

### Server Action:
```typescript
'use server'

import { revalidatePath } from 'next/cache'

export async function createProject(formData: FormData) {
  const session = await getServerSession()
  if (!session) throw new Error('Unauthorized')

  const name = formData.get('name') as string
  await db.projects.create({ data: { name, user_id: session.user.id } })

  revalidatePath('/projects')
}
```

---

**See Also:**
- core/02_authentication_authorization.md - Auth patterns
- core/04_api_security.md - API security
- specialized/ssrf_bola.md - SSRF prevention
