---
id: auth-timing-oracle-comparison
domain: auth
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Timing Oracle via Non-Constant-Time Comparison

## Problem
API tokens, HMAC digests, or password hashes are compared with `===`, `==`, or `String.equals()`. These short-circuit on the first differing byte, leaking information about the correct value through response-time differences. An attacker can recover tokens byte-by-byte with enough samples.

```javascript
// WRONG
if (req.headers['x-api-key'] === process.env.API_KEY) { ... }

// Also wrong
if (providedHmac === expectedHmac) { ... }
```

## Root Cause
JavaScript `===` is not constant-time. Under Node.js, string comparison exits early on mismatch. For short secrets (< 32 bytes) the timing signal is measurable over a network with statistical averaging.

## Fix
```javascript
const crypto = require('crypto');

function timingSafeEqual(a, b) {
  const key  = crypto.randomBytes(32);
  const hmacA = crypto.createHmac('sha256', key).update(a).digest();
  const hmacB = crypto.createHmac('sha256', key).update(b).digest();
  return crypto.timingSafeEqual(hmacA, hmacB);
}

if (!timingSafeEqual(providedToken, process.env.API_KEY)) {
  return res.status(401).json({ error: 'Unauthorized' });
}
```

## Test
```javascript
test('auth rejects wrong token without leaking timing info (uses timingSafeEqual)', () => {
  const spy = jest.spyOn(crypto, 'timingSafeEqual');
  auth({ headers: { 'x-api-key': 'wrong' } }, cfg);
  expect(spy).toHaveBeenCalled();
});
```

## Detection
```
===\s*process\.env\.\w*(?:KEY|TOKEN|SECRET|PASSWORD)
==\s*process\.env\.\w*(?:KEY|TOKEN|SECRET|PASSWORD)
apiKey\s*===\s*
token\s*===\s*
```
