---
id: injection-ssrf-user-controlled-url
domain: injection
severity: high
stack: "node.js, python, go, *"
date_added: 2026-03-25
project: general
---

# SSRF — Server-Side Request Forgery via User-Controlled URL

## Problem
The server makes an outbound HTTP request to a URL supplied by the user. An attacker can point this at internal services (AWS metadata, internal APIs, localhost), cloud provider credentials endpoints, or use it to port-scan private networks.

## Root Cause
AI-generated proxy endpoints, webhook validators, OG metadata scrapers, and URL preview features all tend to pass `req.body.url` directly to `fetch()` or `axios()`.

```typescript
// WRONG
app.post('/api/preview', async (req, res) => {
  const data = await fetch(req.body.url) // SSRF
  res.json(await data.json())
})
```

Classic targets: `http://169.254.169.254/latest/meta-data/` (AWS credentials), `http://localhost:6379` (Redis), `http://internal-api/admin`.

## Fix

Validate the URL against an allowlist of permitted origins before making the request:

```typescript
const ALLOWED_ORIGINS = new Set([
  'https://api.github.com',
  'https://api.stripe.com',
])

function isSafeUrl(raw: string): boolean {
  try {
    const url = new URL(raw)
    // Block private / loopback ranges
    if (['localhost', '127.0.0.1', '0.0.0.0', '::1'].includes(url.hostname)) return false
    if (/^(10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|169\.254\.)/.test(url.hostname)) return false
    // Only allow https and known origins
    if (url.protocol !== 'https:') return false
    if (!ALLOWED_ORIGINS.has(url.origin)) return false
    return true
  } catch {
    return false
  }
}

app.post('/api/preview', async (req, res) => {
  if (!isSafeUrl(req.body.url)) return res.status(400).json({ error: 'Invalid URL' })
  const data = await fetch(req.body.url)
  res.json(await data.json())
})
```

For cases where an allowlist is impractical, use a dedicated outbound proxy with egress filtering.

## Test

```typescript
// Red: must FAIL — SSRF succeeds before the fix

it('blocks requests to AWS metadata endpoint', async () => {
  const res = await fetch('/api/preview', {
    method: 'POST',
    body: JSON.stringify({ url: 'http://169.254.169.254/latest/meta-data/iam/security-credentials/' })
  })
  expect(res.status).toBe(400)
})

it('blocks requests to localhost', async () => {
  const res = await fetch('/api/preview', {
    method: 'POST',
    body: JSON.stringify({ url: 'http://localhost:6379' })
  })
  expect(res.status).toBe(400)
})

it('allows requests to permitted origins', async () => {
  const res = await fetch('/api/preview', {
    method: 'POST',
    body: JSON.stringify({ url: 'https://api.github.com/users/octocat' })
  })
  expect(res.status).toBe(200)
})
```

## Detection

```bash
# fetch/axios with user-controlled URL
grep -rn "fetch(.*req\.(query\|body\|params)" src/
grep -rn "axios\.(get\|post)(.*req\.body" src/
grep -rn "got(.*req\.(query\|params)" src/
```
