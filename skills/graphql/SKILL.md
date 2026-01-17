---
name: graphql
description: >
  GraphQL fundamentals for schema design and query writing. Covers type system
  (scalars, objects, interfaces, unions, enums, inputs), SDL syntax, operations
  (queries, mutations, subscriptions), nullability semantics, and TypeScript
  code generation. Use when working with GraphQL schemas, writing queries, or
  understanding GraphQL types. Does NOT cover Relay-specific patterns.
---

# GraphQL Fundamentals

GraphQL is a query language for APIs and a runtime for executing those queries. This skill covers core GraphQL concepts that are foundational to any GraphQL implementation.

## Quick Syntax Reference

| Operation | Syntax | Execution |
|-----------|--------|-----------|
| Query | `query Name($var: Type) { ... }` | Parallel |
| Mutation | `mutation Name($input: Input!) { ... }` | Sequential |
| Subscription | `subscription Name { ... }` | Streaming |
| Fragment | `fragment Name on Type { ... }` | Reusable selection |

## Field Selection

Fields are the basic unit of data selection in GraphQL:

```graphql
query {
  user {
    id
    name
    email
  }
}
```

### Nested Selection

Select fields on related objects:

```graphql
query {
  user {
    name
    posts {
      title
      comments {
        text
      }
    }
  }
}
```

## Arguments

Fields can accept arguments to filter or modify results:

```graphql
query {
  user(id: "123") {
    name
  }
  posts(first: 10, orderBy: CREATED_AT) {
    title
  }
}
```

### Argument Types

Arguments can be:
- Scalars: `id: "123"`, `count: 10`, `active: true`
- Enums: `status: PUBLISHED` (no quotes)
- Input objects: `filter: { name: "test", active: true }`
- Lists: `ids: ["1", "2", "3"]`

## Variables

Variables make queries reusable and prevent string interpolation:

```graphql
query GetUser($userId: ID!, $includeEmail: Boolean = false) {
  user(id: $userId) {
    name
    email @include(if: $includeEmail)
  }
}
```

Variable JSON:
```json
{
  "userId": "123",
  "includeEmail": true
}
```

### Variable Syntax

| Syntax | Meaning |
|--------|---------|
| `$var: Type` | Nullable variable |
| `$var: Type!` | Required variable |
| `$var: Type = default` | Variable with default |
| `$var: [Type]` | List variable |
| `$var: [Type!]!` | Required list of required items |

## Standard Directives

GraphQL has three built-in directives:

### @include and @skip

Conditionally include or skip fields:

```graphql
query GetUser($withPosts: Boolean!, $skipEmail: Boolean!) {
  user {
    name
    email @skip(if: $skipEmail)
    posts @include(if: $withPosts) {
      title
    }
  }
}
```

### @deprecated

Mark schema fields as deprecated (schema-side only):

```graphql
type User {
  name: String!
  fullName: String! @deprecated(reason: "Use 'name' instead")
}
```

## Field Aliasing

Rename fields in the response or query the same field with different arguments:

```graphql
query {
  currentUser: user(id: "me") {
    name
  }
  otherUser: user(id: "123") {
    userName: name
  }
}
```

Response:
```json
{
  "currentUser": { "name": "Alice" },
  "otherUser": { "userName": "Bob" }
}
```

## Type System Overview

GraphQL has a strong type system. Every field has a type.

### Which Type Should I Use?

| Need | Type | Example |
|------|------|---------|
| Primitive value | Scalar | `String`, `Int`, `DateTime` |
| Complex object with fields | Object Type | `type User { name: String }` |
| Shared fields across types | Interface | `interface Node { id: ID! }` |
| One of several unrelated types | Union | `union SearchResult = User \| Post` |
| Fixed set of values | Enum | `enum Status { ACTIVE, INACTIVE }` |
| Mutation input | Input Type | `input CreateUserInput { ... }` |

### Built-in Scalar Types

| Type | Description | Example |
|------|-------------|---------|
| `Int` | 32-bit signed integer | `42` |
| `Float` | Double-precision float | `3.14` |
| `String` | UTF-8 string | `"hello"` |
| `Boolean` | True or false | `true` |
| `ID` | Unique identifier (serialized as String) | `"abc123"` |

See [types.md](./types.md) for complete type system reference.

## Fragments

Fragments define reusable field selections:

```graphql
fragment UserFields on User {
  id
  name
  email
}

query {
  me {
    ...UserFields
  }
  user(id: "123") {
    ...UserFields
  }
}
```

### Inline Fragments

Use for type-specific selections on interfaces/unions:

```graphql
query {
  search(query: "test") {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
  }
}
```

## Introspection

Query the schema itself:

```graphql
query {
  __schema {
    types {
      name
    }
  }
  __type(name: "User") {
    fields {
      name
      type {
        name
      }
    }
  }
}
```

Common introspection fields:
- `__schema`: The schema definition
- `__type(name: String!)`: A specific type
- `__typename`: The concrete type of any object (can be added to any selection)

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `status: "PUBLISHED"` | Enum values shouldn't be quoted | `status: PUBLISHED` |
| `query { user.name }` | Dot notation doesn't exist | `query { user { name } }` |
| `$var: String` for required | Missing `!` means nullable | `$var: String!` |
| Selecting scalar subfields | Scalars have no subfields | Remove `{ }` from scalar fields |
| `... on User` without fields | Inline fragment needs selection | Add `{ field }` |
| Unused variables | All declared variables must be used | Remove unused `$var` |

## Navigation

- **[types.md](./types.md)** - Complete type system reference (scalars, objects, interfaces, unions, enums, inputs)
- **[operations.md](./operations.md)** - Query, mutation, and subscription patterns
- **[schema.md](./schema.md)** - Schema Definition Language (SDL) and schema design
- **[nullability.md](./nullability.md)** - Null semantics, error handling, and the `!` modifier
- **[typescript.md](./typescript.md)** - TypeScript code generation and type safety

## Request Structure

A complete GraphQL request consists of:

```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { name } }",
  "variables": { "id": "123" },
  "operationName": "GetUser"
}
```

- `query`: The GraphQL document (required)
- `variables`: Variable values (optional)
- `operationName`: Which operation to execute if document has multiple (optional)

## Response Structure

GraphQL responses always have this shape:

```json
{
  "data": { ... },
  "errors": [ ... ],
  "extensions": { ... }
}
```

- `data`: The result (null if top-level error)
- `errors`: Array of errors (optional)
- `extensions`: Implementation-specific metadata (optional)

### Partial Responses

Unlike REST, GraphQL can return partial data with errors:

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "posts": null
    }
  },
  "errors": [
    {
      "message": "Failed to fetch posts",
      "path": ["user", "posts"]
    }
  ]
}
```

See [nullability.md](./nullability.md) for details on null propagation.

## Operation Naming

Always name your operations for:
- Better debugging and logging
- Required for persisted queries
- Multiple operations in one document

```graphql
# Good
query GetUserProfile($id: ID!) {
  user(id: $id) { name }
}

# Avoid
query {
  user(id: "123") { name }
}
```

### Naming Conventions

| Item | Convention | Example |
|------|------------|---------|
| Operations | PascalCase verb + noun | `GetUser`, `CreatePost` |
| Fields | camelCase | `firstName`, `createdAt` |
| Types | PascalCase | `User`, `BlogPost` |
| Enums | SCREAMING_SNAKE_CASE | `ORDER_STATUS`, `ROLE_ADMIN` |
| Input types | PascalCase + "Input" | `CreateUserInput` |

## Best Practices

1. **Always use variables** - Never interpolate values into query strings
2. **Name all operations** - Enables tooling, debugging, and persisted queries
3. **Use fragments for shared fields** - Reduces duplication, aids composition
4. **Prefer specific queries** - Don't over-fetch; request only needed fields
5. **Use enums over strings** - Type safety for fixed value sets
6. **Design nullable by default** - See [nullability.md](./nullability.md)
7. **Use input types for mutations** - Single argument is cleaner and evolvable
