---
id: injection-nosql-injection
domain: injection
severity: critical
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# NoSQL Injection (MongoDB Operator Injection)

## Problem
User-supplied JSON objects are passed directly into MongoDB queries. An attacker sends `{ "$gt": "" }` as a username value, which MongoDB interprets as the operator "greater than empty string" — matching every document and bypassing authentication.

```javascript
// WRONG
app.post('/login', async (req, res) => {
  const user = await User.findOne({
    username: req.body.username,  // could be { "$gt": "" }
    password: req.body.password,
  });
  if (user) res.json({ token: sign(user) });
});
```

## Root Cause
MongoDB operators are embedded in the same namespace as field values. When a request body is parsed as JSON and spliced into a query object, object-valued fields become operator expressions.

## Fix
```javascript
app.post('/login', async (req, res) => {
  // Coerce to string — objects cannot be operator-injected after toString
  const username = String(req.body.username || '');
  const password = String(req.body.password || '');

  if (!username || !password) return res.status(400).json({ error: 'Missing credentials' });

  const user = await User.findOne({ username, password });
  if (user) res.json({ token: sign(user) });
  else res.status(401).json({ error: 'Invalid credentials' });
});
```

Or use `express-mongo-sanitize` middleware to strip `$` and `.` from all request inputs.

## Test
```javascript
test('login with operator payload does not bypass auth', async () => {
  const res = await request(app)
    .post('/login')
    .send({ username: { $gt: '' }, password: { $gt: '' } });
  // Must not return a token
  expect(res.status).not.toBe(200);
  expect(res.body.token).toBeUndefined();
});
```

## Detection
```
findOne\(\s*\{\s*\w+\s*:\s*req\.body\.\w+
find\(\s*\{\s*\w+\s*:\s*req\.(?:body|query|params)\.\w+
updateOne\(\s*\{\s*\w+\s*:\s*req\.body\.\w+
# MongoDB query with user-controlled value not coerced to string/number
```
