---
id: secrets-hardcoded-ai-api-keys
domain: secrets
severity: critical
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# Hardcoded AI Provider API Keys

## Problem
OpenAI, Anthropic, Gemini, Cohere, Mistral, and HuggingFace API keys are committed directly into source code. An attacker with repo access can immediately make API calls billed to the victim, exfiltrate model context, or abuse the provider's APIs at scale. Provider keys carry no IP restriction by default.

```javascript
// WRONG
const client = new OpenAI({ apiKey: 'sk-proj-AbCdEf...' });
const anthropic = new Anthropic({ apiKey: 'sk-ant-api03-...' });
```

## Root Cause
AI-generated scaffolding and tutorial snippets hardcode keys for quick demo runs. Developers copy-paste into production without moving the credential to an env var, and code review misses provider-specific prefix patterns.

## Fix
```javascript
function requireEnv(name) {
  const val = process.env[name];
  if (!val) throw new Error(`Missing required env var: ${name}`);
  return val;
}
const openai = new OpenAI({ apiKey: requireEnv('OPENAI_API_KEY') });
```
If a key was already committed: rotate immediately via provider console, then add to `.gitleaks.toml` and run `gitleaks detect --source .` on git history.

## Test
```javascript
test('throws at startup when OPENAI_API_KEY is missing', () => {
  const saved = process.env.OPENAI_API_KEY;
  delete process.env.OPENAI_API_KEY;
  expect(() => require('../lib/ai-client')).toThrow('Missing required env var');
  process.env.OPENAI_API_KEY = saved;
});
```

## Detection
```
sk-proj-[A-Za-z0-9_\-]{20,}          # OpenAI project key
sk-ant-api03-[A-Za-z0-9_\-]{20,}     # Anthropic key
AIza[A-Za-z0-9_\-]{35}               # Google Gemini / GCP key
hf_[A-Za-z0-9]{30,}                  # HuggingFace token
cohere.*['"][A-Za-z0-9]{40}['"]       # Cohere key
mistral.*['"][A-Za-z0-9]{32}['"]      # Mistral key
```
