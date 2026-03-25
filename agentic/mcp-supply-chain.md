---
id: agentic-mcp-supply-chain
domain: agentic
severity: high
stack: "claude, mcp, node.js"
date_added: 2026-03-25
project: general
---

# MCP Server Supply Chain Risk — Unpinned npx (ASI03)

## Problem
MCP servers installed via `npx` without a pinned version execute whatever version npm resolves at runtime. A compromised or typosquatted package executes with the agent's full tool permissions — filesystem access, shell exec, network access — in the agent's context.

```json
// WRONG — resolves latest at runtime
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-filesystem", "/"]
    }
  }
}
```

If `@modelcontextprotocol/server-filesystem` is compromised or a typosquatted package name is used, malicious code runs immediately with all the permissions the MCP server was granted.

## Root Cause
`npx <package>` resolves and runs the latest published version unless pinned. Claude Desktop / Claude Code docs show this pattern because it's convenient, but it's a supply chain risk in production configurations.

## Fix

**Option 1 — Pin to exact version with integrity check:**
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-filesystem@1.2.3", "/projects"]
    }
  }
}
```

**Option 2 — Install locally and reference directly (preferred):**
```bash
npm install -g @modelcontextprotocol/server-filesystem@1.2.3
```
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "node",
      "args": ["/usr/local/lib/node_modules/@modelcontextprotocol/server-filesystem/dist/index.js", "/projects"]
    }
  }
}
```

**Option 3 — Use only published, audited servers from known maintainers.** Review source before installing. Check `npm audit` after install.

**Principle of least privilege**: Grant only the directories/scopes the MCP server actually needs. Never grant `/` (filesystem root) unless required.

## Test

```bash
# Red: verify unpinned — should show floating version before fix
cat ~/.claude/claude_desktop_config.json | jq '.mcpServers[].args[]' | grep -v "@[0-9]"
# After fix: all servers should have @x.y.z version pinned
```

```typescript
it('all MCP server configurations pin an exact version', () => {
  const config = JSON.parse(readFileSync('claude_desktop_config.json', 'utf8'))
  for (const [name, server] of Object.entries(config.mcpServers)) {
    const args = (server as any).args as string[]
    const packageArg = args.find((a: string) => a.startsWith('@') || !a.startsWith('-'))
    if (packageArg) {
      expect(packageArg, `MCP server "${name}" must pin a version`).toMatch(/@\d+\.\d+\.\d+/)
    }
  }
})
```

## Detection

```bash
# Unpinned npx MCP configs
grep -rn '"command".*"npx"' ~/.claude/ ~/.config/claude/ | grep -v "@[0-9]\+\.[0-9]\+\.[0-9]\+"

# Review all MCP server configurations
grep -rn "mcpServers" ~/.claude/ ~/.config/claude/
```
