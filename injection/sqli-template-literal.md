---
id: injection-sqli-template-literal
domain: injection
severity: critical
stack: "node.js, python, go, *"
date_added: 2026-03-25
project: general
---

# SQL Injection via Template Literal / String Concatenation

## Problem
User-controlled input is interpolated directly into a SQL query string. An attacker can inject SQL syntax to read, modify, or delete arbitrary data, bypass authentication, or in some configs execute OS commands.

## Root Cause
Template literals and string concatenation feel natural when building dynamic queries. AI-generated code frequently produces this pattern, especially when adding search/filter features:

```typescript
// WRONG — classic template literal injection
const results = await db.query(`
  SELECT * FROM users WHERE email = '${req.body.email}'
`)

// WRONG — string concatenation (Python)
cursor.execute("SELECT * FROM items WHERE name = '" + request.args['name'] + "'")
```

Input `' OR '1'='1` makes the WHERE clause always true — returns all rows. Input `'; DROP TABLE users; --` destroys data.

## Fix

Use parameterized queries / prepared statements in every stack:

```typescript
// Node.js / pg
const results = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [req.body.email]
)

// Prisma
const user = await prisma.user.findFirst({
  where: { email: req.body.email }
})

// Drizzle
const user = await db.select().from(users).where(eq(users.email, req.body.email))
```

```python
# Python / psycopg2
cursor.execute('SELECT * FROM items WHERE name = %s', (request.args['name'],))

# SQLAlchemy
stmt = select(Item).where(Item.name == request.args['name'])
```

```go
// Go / database/sql
row := db.QueryRow("SELECT * FROM users WHERE email = ?", r.FormValue("email"))
```

**Never** use `raw()` / `$queryRaw` with user input in Prisma. Use `$queryRawUnsafe` only with pre-validated, allowlisted values.

## Test

```typescript
// Red: must FAIL — injection succeeds before the fix

it('rejects SQL injection in email field', async () => {
  const res = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify({ email: "' OR '1'='1", password: 'anything' })
  })
  // Should NOT return 200 with a user record
  expect(res.status).not.toBe(200)
  const body = await res.json()
  expect(body.user).toBeUndefined()
})

it('handles special characters in search without error', async () => {
  const res = await fetch("/api/search?q=O'Brien")
  expect(res.status).toBe(200) // should work, not 500
})
```

## Detection

```bash
# Template literal SQL (JS/TS)
grep -rn 'SELECT.*\${' src/
grep -rn 'INSERT.*\${' src/
grep -rn 'UPDATE.*\${' src/
grep -rn 'DELETE.*\${' src/

# String concatenation SQL (Python)
grep -rn 'execute.*".*".*+' .
grep -rn "execute(f'" .

# Prisma unsafe
grep -rn 'queryRawUnsafe\|$executeRawUnsafe' src/
```
