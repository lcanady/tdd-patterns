---
id: auth-idor-missing-ownership
domain: auth
severity: critical
stack: "*"
date_added: 2026-03-25
project: general
---

# IDOR — Missing Ownership Check

## Problem
A route looks up a resource by ID from the request (params, query, body) without verifying the requesting user owns that resource. Any authenticated user can read or modify any other user's data by guessing or iterating IDs.

## Root Cause
AI-generated code implements authentication (is the user logged in?) but not authorization (does this user own this record?). The pattern looks correct at a glance:

```typescript
// WRONG — authenticated, but no ownership check
const user = await getUser(req)
if (!user) return res.status(401).json({ error: 'Unauthorized' })

// Any logged-in user can access any record by changing the ID
const document = await db.findById(req.params.id)
return res.json(document)
```

## Fix

Scope every lookup to the authenticated user's ID:

```typescript
// Option 1: Add WHERE clause
const document = await db.query(
  'SELECT * FROM documents WHERE id = $1 AND owner_id = $2',
  [req.params.id, user.id]
)
if (!document) return res.status(404).json({ error: 'Not found' })

// Option 2: Fetch then check (use only when you need the record regardless)
const document = await db.findById(req.params.id)
if (!document || document.owner_id !== user.id) {
  return res.status(403).json({ error: 'Forbidden' })
}
```

Return `403 Forbidden` (not `404`) when the resource exists but the user doesn't own it, unless your threat model requires obscuring existence.

## Test

```typescript
// Red: must FAIL — user A can read user B's resource before the fix

it('returns 403 when user accesses another user resource', async () => {
  // userA creates a document
  const { id } = await createDocumentAs(userA)

  // userB tries to access it
  const res = await authedFetch(userB.token, 'GET', `/api/documents/${id}`)
  expect(res.status).toBe(403)
})

it('returns 200 when user accesses their own resource', async () => {
  const { id } = await createDocumentAs(userA)
  const res = await authedFetch(userA.token, 'GET', `/api/documents/${id}`)
  expect(res.status).toBe(200)
})
```

## Detection

```bash
# findById/findOne keyed only to request param — no user scope
grep -rn "findById(req\.\|findOne({.*id:.*req\.\|\.eq('id', params\." src/

# Routes that auth but never reference user.id or owner_id in the query
grep -rn "getUser\|auth\.uid" src/app/api/ -l \
  | xargs grep -L "owner_id\|user_id\|host_id\|user\.id"
```
