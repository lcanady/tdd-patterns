---
id: auth-jwt-no-verification
domain: auth
severity: critical
stack: "node.js, express, next.js"
date_added: 2026-03-25
project: general
---

# JWT Decoded Without Signature Verification

## Problem
A JWT is decoded (base64-split) to read its payload claims, but the signature is never verified. Any attacker can forge a token with arbitrary claims — including `is_admin: true` or any `user_id` — and the server will trust it.

## Root Cause
`jwt.decode()` (jsonwebtoken) returns the payload without verifying the signature. AI-generated code frequently uses `decode` when `verify` is required, either because decode "works" locally with no apparent failure, or because the developer confused the two.

```typescript
// WRONG — decode reads payload without verifying the signature
const payload = jwt.decode(token)
if (payload.userId) { ... }
```

## Fix

Always use `jwt.verify()` with the secret:

```typescript
import jwt from 'jsonwebtoken'

function verifyToken(token: string) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload
  } catch {
    return null
  }
}

// In route handler
const payload = verifyToken(req.headers.authorization?.split(' ')[1] ?? '')
if (!payload) return res.status(401).json({ error: 'Unauthorized' })
```

For Supabase / Auth.js: use the SDK's `getUser()` which handles verification internally. Never manually decode the Supabase JWT.

## Test

```typescript
// Red: must FAIL — server accepts a forged token before the fix

it('rejects a token signed with the wrong secret', async () => {
  const forgedToken = jwt.sign({ userId: 'admin', is_admin: true }, 'wrong-secret')
  const res = await fetch('/api/protected', {
    headers: { Authorization: `Bearer ${forgedToken}` }
  })
  expect(res.status).toBe(401)
})

it('rejects a tampered token payload', async () => {
  const [header, , sig] = validToken.split('.')
  const tamperedPayload = Buffer.from(JSON.stringify({ userId: 'other', is_admin: true })).toString('base64url')
  const tamperedToken = `${header}.${tamperedPayload}.${sig}`
  const res = await fetch('/api/protected', {
    headers: { Authorization: `Bearer ${tamperedToken}` }
  })
  expect(res.status).toBe(401)
})
```

## Detection

```bash
# Find jwt.decode calls (should be jwt.verify)
grep -rn "jwt\.decode(" src/

# Find verification-disabled flags
grep -rn "algorithms.*none\|verify.*false\|ignoreExpiration.*true" src/
```
