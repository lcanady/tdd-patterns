---
id: agentic-agent-unbounded-loop
domain: agentic
severity: high
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Agent Unbounded Loop

## Problem
Agentic loops that call tools repeatedly have no maximum iteration cap. A model stuck in a reasoning loop, or deliberately manipulated, can spin forever — exhausting API credits, CPU, and time until an out-of-memory or timeout kills the process.

```javascript
// WRONG — no iteration limit
while (true) {
  const { stop, toolCalls } = await agent.step(messages);
  if (stop) break;
  messages.push(...await executeTools(toolCalls));
}
```

## Root Cause
Tutorial examples use `while (true)` for simplicity. Developers assume the model will always reach a stop condition. In practice, malformed tool responses or adversarial prompts can prevent termination.

## Fix
```javascript
const MAX_ITERATIONS = 20;

for (let i = 0; i < MAX_ITERATIONS; i++) {
  const { stop, toolCalls } = await agent.step(messages);
  if (stop) break;
  messages.push(...await executeTools(toolCalls));
  if (i === MAX_ITERATIONS - 1) {
    throw new Error(`Agent exceeded ${MAX_ITERATIONS} iterations — aborting`);
  }
}
```

Also enforce a wall-clock timeout alongside the step limit.

## Test
```javascript
test('agent stops after MAX_ITERATIONS with an error', async () => {
  // stub step to never return stop=true
  mockAgent.step.resolves({ stop: false, toolCalls: [] });
  await expect(runAgent('loop forever')).rejects.toThrow(/exceeded.*iterations/i);
  expect(mockAgent.step).toHaveBeenCalledTimes(20);
});
```

## Detection
```
while\s*\(\s*true\s*\)\s*\{[\s\S]{0,200}agent
for\s*\(.*;;\)\s*\{[\s\S]{0,200}toolCall
runAgent|agentLoop|agentRun[\s\S]{0,300}while\s*\(true\)
```
