---
id: frontend-missing-security-headers
domain: frontend
severity: high
stack: "next.js, express, node.js, vercel"
date_added: 2026-03-25
project: general
---

# Missing HTTP Security Headers

## Problem
Responses lack standard security headers. Without them:
- **No CSP**: any injected script executes freely (XSS amplifier)
- **No X-Frame-Options**: site can be iframed for clickjacking
- **No X-Content-Type-Options**: browser sniffs MIME types, enabling content-type injection attacks
- **No Referrer-Policy**: the full URL (including tokens in query strings) leaks to third parties
- **No HSTS**: downgrade attacks are possible on HTTP

## Root Cause
Frameworks don't ship with security headers by default. AI-generated apps rarely add them because they don't affect visible functionality.

## Fix

**Next.js — `next.config.ts`:**

```typescript
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'nonce-{NONCE}'",   // use nonces for inline scripts
      "style-src 'self' 'unsafe-inline'",     // tighten if possible
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' https://api.example.com",
      "frame-ancestors 'none'",
    ].join('; '),
  },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
]

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

**Express — Helmet:**
```typescript
import helmet from 'helmet'
app.use(helmet())
// Customize CSP:
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    frameAncestors: ["'none'"],
  }
}))
```

## Test

```typescript
// Red: must FAIL — headers absent before the fix

it('sets X-Frame-Options header', async () => {
  const res = await fetch('/')
  expect(res.headers.get('x-frame-options')).toBe('DENY')
})

it('sets X-Content-Type-Options header', async () => {
  const res = await fetch('/')
  expect(res.headers.get('x-content-type-options')).toBe('nosniff')
})

it('sets a Content-Security-Policy header', async () => {
  const res = await fetch('/')
  expect(res.headers.get('content-security-policy')).toBeTruthy()
})

it('does not set CSP with unsafe-eval', async () => {
  const res = await fetch('/')
  expect(res.headers.get('content-security-policy')).not.toContain('unsafe-eval')
})
```

## Detection

```bash
# Check next.config for headers
grep -rn "X-Frame-Options\|Content-Security-Policy\|X-Content-Type" next.config.*

# Check for helmet usage in Express
grep -rn "helmet" src/ package.json

# Runtime check (curl)
curl -sI https://your-app.com | grep -iE "x-frame|csp|x-content|referrer|strict-transport"
```
