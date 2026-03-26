---
id: injection-prototype-pollution-bracket
domain: injection
severity: high
stack: "node.js, javascript"
date_added: 2026-03-26
project: tdd-audit
---

# Prototype Pollution via Bracket Notation

## Problem
User-controlled keys are used to assign object properties via bracket notation without checking for `__proto__`, `constructor`, or `prototype`. Polluting `Object.prototype` affects every object in the process, enabling privilege escalation or authentication bypass.

```javascript
// WRONG
function merge(target, source) {
  for (const key of Object.keys(source)) {
    target[key] = source[key];   // key could be '__proto__'
  }
}
merge({}, JSON.parse(req.body));
// After: ({}).isAdmin === true  ← all objects now have isAdmin
```

## Root Cause
Generic merge/deep-copy utilities and JSON-to-object assignment loops don't filter prototype-chain keys. Nested payloads like `{"__proto__": {"isAdmin": true}}` traverse through the loop unchecked.

## Fix
```javascript
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') continue;
    if (typeof source[key] === 'object' && source[key] !== null) {
      target[key] = safeMerge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
```

Or use `Object.create(null)` for dictionaries and `structuredClone()` for deep copies.

## Test
```javascript
test('merge does not pollute Object.prototype', () => {
  const payload = JSON.parse('{"__proto__":{"isAdmin":true}}');
  safeMerge({}, payload);
  expect(({}).isAdmin).toBeUndefined();
});
```

## Detection
```
\w+\[(?:key|prop|field|param|input)\]\s*=
target\[(?:key|k)\]\s*=\s*source\[(?:key|k)\]
obj\[userInput\]
```
