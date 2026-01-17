# Schema Definition Language (SDL)

SDL is the language for defining GraphQL schemas. It describes types, fields, and their relationships.

## Schema Root Types

Every schema has root types for operations:

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type Query {
  # Read operations
}

type Mutation {
  # Write operations
}

type Subscription {
  # Streaming operations
}
```

If using default names (`Query`, `Mutation`, `Subscription`), the schema declaration is optional.

## Type Definitions

### Object Types

```graphql
type User {
  id: ID!
  name: String!
  email: String
  posts: [Post!]!
}
```

### Field Arguments

```graphql
type Query {
  user(id: ID!): User
  users(
    first: Int = 10
    after: String
    filter: UserFilter
  ): UserConnection!
}
```

### Interfaces

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
```

### Unions

```graphql
union SearchResult = User | Post | Comment
```

### Enums

```graphql
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

### Input Types

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: Role = USER
}
```

### Custom Scalars

```graphql
scalar DateTime
scalar JSON
scalar UUID
```

## Documentation Strings

Use triple-quoted strings for descriptions:

```graphql
"""
A user in the system.
Users can create posts and comments.
"""
type User {
  "The unique identifier"
  id: ID!

  """
  The user's display name.
  This is shown publicly on their profile.
  """
  name: String!

  "Email address (only visible to the user)"
  email: String
}
```

### Argument Descriptions

```graphql
type Query {
  """
  Fetch a user by their unique ID.
  Returns null if user doesn't exist.
  """
  user(
    "The user's unique identifier"
    id: ID!
  ): User

  "List users with pagination"
  users(
    "Maximum number of users to return"
    first: Int = 10
    "Cursor for pagination"
    after: String
  ): UserConnection!
}
```

## Directives

### Using Directives

```graphql
type User {
  id: ID!
  name: String!
  legacyId: String @deprecated(reason: "Use 'id' instead")
}
```

### Defining Custom Directives

```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION

directive @cacheControl(
  maxAge: Int
  scope: CacheScope
) on FIELD_DEFINITION | OBJECT

type Query {
  publicData: String
  privateData: String @auth(requires: ADMIN)
  cachedData: String @cacheControl(maxAge: 300)
}
```

### Directive Locations

| Location | Applies To |
|----------|------------|
| `FIELD_DEFINITION` | Fields in types |
| `OBJECT` | Object types |
| `INPUT_OBJECT` | Input types |
| `ARGUMENT_DEFINITION` | Field/directive arguments |
| `ENUM` | Enum types |
| `ENUM_VALUE` | Enum values |
| `INTERFACE` | Interface types |
| `UNION` | Union types |
| `SCALAR` | Scalar types |

## Extending Types

Extend existing types to add fields:

```graphql
# Base schema
type User {
  id: ID!
  name: String!
}

# Extension (e.g., in another file)
extend type User {
  posts: [Post!]!
  comments: [Comment!]!
}

extend type Query {
  userByEmail(email: String!): User
}
```

Use extensions to:
- Organize large schemas across files
- Add fields from different modules
- Extend third-party schemas

## Schema Design Patterns

### Entry Points

Design clear entry points:

```graphql
type Query {
  # Current user
  viewer: User

  # Single entity by ID
  user(id: ID!): User
  post(id: ID!): Post

  # Lists with pagination
  users(first: Int!, after: String): UserConnection!
  posts(first: Int!, after: String, filter: PostFilter): PostConnection!

  # Global node lookup
  node(id: ID!): Node
}
```

### Mutation Patterns

Single input argument with payload return:

```graphql
input CreatePostInput {
  title: String!
  content: String!
  status: PostStatus = DRAFT
}

type CreatePostPayload {
  post: Post
  errors: [Error!]!
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Types | PascalCase | `User`, `BlogPost` |
| Fields | camelCase | `firstName`, `createdAt` |
| Arguments | camelCase | `userId`, `first` |
| Enums | PascalCase | `PostStatus` |
| Enum values | SCREAMING_SNAKE | `PUBLISHED`, `IN_PROGRESS` |
| Input types | PascalCase + Input | `CreateUserInput` |
| Payloads | PascalCase + Payload | `CreateUserPayload` |

## Schema Evolution

### Adding Fields (Safe)

```graphql
type User {
  id: ID!
  name: String!
  bio: String      # New nullable field - always safe
}
```

### Adding Required Fields (Breaking)

```graphql
type User {
  id: ID!
  name: String!
  email: String!   # New required field - breaks existing clients
}
```

To safely add required fields:
1. Add as nullable first
2. Migrate clients to provide it
3. Make non-null later

### Deprecating Fields

```graphql
type User {
  id: ID!
  name: String!
  fullName: String @deprecated(reason: "Use 'name' field instead")
}
```

### Removing Fields

1. Deprecate the field
2. Monitor usage
3. Remove after clients migrate

### Adding Enum Values

Adding values to enums can break clients not handling unknown values:

```graphql
enum Status {
  ACTIVE
  INACTIVE
  PENDING    # New value - clients must handle unknown
}
```

## Schema Organization

### File Structure

```
schema/
├── schema.graphql      # Root schema definition
├── types/
│   ├── user.graphql
│   ├── post.graphql
│   └── comment.graphql
├── inputs/
│   └── mutations.graphql
└── scalars.graphql
```

### Module Pattern

```graphql
# user.graphql
type User implements Node {
  id: ID!
  name: String!
}

extend type Query {
  user(id: ID!): User
  users(first: Int!): UserConnection!
}

extend type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}
```

Tools like GraphQL Tools merge files into a single schema.
