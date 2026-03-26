---
id: injection-host-header-injection
domain: injection
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Host Header Injection

## Problem
`req.headers.host` is used to construct password-reset links, redirects, or canonical URLs. An attacker sends `Host: evil.com` and the application generates `https://evil.com/reset?token=…`, directing victims to an attacker-controlled domain.

```javascript
// WRONG
app.post('/forgot-password', async (req, res) => {
  const resetUrl = `https://${req.headers.host}/reset?token=${token}`;
  await sendEmail(user.email, `Reset your password: ${resetUrl}`);
});
```

## Root Cause
The `Host` header is user-controlled; load balancers and proxies may not strip it. Developers copy patterns from tutorials that assume a trusted reverse proxy normalises the host.

## Fix
```javascript
// Hardcode the application origin — never derive from request headers
const APP_ORIGIN = process.env.APP_ORIGIN; // e.g. 'https://app.example.com'
if (!APP_ORIGIN) throw new Error('Missing APP_ORIGIN env var');

app.post('/forgot-password', async (req, res) => {
  const resetUrl = `${APP_ORIGIN}/reset?token=${token}`;
  await sendEmail(user.email, `Reset your password: ${resetUrl}`);
});
```

## Test
```javascript
test('password reset link uses configured origin, not Host header', async () => {
  process.env.APP_ORIGIN = 'https://app.example.com';
  const emailSpy = jest.spyOn(emailService, 'send');
  await request(app)
    .post('/forgot-password')
    .set('Host', 'evil.com')
    .send({ email: 'alice@example.com' });
  expect(emailSpy.mock.calls[0][1]).toContain('https://app.example.com');
  expect(emailSpy.mock.calls[0][1]).not.toContain('evil.com');
});
```

## Detection
```
req\.headers\.host
request\.headers\[['"]host['"]\]
req\.get\(['"]host['"]\)
# used in redirect, url construction, or email body
```
