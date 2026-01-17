# GraphQL Type System

GraphQL has a strong, static type system. Every field, argument, and variable has a defined type that determines what values are valid.

## Scalar Types

Scalars are primitive leaf values in GraphQL.

### Built-in Scalars

| Type | Description | JSON Serialization | Example |
|------|-------------|-------------------|---------|
| `Int` | 32-bit signed integer | Number | `42`, `-17` |
| `Float` | Double-precision floating point | Number | `3.14`, `-0.5` |
| `String` | UTF-8 character sequence | String | `"hello"` |
| `Boolean` | True or false | Boolean | `true`, `false` |
| `ID` | Unique identifier | String | `"abc123"` |

### ID vs String

- Use `ID` for unique identifiers that shouldn't be human-meaningful
- `ID` signals "don't display this to users" to clients
- `ID` accepts both string and integer input, serializes as string

```graphql
type User {
  id: ID!           # Unique identifier
  username: String! # Human-readable identifier
}
```

### Custom Scalars

Define domain-specific scalar types:

```graphql
scalar DateTime
scalar JSON
scalar UUID
scalar Email
scalar URL
scalar PositiveInt
```

Common custom scalars and their typical representations:

| Scalar | Typical Format | Example |
|--------|---------------|---------|
| `DateTime` | ISO 8601 | `"2024-01-15T10:30:00Z"` |
| `Date` | ISO 8601 date | `"2024-01-15"` |
| `Time` | ISO 8601 time | `"10:30:00Z"` |
| `JSON` | Any valid JSON | `{"key": "value"}` |
| `UUID` | RFC 4122 | `"550e8400-e29b-41d4-a716-446655440000"` |
| `Email` | RFC 5322 | `"user@example.com"` |

## Object Types

Object types define entities with fields:

```graphql
type User {
  id: ID!
  name: String!
  email: String
  createdAt: DateTime!
  posts: [Post!]!
  profile: Profile
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  tags: [String!]!
}
```

### Field Definitions

Fields can have arguments:

```graphql
type User {
  id: ID!
  posts(
    first: Int = 10
    after: String
    status: PostStatus
  ): [Post!]!
}
```

## Interface Types

Interfaces define a set of fields that multiple types must implement:

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
}

type Post implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
}
```

### Querying Interfaces

```graphql
query {
  node(id: "123") {
    id                    # Available on all Node types
    ... on User {
      name                # User-specific field
    }
    ... on Post {
      title               # Post-specific field
    }
  }
}
```

### When to Use Interfaces

- Types share common fields
- You want to query heterogeneous collections
- Implementing types have meaningful shared behavior
- Example: `Node` interface for global object identification

## Union Types

Unions represent "one of" several types with no shared fields:

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}
```

### Querying Unions

Unions require inline fragments or `__typename`:

```graphql
query {
  search(query: "graphql") {
    __typename
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
    ... on Comment {
      text
    }
  }
}
```

### Interface vs Union

| Aspect | Interface | Union |
|--------|-----------|-------|
| Shared fields | Required | None |
| Common queries | Can query shared fields directly | Must use fragments |
| Use case | Types are related | Types are unrelated |
| Example | `Node { id }` | `SearchResult = User \| Post` |

Decision guide:
- Use **interface** when types share fields and behavior
- Use **union** when types are unrelated but appear together

## Enum Types

Enums define a fixed set of allowed values:

```graphql
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

type Post {
  status: PostStatus!
}

type Query {
  posts(status: PostStatus): [Post!]!
}
```

### Enum Usage

In queries, enum values are **not quoted**:

```graphql
query {
  posts(status: PUBLISHED) {  # Correct: no quotes
    title
  }
}
```

In JSON variables, enum values are **strings**:

```json
{
  "status": "PUBLISHED"
}
```

### Enum Naming Conventions

- Type name: PascalCase (`PostStatus`)
- Values: SCREAMING_SNAKE_CASE (`PUBLISHED`, `IN_PROGRESS`)

## Input Types

Input types define complex argument structures, typically for mutations:

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: Role = USER
  profile: ProfileInput
}

input ProfileInput {
  bio: String
  avatarUrl: URL
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}
```

### Input Type Rules

1. Can only contain scalars, enums, and other input types
2. Cannot contain object types, interfaces, or unions
3. Cannot have field arguments
4. Cannot be used as return types

### Why Use Input Types?

```graphql
# Without input type (fragile, hard to evolve)
mutation {
  createUser(name: "Alice", email: "a@b.com", role: USER)
}

# With input type (better)
mutation {
  createUser(input: { name: "Alice", email: "a@b.com" })
}
```

Benefits:
- Single argument is cleaner
- Easy to add optional fields
- Client caching works better
- Enables input type reuse

## List Types

Lists use brackets `[]`:

```graphql
type User {
  tags: [String]           # Nullable list of nullable strings
  roles: [Role!]           # Nullable list of non-null roles
  posts: [Post!]!          # Non-null list of non-null posts
}
```

### List Nullability Combinations

| Type | List | Items | Valid Values |
|------|------|-------|--------------|
| `[String]` | Nullable | Nullable | `null`, `[]`, `["a", null]` |
| `[String!]` | Nullable | Non-null | `null`, `[]`, `["a", "b"]` |
| `[String]!` | Non-null | Nullable | `[]`, `["a", null]` |
| `[String!]!` | Non-null | Non-null | `[]`, `["a", "b"]` |

**Common pattern**: Use `[Type!]!` for most lists - guarantees a list (possibly empty) with no null items.

## Type Modifiers Summary

| Modifier | Meaning | Default |
|----------|---------|---------|
| `Type` | Nullable | Yes (can be null) |
| `Type!` | Non-null | Must have value |
| `[Type]` | List | Nullable list |
| `[Type]!` | Required list | List must exist |
| `[Type!]` | List of non-null | Items cannot be null |
| `[Type!]!` | Required list of non-null | Strictest |

## Type Composition Patterns

### Entity Pattern

Standard entity with identification and timestamps:

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  # User-specific fields...
}
```

### Result Union Pattern

For operations that can fail in expected ways:

```graphql
type CreateUserSuccess {
  user: User!
}

type ValidationError {
  field: String!
  message: String!
}

type EmailTakenError {
  email: String!
  suggestion: String
}

union CreateUserResult = CreateUserSuccess | ValidationError | EmailTakenError

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

### Payload Pattern

Wrap mutation results for extensibility:

```graphql
type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: ErrorCode!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}
```

## Type References

Types can reference each other (circular references are allowed):

```graphql
type User {
  posts: [Post!]!
}

type Post {
  author: User!
  comments: [Comment!]!
}

type Comment {
  post: Post!
  author: User!
}
```

The schema forms a graph, enabling clients to traverse relationships in queries.
