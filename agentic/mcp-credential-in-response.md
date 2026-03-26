---
id: agentic-mcp-credential-in-response
domain: agentic
severity: critical
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# MCP Credential Leakage in Tool Response

## Problem
An MCP (Model Context Protocol) tool handler returns raw objects that include credentials, tokens, or secrets — which the LLM then echoes verbatim in its response to the user.

```javascript
// WRONG — leaks the API key the tool was configured with
server.tool('get_user', async (args) => {
  const data = await db.getUser(args.userId);
  return { content: [{ type: 'text', text: JSON.stringify(data) }] };
  // data may contain: { id, email, hashedPassword, internalToken }
});
```

## Root Cause
Tool authors return full database rows or configuration objects without stripping sensitive fields, trusting the LLM not to repeat them. The LLM has no way to distinguish safe from sensitive fields.

## Fix
```javascript
server.tool('get_user', async (args) => {
  const data = await db.getUser(args.userId);
  // Allowlist safe fields only
  const safe = { id: data.id, name: data.name, email: data.email };
  return { content: [{ type: 'text', text: JSON.stringify(safe) }] };
});
```

Never return raw DB rows, config objects, or HTTP response headers from tool handlers.

## Test
```javascript
test('get_user tool does not leak internalToken', async () => {
  mockDb.getUser.resolves({ id: 1, name: 'Alice', internalToken: 'secret' });
  const result = await toolHandler('get_user', { userId: 1 });
  const text = result.content[0].text;
  expect(text).not.toContain('internalToken');
  expect(text).not.toContain('secret');
});
```

## Detection
```
JSON\.stringify\((?:data|row|result|record|user|config)\)
return\s+\{\s*content.*text.*JSON\.stringify
\.tool\([^)]+,\s*async.*return.*JSON\.stringify
```
