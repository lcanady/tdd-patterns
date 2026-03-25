---
id: frontend-vercel-deployment-protection
domain: frontend
severity: medium
stack: "vercel, next.js"
date_added: 2026-03-25
project: beltway-events
---

# Vercel Deployment Protection Blocks Preview Testing

## Problem
Vercel's Deployment Protection (SSO) is enabled on preview deployments by default. This blocks:
- Public users from accessing preview URLs (E2E tests, stakeholder review)
- Confirmation and reset email links from working (link hits Vercel SSO wall)
- Any automated fetch to the preview URL without a bypass token

## Symptoms
- Preview URL returns an "Authentication Required" page instead of 404 or the actual route
- Confirmation emails appear sent but links are broken for non-Vercel-team users
- `curl`/`fetch` to preview URL returns HTML for Vercel's SSO redirect page, not JSON
- Debug endpoints appear to return 404 when they're actually returning 200 behind the SSO wall

## Fix

**Vercel Dashboard** → Project → Settings → Deployment Protection:
- Set to **"Only Preview Deployments"** with protection **disabled**, or
- Use a bypass token for automated tests

Bypass token approach (E2E / CI):
```
https://your-preview.vercel.app/api/route?x-vercel-protection-bypass=TOKEN
```
Token: Vercel → Settings → Deployment Protection → "Bypass for Automation".

## Related

Also set `NEXT_PUBLIC_SITE_URL` explicitly in the Vercel Preview environment to the stable branch URL. Without this, emails generated from the preview deployment link to the production site even after the SSO issue is resolved. See `frontend-environment-safe-urls.md`.

## Test

```bash
# Red: must FAIL — preview URL returns SSO page, not the route

curl -s https://your-preview.vercel.app/api/health | jq .
# Before fix: returns HTML for Vercel SSO page
# After fix: returns {"ok":true}
```

```typescript
it('preview API returns JSON, not an SSO redirect page', async () => {
  const res = await fetch(process.env.PREVIEW_URL + '/api/health')
  expect(res.headers.get('content-type')).toContain('application/json')
  const body = await res.json()
  expect(body.ok).toBe(true)
})
```
