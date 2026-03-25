---
id: infra-transactional-vs-bulk-headers
domain: infra
severity: medium
stack: "nodemailer, aws-ses, sendgrid, *"
date_added: 2026-03-25
project: beltway-events
---

# Transactional Email Headers Must Not Include `Precedence: bulk`

## Problem
Applying `Precedence: bulk` to transactional emails (signup confirmation, password reset, RSVP confirmation) causes Gmail and Outlook to classify them as bulk mail and route them to spam.

## Root Cause
A single `sendEmail()` utility was configured with `Precedence: bulk` in its default headers, applied to all outgoing mail regardless of type. This is correct for newsletters but harmful for transactional mail.

## Fix

Remove bulk headers from the default mailer. Apply them only at the call site for newsletter/digest sends:

```typescript
// lib/mailer.ts — transactional-safe defaults only
async function sendEmail({ to, subject, html, headers = {} }) {
  await transporter.sendMail({
    from: FROM_ADDRESS,
    to, subject, html,
    headers: {
      'X-Auto-Response-Suppress': 'OOF, AutoReply',
      ...headers,  // caller adds bulk headers when needed
    },
  })
}
```

```typescript
// Newsletter/digest send — add bulk headers per-call
await sendEmail({
  to: subscriber.email,
  subject: 'Weekly Digest',
  html,
  headers: {
    'Precedence': 'bulk',
    'List-Unsubscribe': `<${unsubscribeUrl}>`,
    'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
  },
})
```

## Rule

| Email type | `Precedence: bulk` | `List-Unsubscribe` |
|---|---|---|
| Signup confirmation | ❌ | ❌ |
| Password reset | ❌ | ❌ |
| RSVP / booking confirmation | ❌ | ❌ |
| Transactional notification | ❌ | ❌ |
| Newsletter / weekly digest | ✅ | ✅ |

## Test

```typescript
// Red: must FAIL — bulk header present on transactional email before the fix

it('confirmation email does not include Precedence: bulk', async () => {
  const captured = await captureEmail(() => sendConfirmationEmail({
    to: 'user@example.com',
    token: 'abc123',
  }))
  expect(captured.headers?.['Precedence']).toBeUndefined()
})

it('newsletter email includes Precedence: bulk', async () => {
  const captured = await captureEmail(() => sendDigestEmail({
    to: 'subscriber@example.com',
    content: 'This week...',
    unsubscribeUrl: 'https://app.com/unsubscribe?token=xyz',
  }))
  expect(captured.headers?.['Precedence']).toBe('bulk')
  expect(captured.headers?.['List-Unsubscribe']).toBeTruthy()
})
```

## Detection

```bash
# Default headers in mailer containing Precedence: bulk
grep -rn "Precedence.*bulk" src/lib/mailer.ts src/lib/email*

# sendEmail calls that don't explicitly set Precedence
# (bulk headers should only appear in newsletter/digest call sites)
grep -rn "Precedence.*bulk" src/ | grep -v "newsletter\|digest\|weekly"
```
