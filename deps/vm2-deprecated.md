---
id: deps-vm2-deprecated
domain: deps
severity: critical
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# vm2 Deprecated — Known Sandbox Escapes

## Problem
`vm2` is used to sandbox untrusted JavaScript. The library has been abandoned (last release 2023-06-06) and has multiple publicly-known sandbox escapes (CVE-2023-29199, CVE-2023-30547, CVE-2023-32314) that allow complete host RCE from inside the sandbox.

```javascript
// WRONG — vm2 is unsafe and unmaintained
const { VM } = require('vm2');
const vm = new VM({ sandbox: {} });
vm.run(userCode);  // attackers escape with known exploits
```

## Root Cause
`vm2` was the de-facto Node.js sandbox for years. Projects that adopted it haven't migrated since maintainers archived it. The CVEs are trivially exploitable with public PoCs.

## Fix
Migrate to a maintained alternative:

```javascript
// Option 1: isolated-vm (V8 isolate — true memory isolation)
const ivm = require('isolated-vm');
const isolate = new ivm.Isolate({ memoryLimit: 128 });
const context = await isolate.createContext();
await context.eval(userCode);

// Option 2: Node.js worker_threads with a permissions model (no npm dep)
// Option 3: WASM sandbox if code is portable
// Option 4: subprocess with seccomp / container isolation
```

Regardless of sandbox library, validate/lint untrusted code before execution.

## Test
```javascript
test('does not import vm2', () => {
  // Scan package.json and source for vm2 usage
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  expect(Object.keys(pkg.dependencies || {})).not.toContain('vm2');
  expect(Object.keys(pkg.devDependencies || {})).not.toContain('vm2');
});
```

## Detection
```
require\(['"]vm2['"]\)
from\s+['"]vm2['"]
import\s+.*from\s+['"]vm2['"]
# package.json: "vm2":
```
