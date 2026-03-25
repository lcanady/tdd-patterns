---
id: injection-command-injection
domain: injection
severity: critical
stack: "node.js, python, go, *"
date_added: 2026-03-25
project: general
---

# Command Injection via exec / shell=True

## Problem
User-controlled input is passed to a shell execution function. An attacker appends shell metacharacters (`;`, `&&`, `|`, `$()`) to run arbitrary OS commands — reading `/etc/passwd`, exfiltrating secrets, or establishing persistence.

## Root Cause
Developers use `exec()` / `execSync()` / `subprocess.run(shell=True)` because it's easy to build complex commands as strings. AI-generated code frequently generates this when implementing features like file conversion, image processing, or report generation.

```typescript
// WRONG — Node.js
exec(`convert ${req.body.filename} output.pdf`)
// Input: `; cat /etc/passwd > /tmp/leak &` — executes as shell command

// WRONG — Python
subprocess.run(f"ffmpeg -i {filename} output.mp4", shell=True)
```

## Fix

**Node.js**: Use `execFile` (no shell interpolation) or `spawn` with an args array:

```typescript
import { execFile } from 'child_process'
import path from 'path'

// Validate filename is safe before using it
const safeName = path.basename(req.body.filename).replace(/[^a-zA-Z0-9._-]/g, '')
if (!safeName) return res.status(400).json({ error: 'Invalid filename' })

execFile('convert', [safeName, 'output.pdf'], (err, stdout) => {
  if (err) return res.status(500).json({ error: 'Conversion failed' })
  res.json({ ok: true })
})
```

**Python**: Pass args as a list, `shell=False` (default):
```python
import subprocess, shlex, re

# Validate first
if not re.match(r'^[\w\-. ]+$', filename):
    abort(400)

subprocess.run(['ffmpeg', '-i', filename, 'output.mp4'], check=True)
# NOT: subprocess.run(f"ffmpeg -i {filename} output.mp4", shell=True)
```

**Go**: Use `exec.Command` with separate args (never `exec.Command("sh", "-c", userInput)`):
```go
cmd := exec.Command("convert", filename, "output.pdf")
```

## Test

```typescript
// Red: must FAIL — injection executes before the fix

it('rejects shell metacharacters in filename', async () => {
  const res = await fetch('/api/convert', {
    method: 'POST',
    body: JSON.stringify({ filename: 'file.jpg; cat /etc/passwd' })
  })
  expect(res.status).toBe(400)
})

it('does not execute injected commands', async () => {
  const probe = `/tmp/inject-${Date.now()}`
  await fetch('/api/convert', {
    method: 'POST',
    body: JSON.stringify({ filename: `x; touch ${probe}` })
  })
  // File must not have been created
  expect(existsSync(probe)).toBe(false)
})
```

## Detection

```bash
# Node.js exec with user input
grep -rn "exec(.*req\.\|execSync(.*req\." src/
grep -rn "child_process" src/ -l

# Python shell=True
grep -rn "shell=True" .

# Go sh -c with variable
grep -rn '"sh".*"-c"' .
```
