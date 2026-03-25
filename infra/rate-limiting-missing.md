---
id: infra-rate-limiting-missing
domain: infra
severity: high
stack: "node.js, express, next.js, *"
date_added: 2026-03-25
project: general
---

# Missing Rate Limiting on Auth and Mutation Routes

## Problem
Auth routes (`/login`, `/register`, `/forgot-password`) and high-value mutation routes have no rate limiting. An attacker can:
- Brute-force passwords with no throttling
- Enumerate valid email addresses via timing differences
- Abuse expensive operations (email sends, DB writes) at scale

## Root Cause
Rate limiting is never visible in the happy path. AI-generated code builds the feature first; cross-cutting concerns like rate limiting rarely make it into initial generation.

## Fix

**Express — express-rate-limit:**
```typescript
import rateLimit from 'express-rate-limit'

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                    // 10 attempts per IP per window
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many attempts, please try again later.' },
  skipSuccessfulRequests: true,  // only count failures
})

app.post('/api/auth/login', authLimiter, loginHandler)
app.post('/api/auth/register', authLimiter, registerHandler)
app.post('/api/auth/forgot-password', authLimiter, forgotPasswordHandler)
```

**Next.js — middleware or per-route:**
```typescript
// lib/rate-limit.ts — Upstash Redis or in-memory for edge
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '15 m'),
})

export async function isIpRateLimited(req: NextRequest): Promise<boolean> {
  const ip = req.ip ?? req.headers.get('x-forwarded-for') ?? 'unknown'
  const { success } = await ratelimit.limit(ip)
  return !success
}

// In route handler
if (await isIpRateLimited(request)) {
  return NextResponse.json({ error: 'Too many requests' }, { status: 429 })
}
```

## Test

```typescript
// Red: must FAIL — no rate limit enforced before the fix

it('returns 429 after too many failed login attempts', async () => {
  const attempts = Array.from({ length: 15 }, () =>
    fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email: 'user@example.com', password: 'wrong' })
    })
  )
  const responses = await Promise.all(attempts)
  const statuses = responses.map(r => r.status)
  expect(statuses).toContain(429)
})

it('allows requests under the rate limit', async () => {
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email: 'user@example.com', password: 'correct' })
  })
  expect(res.status).not.toBe(429)
})
```

## Detection

```bash
# Auth routes without rate limit middleware
grep -rn "router\.(post\|put\|delete)\|app\.post\|app\.put" src/ \
  | grep -iE "login\|register\|forgot\|reset\|password\|verify" \
  | grep -v "rateLimit\|rateLimiter\|isIpRateLimited\|throttle"

# POST handlers in Next.js API routes without rate limit import
grep -rn "export async function POST" src/app/api/ -l \
  | xargs grep -L "rateLimit\|isIpRateLimited"
```
