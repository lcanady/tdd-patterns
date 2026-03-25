---
id: auth-auth-chain
domain: auth
severity: critical
stack: "*"
date_added: 2026-03-25
project: beltway-events
---

# Broken API Route Auth Chain

## Problem
API routes with missing, skipped, or out-of-order authentication and authorization checks allow unauthenticated users to read private data, and authenticated-but-unauthorized users to mutate resources they don't own.

## Root Cause
AI-generated route handlers frequently implement auth as an afterthought or collapse authentication and authorization into a single weak check. Common patterns:

- Route calls the DB before checking if the user is logged in
- Auth check present but authorization (ownership) check missing — any logged-in user can access any record
- Admin check queries the DB (`profiles.is_admin`) instead of reading the JWT claim — can be bypassed if the DB is compromised or if RLS is broken

## Fix

Every mutating API route must follow this exact sequence:

```
authenticate → authorize → validate → execute → respond
```

```typescript
export async function POST(request: NextRequest, { params }) {
  // 1. Authenticate — is someone logged in?
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  // 2. Authorize — do they own this resource?
  const { data: resource } = await db.from('items').select('owner_id').eq('id', params.id).single()
  if (!resource || resource.owner_id !== user.id) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  // 3. Validate — is the input well-formed?
  const { title } = await request.json()
  if (!title?.trim()) return NextResponse.json({ error: 'Title required' }, { status: 400 })

  // 4. Execute
  await db.from('items').update({ title }).eq('id', params.id)

  // 5. Respond — never expose stack traces or internal error details
  return NextResponse.json({ ok: true })
}
```

**Admin check: always JWT, never DB**
```typescript
// Correct — JWT claim is authoritative and unforgeable
const isAdmin = user.user_metadata?.is_admin === true

// Wrong — DB column is not authoritative; breaks if RLS is misconfigured
const { data: profile } = await db.from('profiles').select('is_admin').eq('id', user.id).single()
```

## Test

```typescript
// Red: these must FAIL before the fix is applied

it('returns 401 for unauthenticated POST', async () => {
  const res = await fetch('/api/items/123', { method: 'POST', body: JSON.stringify({ title: 'x' }) })
  expect(res.status).toBe(401)
})

it('returns 403 when authenticated user does not own the resource', async () => {
  const res = await authedFetch(otherUsersToken, 'POST', '/api/items/123', { title: 'x' })
  expect(res.status).toBe(403)
})

it('returns 200 when owner updates their own resource', async () => {
  const res = await authedFetch(ownerToken, 'POST', '/api/items/123', { title: 'updated' })
  expect(res.status).toBe(200)
})
```

## Detection

```bash
# Find route handlers without an auth check
grep -rn "export async function POST\|export async function PUT\|export async function DELETE" src/app/api/ \
  | grep -v "getUser\|auth\.uid\|authenticate"

# Find routes that check auth but skip authorization (no ownership check)
grep -rn "getUser" src/app/api/ -l | xargs grep -L "owner_id\|user_id\|host_id\|is_admin"
```
