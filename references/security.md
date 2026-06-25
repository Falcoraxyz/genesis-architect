# Security Consciousness Framework

## Mental Model

Security is not a feature added at the end. It's a way of reading code.

Every time a route is written, read it as an attacker first:
- What does this expose if called without authentication?
- What does this expose if called by a different user?
- What happens if I send unexpected input?

The answer to those three questions catches 80% of real-world vulnerabilities.

---

## Layer 1 — Authentication & Authorization

### The IDOR trap (most common real-world bug)

Insecure Direct Object Reference: a logged-in user accessing another user's data by guessing an ID.

```typescript
// BROKEN — checks login but not ownership
export async function getInvoice(invoiceId: string) {
  const user = await requireUser()
  return db.invoices.findById(invoiceId)
  // Any logged-in user can read any invoice by changing the ID
}

// CORRECT — verifies ownership before returning data
export async function getInvoice(invoiceId: string) {
  const user = await requireUser()
  const invoice = await db.invoices.findById(invoiceId)
  if (!invoice) throw new NotFoundError()
  if (invoice.userId !== user.id) throw new ForbiddenError()
  return invoice
}
```

### Supabase RLS as the last defense line

Application logic can have bugs. RLS does not care:

```sql
-- Users can only see and modify their own records
create policy "users_own_records" on invoices
  for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- Public read, owner write
create policy "public_read" on posts
  for select using (true);

create policy "owner_write" on posts
  for insert, update, delete
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

Enable RLS on every table. Without exception.

---

## Layer 2 — Input Validation

### Validate at every trust boundary with Zod

A trust boundary is anywhere data enters from outside your process: HTTP request body, query params, headers, file uploads, webhook payloads, cron job arguments.

```typescript
import { z } from 'zod'

// Define the schema once, reuse for validation and types
const CreatePostSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  body: z.string().min(1).max(10_000),
  published: z.boolean().default(false),
  // Never include: id, userId, createdAt, updatedAt — server sets these
})

type CreatePostInput = z.infer<typeof CreatePostSchema>

export async function createPost(raw: unknown) {
  const input = CreatePostSchema.parse(raw)
  // input is typed and safe — schema.parse throws ZodError if invalid
  return db.posts.create({ ...input, userId: (await requireUser()).id })
}
```

### File upload validation

```typescript
const ALLOWED_MIME_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp', 'application/pdf'])
const MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024 // 10MB

export function validateUpload(file: File): void {
  if (!ALLOWED_MIME_TYPES.has(file.type)) {
    throw new ValidationError(`File type "${file.type}" not permitted`)
  }
  if (file.size > MAX_FILE_SIZE_BYTES) {
    throw new ValidationError(`File size ${file.size} exceeds ${MAX_FILE_SIZE_BYTES} limit`)
  }
}
```

Never trust the file extension. Check the MIME type. For sensitive applications, validate the magic bytes too.

---

## Layer 3 — Rate Limiting

Every public-facing endpoint. Not just login.

```typescript
// middleware.ts (Vercel Edge — runs before any route handler)
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(20, '10 s'),
  analytics: true,
})

export async function middleware(req: NextRequest) {
  // Use user ID if authenticated, fall back to IP
  const identifier = req.headers.get('x-user-id') ?? req.ip ?? '127.0.0.1'
  const { success, remaining, reset } = await ratelimit.limit(identifier)

  if (!success) {
    return NextResponse.json(
      { error: 'Too many requests', reset: new Date(reset).toISOString() },
      {
        status: 429,
        headers: { 'Retry-After': String(Math.ceil((reset - Date.now()) / 1000)) },
      }
    )
  }

  return NextResponse.next()
}

export const config = {
  matcher: '/api/:path*',
}
```

Set stricter limits on auth endpoints (`3 per 60s`) versus data endpoints (`20 per 10s`).

---

## Layer 4 — HTTP Security Headers

```typescript
// next.config.ts
const ContentSecurityPolicy = [
  "default-src 'self'",
  "script-src 'self' 'unsafe-inline'",   // tighten to nonce-based in production
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data: blob: https:",
  `connect-src 'self' ${process.env.SUPABASE_URL} wss://${new URL(process.env.SUPABASE_URL ?? 'https://x').host}`,
  "font-src 'self'",
  "object-src 'none'",
  "frame-ancestors 'none'",
].join('; ')

const securityHeaders = [
  { key: 'Content-Security-Policy',         value: ContentSecurityPolicy },
  { key: 'Strict-Transport-Security',       value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Frame-Options',                 value: 'DENY' },
  { key: 'X-Content-Type-Options',          value: 'nosniff' },
  { key: 'Referrer-Policy',                 value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy',              value: 'camera=(), microphone=(), geolocation=()' },
]

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

---

## Layer 5 — Secrets & Environment Variables

### Never expose server secrets to the client bundle

```typescript
// WRONG — this runs on the client, key is exposed
'use client'
const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_SERVICE_KEY!)

// CORRECT — service key stays on the server
// src/server/db/client.ts (never imported from client components)
import { env } from '@/config/env'
export const supabase = createClient(env.SUPABASE_URL, env.SUPABASE_SERVICE_KEY)
```

Verify after every build:
```bash
# Should return nothing — if it prints a key, you have a leak
grep -r "SUPABASE_SERVICE_KEY" .next/static/ 2>/dev/null
```

### Validate all env vars at startup

If a required variable is missing, the app should refuse to start — not crash at runtime.

```typescript
// src/config/env.ts
import { z } from 'zod'

const EnvSchema = z.object({
  NODE_ENV:              z.enum(['development', 'test', 'production']).default('development'),
  SUPABASE_URL:          z.string().url(),
  SUPABASE_ANON_KEY:     z.string().min(20),
  SUPABASE_SERVICE_KEY:  z.string().min(20),
  UPSTASH_REDIS_REST_URL:   z.string().url().optional(),
  UPSTASH_REDIS_REST_TOKEN: z.string().optional(),
})

export const env = EnvSchema.parse(process.env)
```

---

## Layer 6 — Dependencies

```bash
# Run before every deploy
npm audit --audit-level=high

# Fail CI if vulnerabilities exist
npx audit-ci --high
```

If a vulnerability cannot be patched:
1. Confirm it's in the execution path (not a devDependency, not dead code)
2. Document the risk and accepted workaround in `docs/security-exceptions.md`
3. Create a ticket to resolve it — do not leave it in silence

---

## Threat Model Template

Complete this before shipping any project with real user data:

```markdown
## Threat Model — [Project Name] — [Date]

### What's worth attacking?
[List the most valuable data and the most critical functions]

### Who might attack this?
[Describe realistic attacker types — not "nation state APT", but "bored competitor", "automated scanner", "disgruntled user"]

### Attack surface
Public routes:
  GET  /api/posts        (unauthenticated)
  POST /api/posts        (authenticated)
  DELETE /api/posts/:id  (owner only)

File upload:
  POST /api/upload  (authenticated, images only, 10MB limit)

Auth:
  POST /api/auth/login   (3 req / 60s rate limit)
  POST /api/auth/signup

### Top risks

1. [Attack]: A user deletes another user's post by guessing IDs
   [Mitigation]: Ownership check in route handler + RLS policy

2. [Attack]: Brute force login with known email list
   [Mitigation]: Rate limit 3/60s per IP + account lockout after 10 fails

3. [Attack]: Upload a malicious file disguised as an image
   [Mitigation]: MIME type check + file size limit + store in isolated Supabase bucket with no execute permissions

### If the database leaked today
Sensitive fields: email, hashed passwords, [list others]
Encryption status: [encrypted at rest by Supabase / needs column-level encryption]
Passwords: [bcrypt hashed via Supabase Auth — not exposed as plaintext]
Worst case: [describe what an attacker gains]

### Pre-ship checklist
- [ ] Tested unauthenticated access to all authenticated routes
- [ ] Tested IDOR on all resource-specific routes (tried accessing other users' records)
- [ ] Tested input with SQL injection payloads
- [ ] Confirmed rate limits activate under load
- [ ] Ran npm audit — no high or critical findings
- [ ] Scanned client bundle for leaked server secrets
- [ ] Verified RLS policies block unauthorized Supabase direct access
- [ ] Reviewed CSP headers in browser dev tools Network tab
```
