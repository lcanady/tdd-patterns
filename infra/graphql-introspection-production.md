---
id: infra-graphql-introspection-production
domain: infra
severity: medium
stack: "node.js, python, *"
date_added: 2026-03-26
project: tdd-audit
---

# GraphQL Introspection Enabled in Production + Missing Depth/Complexity Limits

## Problem
GraphQL introspection is left enabled in production, revealing the full schema to attackers. Combined with missing query depth or complexity limits, an attacker can construct deeply-nested queries that exhaust CPU and memory.

```javascript
// WRONG — introspection on + no depth limit
const server = new ApolloServer({ typeDefs, resolvers });
// Default: introspection=true in all environments, no depth/complexity guards
```

## Root Cause
Introspection is enabled by default in Apollo and most GraphQL servers for developer convenience. Teams forget to disable it for production and never add query complexity analysis.

## Fix
```javascript
const depthLimit   = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production',
  validationRules: [
    depthLimit(7),
    createComplexityLimitRule(1000),
  ],
});
```

## Test
```javascript
test('introspection returns 400 in production', async () => {
  process.env.NODE_ENV = 'production';
  const res = await request(app)
    .post('/graphql')
    .send({ query: '{ __schema { types { name } } }' });
  expect(res.body.errors?.[0]?.message).toMatch(/introspection/i);
});

test('deeply nested query is rejected', async () => {
  const deepQuery = 'query { user { friends { friends { friends { id } } } } }';
  const res = await request(app).post('/graphql').send({ query: deepQuery });
  expect(res.body.errors?.[0]?.message).toMatch(/depth/i);
});
```

## Detection
```
new ApolloServer\(\s*\{(?![\s\S]{0,300}introspection\s*:\s*false)
graphql(?:Http|Server)[\s\S]{0,200}(?:schema|resolvers)(?![\s\S]{0,300}depthLimit)
# missing complexity or depth validation rules
```
