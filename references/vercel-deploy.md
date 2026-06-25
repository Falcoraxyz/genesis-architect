# Vercel Deployment — Setup & Operations

## First-Time Setup

### 1. Supabase Project

Go to supabase.com → New Project. After creation:

```
Project URL:         https://[your-project-id].supabase.co
Anon (public) key:   eyJ...
Service role key:    eyJ...  ← server-side only, never in client code
```

Run the schema from `references/memory-schema.md` in the Supabase SQL editor.
Enable RLS on all tables (the schema SQL does this automatically).

### 2. Local Vercel CLI

```bash
npm install -g vercel
vercel login
vercel link   # run from project root — links to your Vercel account
```

### 3. Environment Variables

Set via CLI (recommended — avoids copy-paste errors):

```bash
vercel env add SUPABASE_URL          # paste your project URL
vercel env add SUPABASE_ANON_KEY     # paste anon key (safe for client)
vercel env add SUPABASE_SERVICE_KEY  # paste service key (server only)
```

Or via the Vercel dashboard: Project → Settings → Environment Variables.

Set all three for **Production**, **Preview**, and **Development** environments.

If using rate limiting:
```bash
vercel env add UPSTASH_REDIS_REST_URL
vercel env add UPSTASH_REDIS_REST_TOKEN
```

### 4. vercel.json

```json
{
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "installCommand": "npm ci",
  "regions": ["sin1"],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Access-Control-Allow-Origin",
          "value": "https://yourdomain.com"
        },
        {
          "key": "Access-Control-Allow-Methods",
          "value": "GET, POST, PUT, DELETE, OPTIONS"
        }
      ]
    }
  ]
}
```

Replace `sin1` with your preferred region (`iad1` US East, `cdg1` EU West, `hnd1` Japan).

### 5. next.config.ts

```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Strict mode catches double-render bugs early
  reactStrictMode: true,

  // Explicitly mark server-only packages
  serverExternalPackages: ['@supabase/supabase-js'],

  // Security headers (full list in references/security.md)
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options',       value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload',
          },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
    ]
  },
}

export default nextConfig
```

---

## .env Files

```bash
# .env.example — commit this to the repo
SUPABASE_URL=https://your-project-id.supabase.co
SUPABASE_ANON_KEY=your-anon-key-here
SUPABASE_SERVICE_KEY=your-service-role-key-here

# Optional: rate limiting
UPSTASH_REDIS_REST_URL=https://your-upstash-url.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-upstash-token
```

```bash
# .env.local — never commit, in .gitignore
SUPABASE_URL=https://actual-project-id.supabase.co
SUPABASE_ANON_KEY=eyJhbGc...actual-key
SUPABASE_SERVICE_KEY=eyJhbGc...actual-service-key
```

```bash
# .gitignore additions
.env.local
.env.*.local
memory/brain.json   # generated, not source-controlled
```

---

## Deploy Commands

```bash
# Build check locally before any deploy
npm run build
npm run type-check

# Deploy to preview URL (branch deploy)
vercel

# Deploy to production
vercel --prod

# Promote a specific preview to production
vercel promote [deployment-url]
```

---

## CI/CD with GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  quality:
    name: Type check, lint, audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npm run lint

      - name: Security audit
        run: npm audit --audit-level=high

      - name: Build
        run: npm run build
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
          SUPABASE_SERVICE_KEY: ${{ secrets.SUPABASE_SERVICE_KEY }}
```

Add secrets in GitHub: Repository → Settings → Secrets and Variables → Actions.

---

## Preview Environments with Supabase Branching

Each pull request gets its own isolated database. No shared state between feature branches.

```bash
# Install Supabase CLI
npm install -g supabase
supabase login

# Create a database branch for the PR
supabase branches create feature/my-feature --project-ref your-project-id
```

In Vercel, the preview environment automatically gets a different `SUPABASE_URL` pointing to the branch database. Set this up once in Vercel dashboard under Environment Variables → override for Preview.

---

## Post-Deploy Verification

Run after every production deploy:

```bash
# 1. Health check
curl -f https://yourdomain.com/api/health

# 2. Check no server secrets are in the client bundle
curl -s https://yourdomain.com/_next/static/chunks/main.js | grep -i "service_key"
# Should return nothing

# 3. Verify security headers
curl -I https://yourdomain.com | grep -E "(x-frame|strict-transport|x-content)"
```

---

## Brain Map on Vercel

The `docs/brainmap.html` file is served as a static asset by Vercel automatically.

To expose it at a clean URL, add this route:

```typescript
// src/app/map/route.ts
import { readFileSync } from 'fs'
import { join } from 'path'
import { NextResponse } from 'next/server'

export async function GET() {
  const html = readFileSync(join(process.cwd(), 'docs', 'brainmap.html'), 'utf-8')
  return new NextResponse(html, {
    headers: { 'Content-Type': 'text/html' },
  })
}
```

Access at: `https://yourdomain.com/map`

Or just link directly to `https://yourdomain.com/brainmap.html` if the file is in the `public/` directory instead of `docs/`.

---

## Monitoring

Vercel provides out of the box:
- **Runtime Logs**: Dashboard → Functions → Logs (real-time, filterable)
- **Web Analytics**: Enable in project settings (page views, performance)
- **Speed Insights**: Enable in project settings (Core Web Vitals per route)

For structured logging in server functions:

```typescript
// src/lib/log.ts
type Level = 'info' | 'warn' | 'error'

export function log(level: Level, message: string, meta?: Record<string, unknown>): void {
  const entry = { level, message, ...meta, timestamp: new Date().toISOString() }
  // Vercel captures stdout — JSON format enables structured log queries in dashboard
  console[level === 'error' ? 'error' : 'log'](JSON.stringify(entry))
}
```

Usage:
```typescript
log('info', 'Post created', { postId: post.id, userId: user.id })
log('error', 'Supabase query failed', { table: 'posts', error: err.message })
```

---

## Rollback

If a deploy breaks production:

```bash
# List recent deployments
vercel ls

# Instant rollback to previous production deployment
vercel rollback [previous-deployment-url]
```

Rollback is instant — Vercel switches traffic at the edge with no rebuild required.
