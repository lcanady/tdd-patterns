---
id: secrets-hardcoded-env-fallback
domain: secrets
severity: high
stack: "node.js, next.js, *"
date_added: 2026-03-25
project: general
---

# Hardcoded Secret as Environment Variable Fallback

## Problem
A secret (API key, JWT secret, webhook token) is hardcoded as the fallback when the environment variable is not set:

```typescript
const secret = process.env.JWT_SECRET || 'supersecret123'
```

This commits a real-looking secret to source control. Even if the env var is always set in production, the hardcoded fallback:
- Ends up in git history permanently
- Gets used in CI/CD if the env var is not configured
- Signals to attackers the key format and naming convention
- May be the actual dev/staging secret that happens to match production

## Root Cause
Defensive coding instinct: "add a fallback so it doesn't crash if the var is missing." AI-generated code produces this pattern constantly. It's subtly wrong because it trades a loud startup failure for a silent security hole.

## Fix

Fail loudly at startup if required secrets are absent:

```typescript
function requireEnv(name: string): string {
  const val = process.env[name]
  if (!val) throw new Error(`Missing required environment variable: ${name}`)
  return val
}

const JWT_SECRET = requireEnv('JWT_SECRET')
const STRIPE_SECRET = requireEnv('STRIPE_KEY')
const CRON_SECRET = requireEnv('CRON_SECRET')
```

For Next.js, validate in a `lib/env.ts` that is imported at module load:

```typescript
// lib/env.ts — imported by any file that needs these vars
export const env = {
  jwtSecret: requireEnv('JWT_SECRET'),
  stripeKey: requireEnv('STRIPE_KEY'),
  cronSecret: requireEnv('CRON_SECRET'),
} as const
```

This causes `next build` and `next start` to fail immediately and clearly rather than deploying with a hardcoded secret.

## Test

```typescript
// Red: must FAIL — server starts without the secret before the fix

it('throws on startup if JWT_SECRET is missing', async () => {
  const savedSecret = process.env.JWT_SECRET
  delete process.env.JWT_SECRET

  await expect(async () => {
    vi.resetModules()
    await import('../lib/env')
  }).rejects.toThrow('Missing required environment variable: JWT_SECRET')

  process.env.JWT_SECRET = savedSecret
})
```

## Detection

```bash
# env var with hardcoded fallback
grep -rn "process\.env\.\w\+\s*||\s*['\"]" src/
grep -rn "process\.env\.\w\+\s*??\s*['\"]" src/

# Hardcoded key-like strings (20+ alphanumeric chars) in source
grep -rn "API_KEY\s*=\s*['\"][A-Za-z0-9]{20,}" src/
grep -rn "SECRET\s*=\s*['\"][A-Za-z0-9]{16,}" src/

# Secret history scan (run separately)
# gitleaks detect --source .
# trufflehog filesystem .
```
