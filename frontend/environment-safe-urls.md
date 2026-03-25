---
id: frontend-environment-safe-urls
domain: frontend
severity: high
stack: "next.js, vercel, *"
date_added: 2026-03-25
project: beltway-events
---

# Hardcoded Production URLs in Transactional Flows

## Problem
Confirmation emails, RSVP links, password reset links, and unsubscribe URLs are built with hardcoded production domain strings. Preview and staging deployments send users to the production site — breaking preview testing and potentially leaking preview data into production flows.

## Root Cause
URLs were written as string literals: `` `https://myapp.com/auth/confirm?token=${token}` ``

The environment variable (`NEXT_PUBLIC_SITE_URL`) was only configured in the Production Vercel environment, so the hardcoded string was the only fallback available on preview branches.

## Fix

All generated URLs must go through a safe URL builder that walks a fallback chain:

```typescript
// lib/site-url.ts
function getVercelAutoUrl(): string | null {
  // VERCEL_BRANCH_URL: stable per branch, auto-injected by Vercel (no https:// prefix)
  // VERCEL_URL: per-deployment, auto-injected (no https:// prefix)
  const raw = process.env.VERCEL_BRANCH_URL || process.env.VERCEL_URL
  return raw ? `https://${raw}` : null
}

const ALLOWED_ORIGINS = ['https://myapp.com', 'https://staging.myapp.com']

export function getSafeSiteUrl(rawUrl?: string | null): string {
  const candidate = (
    rawUrl ||
    process.env.NEXT_PUBLIC_SITE_URL ||
    getVercelAutoUrl() ||
    'https://myapp.com'  // production fallback — last resort only
  ).trim().replace(/\/$/, '')

  try {
    const { origin } = new URL(candidate)
    // Require https in production
    if (process.env.NODE_ENV === 'production' && !origin.startsWith('https://')) {
      return 'https://myapp.com'
    }
    return origin
  } catch {
    return 'https://myapp.com'
  }
}
```

Priority order:
1. `NEXT_PUBLIC_SITE_URL` — explicit override (set per environment in Vercel)
2. `VERCEL_BRANCH_URL` — stable per branch
3. `VERCEL_URL` — per-deployment
4. Hardcoded default — last resort

**Exceptions** (SEO artifacts — hardcoded production URL is correct):
- `sitemap.ts`, `robots.ts`
- `layout.tsx` `metadataBase` (OG tags)

## Test

```typescript
it('uses NEXT_PUBLIC_SITE_URL when set', async () => {
  process.env.NEXT_PUBLIC_SITE_URL = 'https://preview.myapp.com'
  vi.resetModules()
  const { getSafeSiteUrl } = await import('../lib/site-url')
  expect(getSafeSiteUrl()).toBe('https://preview.myapp.com')
})

it('falls back to VERCEL_BRANCH_URL when NEXT_PUBLIC_SITE_URL is not set', async () => {
  delete process.env.NEXT_PUBLIC_SITE_URL
  process.env.VERCEL_BRANCH_URL = 'myapp-git-dev.vercel.app'
  vi.resetModules()
  const { getSafeSiteUrl } = await import('../lib/site-url')
  expect(getSafeSiteUrl()).toBe('https://myapp-git-dev.vercel.app')
})

it('confirmation email links to the current environment, not hardcoded production', async () => {
  process.env.NEXT_PUBLIC_SITE_URL = 'https://preview.myapp.com'
  const email = buildConfirmationEmail({ token: 'abc123' })
  expect(email.html).toContain('https://preview.myapp.com')
  expect(email.html).not.toContain('https://myapp.com/auth/confirm')
})
```

## Detection

```bash
# Hardcoded domain in non-SEO files
grep -rn "https://myapp\.com" src/ \
  | grep -v "sitemap\|robots\|layout\|metadataBase"

# getSafeSiteUrl not imported in email/url-generating files
grep -rn "sendEmail\|buildEmail\|confirmUrl\|resetUrl" src/ -l \
  | xargs grep -L "getSafeSiteUrl"
```
