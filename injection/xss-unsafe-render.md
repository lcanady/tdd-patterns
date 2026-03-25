---
id: injection-xss-unsafe-render
domain: injection
severity: high
stack: "react, next.js, express, *"
date_added: 2026-03-25
project: general
---

# XSS via Unsafe HTML Rendering

## Problem
User-controlled content is rendered as raw HTML without sanitization. An attacker injects `<script>` tags, event handlers, or data: URIs that execute in victims' browsers — stealing sessions, redirecting users, or logging keystrokes.

## Root Cause
React's `dangerouslySetInnerHTML`, the DOM's `innerHTML =`, Express's `res.send()` with reflected input, and Flask's `render_template_string()` all bypass output encoding when user content is passed directly.

```tsx
// WRONG — React
<div dangerouslySetInnerHTML={{ __html: user.bio }} />

// WRONG — DOM
document.getElementById('output').innerHTML = searchParams.get('q')

// WRONG — Express reflecting request data
app.get('/search', (req, res) => res.send(`Results for: ${req.query.q}`))
```

## Fix

**React**: Never use `dangerouslySetInnerHTML` with user content. For rich text that genuinely needs HTML, sanitize with DOMPurify first:

```tsx
import DOMPurify from 'isomorphic-dompurify'

// Only when HTML rendering is truly required
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(user.bio) }} />

// Prefer plain text rendering — React escapes by default
<p>{user.bio}</p>
```

**DOM**: Use `textContent`, not `innerHTML`:
```javascript
document.getElementById('output').textContent = searchParams.get('q') // safe
```

**Express**: Escape output or use a template engine with auto-escaping (Handlebars, Pug, EJS by default):
```typescript
import escapeHtml from 'escape-html'
res.send(`Results for: ${escapeHtml(req.query.q)}`)
```

**CSP as defense-in-depth**: Add `Content-Security-Policy: default-src 'self'` to all responses. This limits damage if an XSS slip occurs.

## Test

```typescript
// Red: must FAIL — XSS executes before the fix

it('does not render script tags from user bio', async () => {
  await createUserWithBio('<script>window.__xss=1</script>Hello')
  const res = await fetch('/api/users/profile')
  const html = await res.text()
  // Must not contain raw <script> — must be escaped or stripped
  expect(html).not.toContain('<script>')
})

it('renders user bio as plain text, not HTML', () => {
  render(<UserBio bio='<img onerror="alert(1)" src=x>' />)
  // Element must not contain an img tag
  expect(document.querySelector('img')).toBeNull()
})
```

## Detection

```bash
# React unsafe HTML
grep -rn "dangerouslySetInnerHTML" src/

# DOM innerHTML writes
grep -rn "innerHTML\s*=" src/

# Express reflecting request data directly
grep -rn "res\.send(.*req\." src/
grep -rn "res\.send(.*req\.query\|res\.send(.*req\.body\|res\.send(.*req\.params" src/

# Flask dynamic template with user input
grep -rn "render_template_string" .
```
