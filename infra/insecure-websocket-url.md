---
id: infra-insecure-websocket-url
domain: infra
severity: medium
stack: "node.js, javascript"
date_added: 2026-03-26
project: tdd-audit
---

# Insecure WebSocket URL (ws:// instead of wss://)

## Problem
WebSocket connections are opened with the plaintext `ws://` scheme. Traffic is unencrypted, allowing network-level attackers to intercept messages, inject frames, or hijack sessions.

```javascript
// WRONG
const socket = new WebSocket('ws://api.example.com/stream');

// Also wrong — constructing from window.location without upgrading
const socket = new WebSocket(`ws://${window.location.host}/ws`);
```

## Root Cause
Developers use `ws://` during local development and forget to switch to `wss://` for production. Environment-conditional logic is absent or the protocol is hardcoded.

## Fix
```javascript
// Derive from current page protocol (works for both dev and prod)
const proto  = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
const socket = new WebSocket(`${proto}//${window.location.host}/ws`);

// Or hardcode wss:// for production builds
const WS_URL = process.env.WS_URL; // e.g. 'wss://api.example.com/stream'
const socket = new WebSocket(WS_URL);
```

## Test
```javascript
test('WebSocket URL uses wss:// in production', () => {
  process.env.WS_URL = 'wss://api.example.com/stream';
  const url = getWebSocketUrl();
  expect(url.startsWith('wss://')).toBe(true);
});
```

## Detection
```
new WebSocket\(['"]ws://
WebSocket\(['"]ws://
WebSocket\(`ws://
ws://[a-zA-Z0-9]
```
