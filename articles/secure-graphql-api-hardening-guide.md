---
layout: default
title: "Secure GraphQL API Hardening Guide"
description: "Harden GraphQL APIs against introspection abuse, query depth attacks, batching exploits, and injection vulnerabilities with practical configuration examples"
date: 2026-03-22
author: theluckystrike
permalink: /secure-graphql-api-hardening-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api]
---

{% raw %}

# Secure GraphQL API Hardening Guide

GraphQL's flexibility is also its attack surface. A single endpoint accepting arbitrary queries can expose your entire data model if left unprotected. This guide covers the most exploited GraphQL vulnerabilities and how to fix each one with concrete configuration.

## Common GraphQL Attack Vectors

Before hardening, understand what you're protecting against:

1. **Introspection abuse** — attackers query your schema to map the full data model
2. **Query depth attacks** — deeply nested queries exhaust resources
3. **Field suggestion leakage** — error messages hint at valid field names
4. **Batching attacks** — send thousands of mutations in one request to brute-force or overwhelm
5. **Injection via arguments** — unsanitized arguments passed to SQL/NoSQL/OS commands
6. **Overly permissive resolvers** — resolvers that return more than the authenticated user should see

## Step 1 — Disable Introspection in Production

Introspection lets anyone enumerate your entire schema. Disable it unless you have a specific developer tooling need:

**Apollo Server (Node.js):**

```javascript
const { ApolloServer } = require('@apollo/server');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production',
  // Also disable playground in production
  plugins: [
    process.env.NODE_ENV === 'production'
      ? ApolloServerPluginDisabledLanding()
      : ApolloServerPluginLandingPageLocalDefault()
  ]
});
```

**graphene-django (Python):**

```python
# settings.py
GRAPHENE = {
    'SCHEMA': 'myapp.schema.schema',
    'INTROSPECTION': not DEBUG,  # False in production
}
```

Verify introspection is disabled:

```bash
curl -s -X POST https://api.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' | jq .

# Expected: {"errors":[{"message":"GraphQL introspection is not allowed..."}]}
```

## Step 2 — Enforce Query Depth Limits

A query like `user { friends { friends { friends { name } } } }` can cause exponential resolver work. Limit nesting depth:

**Node.js with graphql-depth-limit:**

```bash
npm install graphql-depth-limit
```

```javascript
const depthLimit = require('graphql-depth-limit');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(5)], // max 5 levels of nesting
});
```

**Python with graphene + custom validator:**

```python
from graphql import validate, build_ast_schema
from graphql.language import parse

MAX_DEPTH = 5

def check_query_depth(query_str):
    def get_depth(node, depth=0):
        if hasattr(node, 'selection_set') and node.selection_set:
            return max(
                get_depth(sel, depth + 1)
                for sel in node.selection_set.selections
            )
        return depth

    ast = parse(query_str)
    for definition in ast.definitions:
        if get_depth(definition) > MAX_DEPTH:
            raise ValueError(f"Query depth exceeds maximum of {MAX_DEPTH}")
```

## Step 3 — Limit Query Complexity

Beyond depth, complex queries with many fields are expensive. Use a complexity cost analysis:

```bash
npm install graphql-query-complexity
```

```javascript
const {
  createComplexityLimitRule,
  fieldExtensionsEstimator,
  simpleEstimator,
} = require('graphql-query-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      estimators: [
        fieldExtensionsEstimator(),
        simpleEstimator({ defaultComplexity: 1 }),
      ],
      onCost: (cost) => {
        console.log('Query cost:', cost);
      },
    }),
  ],
});
```

## Step 4 — Disable Field Suggestions

GraphQL returns helpful error messages like "Did you mean 'password'?" when a field typo occurs. This leaks your schema even when introspection is disabled.

**Apollo Server:**

```javascript
const { ApolloServerErrorCode } = require('@apollo/server/errors');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (formattedError) => {
    // Remove field suggestions from errors
    if (formattedError.message.includes('Did you mean')) {
      return { message: 'Unknown field.' };
    }
    return formattedError;
  },
});
```

## Step 5 — Rate Limit and Throttle

Per-IP or per-user rate limiting prevents brute force via GraphQL mutations:

```javascript
const rateLimit = require('express-rate-limit');

const graphqlLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute per IP
  message: 'Too many requests.',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/graphql', graphqlLimiter);
```

For query batching (array of operations in one POST), disable it or limit batch size:

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  allowBatchedHttpRequests: false, // or set a max: allowBatchedHttpRequests: 10
});
```

## Step 6 — Authorization at Resolver Level

Authentication (are you logged in?) and authorization (can you see this?) must both be enforced. Never rely on the schema structure alone:

```javascript
const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      // Authentication check
      if (!context.user) {
        throw new AuthenticationError('You must be logged in.');
      }

      // Authorization check — users can only see their own data
      if (context.user.id !== id && !context.user.isAdmin) {
        throw new ForbiddenError('Not authorized to access this user.');
      }

      return db.users.findById(id);
    },
  },
};
```

Use directives for declarative authorization:

```graphql
directive @auth(requires: Role = USER) on FIELD_DEFINITION

type Query {
  adminData: AdminPayload @auth(requires: ADMIN)
  userData: UserPayload @auth(requires: USER)
}
```

## Step 7 — Input Validation and Sanitization

GraphQL arguments should be treated as untrusted input:

```javascript
const { GraphQLScalarType, Kind } = require('graphql');

// Custom scalar for validated email
const EmailAddressType = new GraphQLScalarType({
  name: 'EmailAddress',
  description: 'A valid email address',
  parseValue(value) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value)) {
      throw new UserInputError('Invalid email address format');
    }
    return value.toLowerCase().trim();
  },
  serialize(value) { return value; },
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      return this.parseValue(ast.value);
    }
    throw new UserInputError('Email must be a string');
  }
});
```

## Step 8 — Persisted Queries

Replace arbitrary query strings with query hashes. Clients register queries in advance; the server only executes known queries:

```javascript
// Apollo Server with automatic persisted queries
const { ApolloServerPluginCacheControl } = require('@apollo/server/plugin/cacheControl');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    ApolloServerPluginCacheControl({ defaultMaxAge: 5 }),
  ],
  // In combination with APQ on the client side,
  // only registered query hashes are accepted
});
```

For maximum hardening in production, reject any query not on a pre-approved allowlist.

## Security Checklist

```
[ ] Introspection disabled in production
[ ] Query depth limited (<= 10 levels)
[ ] Query complexity budgeted
[ ] Field suggestion messages scrubbed from errors
[ ] Rate limiting per IP/user on /graphql endpoint
[ ] Batched requests disabled or limited
[ ] Authorization enforced inside resolvers, not just schema
[ ] Input validation with custom scalars or validation library
[ ] HTTPS enforced — no plain HTTP accepted
[ ] CORS configured for known origins only
[ ] HTTP method restricted to POST (GET GraphQL exposes queries in logs)
```

## Related Reading

- [Secure CORS Configuration Guide](/privacy-tools-guide/secure-cors-configuration-guide/)
- [Secure OAuth2 Implementation Checklist](/privacy-tools-guide/secure-oauth2-implementation-checklist/)
- [Secure gRPC Communication Setup Guide](/privacy-tools-guide/secure-grpc-communication-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
