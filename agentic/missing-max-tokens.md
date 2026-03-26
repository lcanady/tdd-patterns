---
id: agentic-missing-max-tokens
domain: agentic
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Missing max_tokens Limit

## Problem
LLM API calls omit the `max_tokens` (or `max_completion_tokens`) parameter, allowing the model to return arbitrarily long completions. An adversarial prompt can trigger multi-thousand-token responses on every request, leading to unbounded cost and potential denial-of-wallet.

```javascript
// WRONG — no max_tokens cap
const res = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages,
});
```

## Root Cause
Quick prototypes and AI-generated scaffolding omit the limit because it works fine in testing with short prompts. In production, a jailbreak or user-crafted prompt can force maximal output.

## Fix
```javascript
const MAX_TOKENS = 2048; // tune per use-case

const res = await openai.chat.completions.create({
  model:      'gpt-4o',
  messages,
  max_tokens: MAX_TOKENS,
});
```

Enforce a sane upper bound and add cost monitoring alerts.

## Test
```javascript
test('chat call always sets max_tokens', async () => {
  const spy = jest.spyOn(openai.chat.completions, 'create');
  await callMyLLM('Hello');
  expect(spy).toHaveBeenCalledWith(
    expect.objectContaining({ max_tokens: expect.any(Number) })
  );
});
```

## Detection
```
openai\.chat\.completions\.create\(\s*\{(?![\s\S]{0,500}max_tokens)
anthropic\.messages\.create\(\s*\{(?![\s\S]{0,500}max_tokens)
completion\s*=\s*client\.complete\(\s*\{(?![\s\S]{0,500}max_tokens)
```
