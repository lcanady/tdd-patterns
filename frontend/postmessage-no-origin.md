---
id: frontend-postmessage-no-origin
domain: frontend
severity: high
stack: "javascript"
date_added: 2026-03-26
project: tdd-audit
---

# postMessage Without Origin Validation

## Problem
`window.addEventListener('message', handler)` processes messages without checking `event.origin`. Any window (including attacker-controlled iframes or tabs) can post arbitrary data that the handler treats as trusted.

```javascript
// WRONG
window.addEventListener('message', (event) => {
  const { action, data } = event.data;
  if (action === 'navigate') window.location.href = data; // open redirect / XSS
});
```

## Root Cause
Developers use `postMessage` for cross-frame communication and focus on the message structure, not the sender. MDN examples historically omitted origin checks.

## Fix
```javascript
const ALLOWED_ORIGIN = 'https://parent.example.com';

window.addEventListener('message', (event) => {
  if (event.origin !== ALLOWED_ORIGIN) return; // reject all other origins
  const { action, data } = event.data;
  if (action === 'navigate' && isAllowedUrl(data)) {
    window.location.href = data;
  }
});
```

Never use `event.origin === '*'` as an allowlist condition.

## Test
```javascript
test('ignores messages from unknown origins', () => {
  const navigateSpy = jest.spyOn(window, 'location', 'get');
  const event = new MessageEvent('message', {
    data:   { action: 'navigate', data: 'https://evil.com' },
    origin: 'https://evil.com',
  });
  window.dispatchEvent(event);
  expect(navigateSpy).not.toHaveBeenCalled();
});
```

## Detection
```
addEventListener\(['"]message['"]
window\.on(?:message)
# absence of event\.origin check inside handler
```
