---
id: infra-body-parser-dos
domain: infra
severity: high
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# Body Parser DoS — No Request Size Limit

## Problem
`express.json()` or `body-parser` is configured without a `limit`, defaulting to 100 KB in some versions but unbounded in others. An attacker sends a multi-megabyte or multi-gigabyte JSON body, exhausting server memory and causing denial of service.

```javascript
// WRONG — no size limit
app.use(express.json());

// Also wrong — old body-parser default
app.use(bodyParser.json({ limit: '50mb' }));  // far too large for most APIs
```

## Root Cause
Developers copy tutorial snippets that omit the limit. The "50mb" value often comes from copy-pasting upload examples into JSON API code.

## Fix
```javascript
// Set a strict per-endpoint limit appropriate to your payloads
app.use(express.json({ limit: '64kb' }));
app.use(express.urlencoded({ extended: false, limit: '64kb' }));
```

Use `Content-Length` enforcement at the reverse proxy layer as a defence-in-depth measure.

## Test
```javascript
test('rejects body larger than limit with 413', async () => {
  const hugePayload = { data: 'x'.repeat(200_000) };
  const res = await request(app)
    .post('/api/data')
    .set('Content-Type', 'application/json')
    .send(JSON.stringify(hugePayload));
  expect(res.status).toBe(413);
});
```

## Detection
```
express\.json\(\s*\)(?!\s*\))                     # no options at all
express\.json\(\s*\{(?![\s\S]{0,100}limit)        # options but no limit
bodyParser\.json\(\s*\{[^}]*limit\s*:\s*['"](?:5\d|[6-9]\d|\d{3,})(?:mb|MB)
```
