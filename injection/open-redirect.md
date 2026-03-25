---
id: injection-open-redirect
domain: injection
severity: high
stack: "node.js, next.js, express, *"
date_added: 2026-03-25
project: general
---

# Open Redirect via Unvalidated `next` / `redirect` Parameter

## Problem
A route reads a destination URL from user input and redirects to it without validation. An attacker crafts a link like `/login?next=https://evil.com` — users follow a trusted domain URL and are silently redirected to a phishing site.

## Root Cause
Post-login redirects and OAuth return URLs commonly take their destination from a query parameter. AI-generated auth flows pass this directly to `res.redirect()` or `router.push()`.

```typescript
// WRONG
const next = req.query.next as string
return res.redirect(next) // attacker can set next=https://evil.com

// WRONG — Next.js
const { searchParams } = new URL(request.url)
redirect(searchParams.get('next') ?? '/')
```

## Fix

Validate the redirect target is a relative path on the same origin:

```typescript
function isSafeRedirect(url: string): boolean {
  // Allow only relative paths starting with /
  // Block protocol-relative URLs (//evil.com) and absolute URLs
  return url.startsWith('/') && !url.startsWith('//')
}

// Usage
const next = (req.query.next as string) ?? '/'
const destination = isSafeRedirect(next) ? next : '/'
return res.redirect(destination)
```

For OAuth return URLs that must be absolute, validate against an allowlist of your own origins:

```typescript
const ALLOWED_ORIGINS = ['https://app.example.com', 'https://preview.example.com']

function isSafeAbsoluteRedirect(url: string): boolean {
  try {
    const parsed = new URL(url)
    return ALLOWED_ORIGINS.includes(parsed.origin)
  } catch {
    return false
  }
}
```

## Test

```typescript
// Red: must FAIL — redirect to external URL succeeds before the fix

it('does not redirect to an external URL', async () => {
  const res = await fetch('/login?next=https://evil.com', { redirect: 'manual' })
  const location = res.headers.get('location') ?? ''
  expect(location).not.toContain('evil.com')
})

it('does not follow protocol-relative redirect', async () => {
  const res = await fetch('/login?next=//evil.com', { redirect: 'manual' })
  const location = res.headers.get('location') ?? ''
  expect(location).not.toMatch(/^\/\//)
})

it('redirects to safe relative path after login', async () => {
  const res = await loginAndFetch('/login?next=/dashboard', { redirect: 'manual' })
  expect(res.headers.get('location')).toBe('/dashboard')
})
```

## Detection

```bash
# res.redirect with query/body param
grep -rn "res\.redirect(.*req\.(query\|body)" src/
grep -rn "redirect(.*searchParams\|redirect(.*params\." src/

# Next.js router.push with user param
grep -rn "router\.push(.*searchParams\|router\.push(.*params\." src/
grep -rn "window\.location.*=.*params\." src/
```
