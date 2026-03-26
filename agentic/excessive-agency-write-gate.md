---
id: agentic-excessive-agency-write-gate
domain: agentic
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Excessive Agency — Write Actions Without Human Confirmation

## Problem
An LLM agent can invoke write tools (file system writes, database mutations, email sends, API POST/DELETE calls) without requiring human approval. A single adversarial prompt can trigger destructive or irreversible actions autonomously.

```javascript
// WRONG — write tool registered with no confirmation gate
server.tool('delete_records', async ({ table, where }) => {
  await db.delete(table, where);
  return { content: [{ type: 'text', text: 'Deleted.' }] };
});
```

## Root Cause
Developers focus on capability and trust the LLM's judgement for destructive actions. The model cannot distinguish a legitimate request from a prompt-injection attack.

## Fix
```javascript
// Safe — require human confirmation for write/delete actions
server.tool('delete_records', {
  description: 'Delete database records. REQUIRES human confirmation.',
  requiresConfirmation: true,   // MCP UI gate
}, async ({ table, where }, { confirmed }) => {
  if (!confirmed) {
    return { content: [{ type: 'text', text: 'Awaiting human approval.' }] };
  }
  await db.delete(table, where);
  return { content: [{ type: 'text', text: 'Deleted.' }] };
});
```

For non-MCP agents, implement a confirmation callback or human-in-the-loop webhook before any irreversible action.

## Test
```javascript
test('delete_records does nothing without confirmed=true', async () => {
  const spy = jest.spyOn(db, 'delete');
  await toolHandler('delete_records', { table: 'users', where: '1=1' }, { confirmed: false });
  expect(spy).not.toHaveBeenCalled();
});
```

## Detection
```
server\.tool\(['"](?:delete|remove|drop|write|send|post|patch|update)[^'"]*['"]
\.tool\([^)]+\)\s*\{[\s\S]{0,300}(?:fs\.write|db\.delete|fetch.*method.*DELETE)
# write-capable tools without requiresConfirmation or confirmed check
```
