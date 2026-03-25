# tdd-patterns

Institutional security memory for the TDD Remediation Protocol. Each file documents a real vulnerability class, its root cause, the correct fix, and the exploit test that proves the hole is closed.

Pulled automatically by `tdd-remediation` at the start of every audit run. New patterns are contributed back via PR at the end of each run.

## Structure

```
auth/        — Authentication and authorization failures (IDOR, JWT, auth chain)
injection/   — SQLi, XSS, CMDi, path traversal, SSRF, open redirect, NoSQL
secrets/     — Hardcoded credentials, env fallbacks, secret history
frontend/    — Security headers, CSP, environment URLs, deployment config
agentic/     — AI-specific vulnerabilities (prompt injection, MCP, GitHub Actions)
deps/        — Lockfile sync, tsconfig exclusion, unpinned dependencies
infra/       — Rate limiting, email deliverability, CORS
```

## Pattern Format

Every file includes:
- `id` — unique slug (`domain-short-name`)
- `domain` — category
- `severity` — critical / high / medium / low
- `stack` — relevant technologies (`*` = all stacks)
- `date_added` and `project` — where this was first found
- **Problem** — what the vulnerability looks like
- **Root Cause** — why it happens (especially in AI-generated code)
- **Fix** — the correct implementation
- **Test** — the Red-phase exploit test that must fail before patching
- **Detection** — grep/query to find this in a new codebase

## TDD Principle

Every pattern in this repo was proven closed with a failing test before the fix was applied. The test is the contract. The fix without the test is just hope.

```
Red  → write the exploit test. It must FAIL.
Green → apply the fix. The test must PASS.
Refactor → run the full suite. Zero regressions.
```

## Contributing

Patterns are added via PR from `tdd-remediation` at the end of each audit run. To add manually, follow the frontmatter format in any existing file.

## Remote

`github.com/lhi/tdd-patterns`
