# GraphQL Nullability

Nullability is one of GraphQL's most important and nuanced concepts. Understanding null propagation is essential for schema design.

## The Basics

In GraphQL, fields are **nullable by default**:

```graphql
type User {
  name: String    # Can be null
  email: String!  # Cannot be null (required)
}
```

This is the opposite of many programming languages where non-null is the default.

## Why Nullable by Default?

1. **Resilience**: Partial failures don't break entire responses
2. **Backend flexibility**: Services can fail independently
3. **Evolution**: Fields can be deprecated without breaking clients
4. **Permissions**: Fields can return null if unauthorized

## Null Propagation (Bubbling)

When a non-null field resolves to null, the null "bubbles up" to the nearest nullable parent.

### Example Schema

```graphql
type Query {
  user: User          # nullable
}

type User {
  id: ID!             # non-null
  name: String!       # non-null
  email: String       # nullable
  posts: [Post!]!     # non-null list of non-null items
}

type Post {
  title: String!      # non-null
}
```

### Propagation Scenarios

**Scenario 1**: `email` (nullable) returns null
```json
{
  "data": {
    "user": {
      "id": "123",
      "name": "Alice",
      "email": null
    }
  }
}
```
Result: Field is null, no propagation needed.

**Scenario 2**: `name` (non-null) fails/returns null
```json
{
  "data": {
    "user": null
  },
  "errors": [{ "message": "...", "path": ["user", "name"] }]
}
```
Result: Null bubbles up to `user` (nearest nullable ancestor).

**Scenario 3**: `user` (nullable) returns null
```json
{
  "data": {
    "user": null
  }
}
```
Result: Normal null, no error.

### List Null Propagation

```graphql
type User {
  posts: [Post!]!    # Non-null list of non-null posts
}
```

If one post in the list fails:
- The individual `Post!` cannot be null
- Null bubbles up to the list `[Post!]!`
- The list cannot be null, so it bubbles to `user`

```json
{
  "data": {
    "user": null
  },
  "errors": [{ "path": ["user", "posts", 2] }]
}
```

## List Nullability Matrix

| Type | Null List | Empty List | Null Items |
|------|-----------|------------|------------|
| `[Post]` | Yes | Yes | Yes |
| `[Post!]` | Yes | Yes | No |
| `[Post]!` | No | Yes | Yes |
| `[Post!]!` | No | Yes | No |

**Recommendation**: Use `[Type!]!` for most lists - guarantees a list with no null items, but allows empty.

## When to Use Non-Null

### Use Non-Null (`!`) For:

- **IDs**: `id: ID!` - entities always have identity
- **Required relationships**: `author: User!` - every post has an author
- **Guaranteed data**: `createdAt: DateTime!` - always present
- **Non-null lists**: `tags: [String!]!` - list exists, items exist

### Keep Nullable For:

- **Optional relationships**: `manager: User` - not everyone has one
- **Potentially missing data**: `bio: String` - user might not set
- **Fallible computations**: `recommendations: [Post]` - service might fail
- **Privacy-sensitive**: `email: String` - might be hidden

### The Nullable Principle

> "When in doubt, keep it nullable"

It's easier to make a nullable field non-null later than the reverse. Making a field non-null is a breaking change if any client depends on receiving null.

## Error Handling Patterns

### Using Nullable Fields

```graphql
type Query {
  user(id: ID!): User  # Returns null if not found
}
```

Client handles:
```typescript
const user = data.user;
if (user === null) {
  // Handle "not found"
}
```

### Using Result Unions

```graphql
type UserResult = User | NotFoundError | PermissionError

type Query {
  user(id: ID!): UserResult!  # Always returns something
}
```

Client handles:
```graphql
query {
  user(id: "123") {
    ... on User {
      name
    }
    ... on NotFoundError {
      message
    }
    ... on PermissionError {
      requiredRole
    }
  }
}
```

### Using Payload Types

```graphql
type UserPayload {
  user: User
  error: Error
}

type Query {
  user(id: ID!): UserPayload!
}
```

## Partial Responses

GraphQL can return data AND errors together:

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "posts": null,
      "friends": [
        { "name": "Bob" },
        { "name": "Carol" }
      ]
    }
  },
  "errors": [
    {
      "message": "Posts service unavailable",
      "path": ["user", "posts"]
    }
  ]
}
```

This enables graceful degradation - show what you can, indicate what failed.

## Schema Design Implications

### Overly Strict (Fragile)

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  profile: Profile!     # If profile service fails, entire user is null
  posts: [Post!]!       # If one post fails, user is null
  friends: [User!]!     # If one friend fails, user is null
}
```

### Appropriately Resilient

```graphql
type User {
  id: ID!
  name: String!
  email: String          # Optional
  profile: Profile       # Can gracefully fail
  posts: [Post!]!        # List required, items required (common pattern)
  friends: [User]!       # List required, items can fail individually
}
```

## Argument Nullability

For arguments, non-null means "required":

```graphql
type Query {
  user(id: ID!): User              # id is required
  users(
    first: Int = 10                # Optional with default
    filter: UserFilter             # Optional, can be omitted
    sort: SortOrder!               # Required
  ): [User!]!
}
```

## Checking Nullability in Queries

Use `__typename` to verify you got an object:

```graphql
query {
  user(id: "123") {
    __typename  # "User" if found, null propagates if not
    name
  }
}
```

For unions/interfaces:

```graphql
query {
  result(id: "123") {
    __typename
    ... on User { name }
    ... on NotFoundError { message }
  }
}
```
