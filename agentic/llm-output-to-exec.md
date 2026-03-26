---
id: agentic-llm-output-to-exec
domain: agentic
severity: critical
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# LLM Output to exec / eval

## Problem
Raw text from an LLM completion is passed to `exec()`, `execSync()`, `eval()`, `spawn()`, or `Function()` without validation. If the model is jailbroken, fine-tuned adversarially, or the API response is intercepted, the attacker achieves remote code execution on the server.

```javascript
// WRONG
const result = await openai.chat.completions.create({ ... });
exec(result.choices[0].message.content); // RCE
```

## Root Cause
Developers trust that the model will only produce "safe" commands and use exec() as a convenient way to run model-suggested operations. AI code generation frequently produces this pattern when implementing shell-automation features.

## Fix
```javascript
const ALLOWED_COMMANDS = new Set(['ls', 'pwd', 'echo']);
function safeExec(llmSuggested) {
  const cmd = String(llmSuggested).trim().split(/\s+/)[0];
  if (!ALLOWED_COMMANDS.has(cmd)) throw new Error(`Blocked command: ${cmd}`);
  return execFile(cmd, [], { timeout: 5000 });
}
```
Never eval or exec raw LLM output. Use structured output schemas and parse them instead.

## Test
```javascript
test('does not execute LLM-suggested command directly', async () => {
  mockLLM.returns('rm -rf /');
  await expect(runAiSuggestion('clean project')).rejects.toThrow(/Blocked/);
});
```

## Detection
```
exec\([^)]*(?:response|result|output|completion|generated|llmResult|aiResult)
execSync\([^)]*(?:response|result|output|completion)
eval\(\s*(?:await\s+)?(?:response|result|completion)
spawn\([^)]*(?:response|result|output)
```
