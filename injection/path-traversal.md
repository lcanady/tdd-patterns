---
id: injection-path-traversal
domain: injection
severity: critical
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Path Traversal (Directory Traversal)

## Problem
User-controlled input is concatenated into a file path without normalisation, allowing `../../etc/passwd` sequences to escape the intended directory.

```javascript
// WRONG
app.get('/file', (req, res) => {
  const filePath = path.join('./uploads', req.query.name);
  res.sendFile(filePath);  // '../../../etc/passwd' escapes uploads/
});
```

## Root Cause
`path.join()` removes leading slashes but does not prevent `..` traversal. Developers assume the join is sufficient sanitisation.

## Fix
```javascript
const BASE_DIR = path.resolve('./uploads');

app.get('/file', (req, res) => {
  const resolved = path.resolve(BASE_DIR, req.query.name);
  if (!resolved.startsWith(BASE_DIR + path.sep) && resolved !== BASE_DIR) {
    return res.status(400).send('Invalid path');
  }
  res.sendFile(resolved);
});
```

Key: resolve first, then check the prefix with `path.sep` to avoid false-positive prefix matches (e.g. `/uploads-extra`).

## Test
```javascript
test('rejects path traversal to /etc/passwd', async () => {
  const res = await request(app).get('/file?name=../../etc/passwd');
  expect(res.status).toBe(400);
});

test('allows access to valid file in uploads/', async () => {
  fs.writeFileSync('./uploads/test.txt', 'hello');
  const res = await request(app).get('/file?name=test.txt');
  expect(res.status).toBe(200);
});
```

## Detection
```
path\.join\([^)]*(?:req\.|params\.|query\.|body\.)
path\.resolve\([^)]*(?:req\.|params\.|query\.|body\.)(?![\s\S]{0,200}startsWith)
__dirname.*req\.|req\..*__dirname
```
