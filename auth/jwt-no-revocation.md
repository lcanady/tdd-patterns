---
id: auth-jwt-no-revocation
domain: auth
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# JWT Without Revocation Mechanism

## Problem
JWTs are verified for signature and expiry but there is no revocation list or session store. A stolen or logged-out token remains valid until its `exp` claim elapses — potentially hours or days.

```javascript
// WRONG — only checks signature; stolen tokens cannot be invalidated
app.use((req, res, next) => {
  const token = req.headers.authorization?.slice(7);
  req.user = jwt.verify(token, process.env.JWT_SECRET);
  next();
});
```

## Root Cause
JWTs are stateless by design. Developers embrace this for scalability but don't implement a revocation layer for logout, password changes, or compromised-token scenarios.

## Fix
```javascript
// Maintain a Redis allowlist of active token JTIs
async function verifyToken(token) {
  const payload = jwt.verify(token, process.env.JWT_SECRET);
  const valid = await redis.get(`session:${payload.jti}`);
  if (!valid) throw new Error('Token revoked or expired');
  return payload;
}

// On logout: redis.del(`session:${jti}`)
// On password change: redis.del all sessions for that user
```

Short token TTL (≤ 15 min) + refresh token pattern reduces the exposure window without a full allowlist.

## Test
```javascript
test('revoked JWT is rejected even before exp', async () => {
  const { token, jti } = await loginUser();
  await redis.del(`session:${jti}`);   // simulate logout
  await expect(verifyToken(token)).rejects.toThrow(/revoked/i);
});
```

## Detection
```
jwt\.verify\(
jsonwebtoken.*verify
PyJWT.*decode
# look for absence of revocation check (no allowlist/denylist lookup after verify)
```
