---
id: injection-dynamic-import-user-input
domain: injection
severity: critical
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# Dynamic import() with User Input

## Problem
`import()` or `require()` is called with a string that includes or is derived from user input. An attacker can load arbitrary Node.js built-in modules or traverse to any file on the server.

```javascript
// WRONG
app.get('/plugin', async (req, res) => {
  const mod = await import(`./plugins/${req.query.name}`);  // path traversal → RCE
  res.json(mod.run());
});
```

## Root Cause
Plugin/module loader patterns are convenient but developers forget that `import()` resolves paths relative to the module, allowing `../../etc/passwd` or `child_process` as inputs.

## Fix
```javascript
const ALLOWED_PLUGINS = new Set(['csv-parser', 'xml-parser', 'json-parser']);

app.get('/plugin', async (req, res) => {
  const name = req.query.name;
  if (!ALLOWED_PLUGINS.has(name)) return res.status(400).json({ error: 'Unknown plugin' });
  const mod = await import(`./plugins/${name}`);  // name is now allowlisted
  res.json(mod.run());
});
```

## Test
```javascript
test('rejects path traversal in plugin name', async () => {
  const res = await request(app).get('/plugin?name=../../etc/passwd');
  expect(res.status).toBe(400);
});

test('rejects child_process plugin', async () => {
  const res = await request(app).get('/plugin?name=child_process');
  expect(res.status).toBe(400);
});
```

## Detection
```
import\(\s*`[^`]*\$\{(?:req|params|query|body)
require\(\s*(?:req|params|query|body)
import\(\s*(?:req|params|query|body)
```
