---
id: frontend-x-powered-by-exposed
domain: frontend
severity: low
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# X-Powered-By Header Exposes Technology Stack

## Problem
Express (and many other frameworks) send `X-Powered-By: Express` by default. This fingerprinting header tells attackers exactly which framework version family to target for known CVEs.

```javascript
// WRONG — default Express behaviour
const app = express(); // sends X-Powered-By: Express on every response
```

## Root Cause
It's an opt-out feature that developers don't disable, and it's rarely noticed until a security review.

## Fix
```javascript
// Option 1: disable in Express
app.disable('x-powered-by');

// Option 2: use Helmet (also sets many other security headers)
const helmet = require('helmet');
app.use(helmet());
```

For Fastify, the framework does not send this header by default.

## Test
```javascript
test('no X-Powered-By header in responses', async () => {
  const res = await request(app).get('/health');
  expect(res.headers['x-powered-by']).toBeUndefined();
});
```

## Detection
```
app\.(?:use|set)\(\s*['"]x-powered-by['"]   # if explicitly re-enabled
express\(\)(?![\s\S]{0,200}disable.*x-powered-by)
# absence of helmet() or app.disable('x-powered-by')
```
