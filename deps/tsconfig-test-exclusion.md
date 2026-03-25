---
id: deps-tsconfig-test-exclusion
domain: deps
severity: medium
stack: "next.js, vitest, jest, typescript"
date_added: 2026-03-25
project: beltway-events
---

# Test Files Must Be Excluded from tsconfig.json

## Problem
Next.js TypeScript compilation picks up `vitest.setup.ts`, `vitest.config.ts`, `jest.config.ts`, and test files in non-standard locations, causing build failures:

```
Type error: Cannot find module 'vite' or its corresponding type declarations.
Type error: Cannot find module 'vitest' or its corresponding type declarations.
```

## Root Cause
Next.js runs `tsc` over all files matched by `tsconfig.json`. If test infrastructure files are not explicitly excluded, they get included in the type-check pass. Test files import test-only dependencies (`vitest`, `@testing-library/*`) that are `devDependencies` and are not available in the production type-check.

## Fix

Add test infrastructure files to the `exclude` array in `tsconfig.json`:

```json
{
  "compilerOptions": { ... },
  "exclude": [
    "node_modules",
    "supabase/tests",
    "**/__tests__/**",
    "**/*.test.ts",
    "**/*.test.tsx",
    "**/*.spec.ts",
    "**/*.spec.tsx",
    "vitest.setup.ts",
    "vitest.config.ts",
    "jest.config.ts",
    "jest.setup.ts"
  ]
}
```

## Rule

Any file that imports from `vitest`, `jest`, `@testing-library/*`, or other test-only packages must be in the `exclude` list, or in a separate `tsconfig.test.json`.

## Test

```bash
# Red: build fails with type errors before the fix
pnpm run build
# Must complete without errors related to vitest/jest modules
```

```typescript
// Verify via tsc directly
// tsc --noEmit should exit 0
```
