---
id: deps-lockfile-sync
domain: deps
severity: high
stack: "pnpm, npm, yarn, vercel"
date_added: 2026-03-25
project: beltway-events
---

# Package Lockfile Out of Sync

## Problem
CI/CD builds fail with a frozen lockfile error when `package.json` has been updated but the lockfile has not been regenerated. Vercel uses `--frozen-lockfile` by default — any mismatch is a hard build failure.

```
ERR_PNPM_OUTDATED_LOCKFILE: Cannot install with "frozen-lockfile" because the lockfile needs updates
```

## Root Cause
Adding or removing packages during development without running the package manager locally. Common in AI-assisted development where the model edits `package.json` directly without running install.

## Fix

After any `package.json` change, regenerate the lockfile immediately and commit it in the same commit as the dependency change:

```bash
# pnpm
pnpm install --no-frozen-lockfile
git add pnpm-lock.yaml
git commit -m "Add <package>; update lockfile"

# npm
npm install
git add package-lock.json
git commit -m "Add <package>; update lockfile"

# yarn
yarn install
git add yarn.lock
git commit -m "Add <package>; update lockfile"
```

If the package manager is not installed locally:
```bash
# pnpm via npm
npm install -g pnpm@10
pnpm install --no-frozen-lockfile
```

**Prevention**: Never manually edit `package.json` dependencies without running install. Treat the lockfile as part of the same atomic commit as the dependency change.

## Detection

```bash
# pnpm — dry run to detect mismatch without modifying
pnpm install --frozen-lockfile --dry-run
# Any output means the lockfile is out of sync

# npm
npm ci --dry-run

# yarn
yarn install --frozen-lockfile --check-files
```

## Test

The build is the test. The lockfile is in sync when:
```bash
# pnpm
pnpm install --frozen-lockfile  # exits 0

# Vercel build
vercel build  # no ERR_PNPM_OUTDATED_LOCKFILE error
```
