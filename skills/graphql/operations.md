# GraphQL Operations

GraphQL has three operation types: queries (read), mutations (write), and subscriptions (streaming).

## Query Operations

Queries fetch data without side effects. Fields resolve in parallel when possible.

### Basic Query

```graphql
query {
  viewer {
    name
  }
}
```

### Named Query with Variables

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

Variables:
```json
{
  "id": "123"
}
```

### Multiple Root Fields

Root fields execute in parallel:

```graphql
query Dashboard {
  viewer {
    name
  }
  notifications {
    count
  }
  recentPosts(first: 5) {
    title
  }
}
```

## Mutation Operations

Mutations perform writes. Root fields execute **sequentially** (in order).

### Basic Mutation

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
  }
}
```

Variables:
```json
{
  "input": {
    "title": "Hello World",
    "content": "My first post"
  }
}
```

### Sequential Execution

Multiple mutation fields run in order, enabling dependent operations:

```graphql
mutation SetupUser {
  createUser(input: { name: "Alice" }) {
    id
  }
  # Runs after createUser completes
  sendWelcomeEmail(userId: $userId) {
    sent
  }
}
```

### Mutation Response Patterns

Always return the modified data:

```graphql
mutation UpdateUser($id: ID!, $input: UpdateUserInput!) {
  updateUser(id: $id, input: $input) {
    id
    name        # Return updated fields
    updatedAt   # Include metadata
  }
}
```

### Mutation Input Convention

Use a single `input` argument with an input type:

```graphql
# Preferred
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    user { id }
  }
}

# Avoid: multiple arguments
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    user { id }
  }
}
```

## Subscription Operations

Subscriptions provide real-time updates via a persistent connection.

### Basic Subscription

```graphql
subscription OnPostCreated {
  postCreated {
    id
    title
    author {
      name
    }
  }
}
```

### Filtered Subscription

```graphql
subscription OnMessageReceived($channelId: ID!) {
  messageReceived(channelId: $channelId) {
    id
    text
    sender {
      name
    }
  }
}
```

### Subscription Semantics

- Client opens long-lived connection (usually WebSocket)
- Server pushes data when events occur
- Each push has same shape as initial subscription
- Connection stays open until client disconnects

## Fragments

Fragments define reusable field selections.

### Named Fragments

```graphql
fragment UserFields on User {
  id
  name
  email
  avatarUrl
}

query GetUsers {
  me {
    ...UserFields
  }
  user(id: "123") {
    ...UserFields
    role   # Can add extra fields
  }
}
```

### Fragment on Interface

```graphql
fragment NodeFields on Node {
  id
  __typename
}

query {
  nodes(ids: ["1", "2"]) {
    ...NodeFields
  }
}
```

### Inline Fragments

Use for type-conditional selection:

```graphql
query Search($query: String!) {
  search(query: $query) {
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
      author { name }
    }
  }
}
```

### Inline Fragment for Directives

Apply directives to a group of fields:

```graphql
query GetUser($includeDetails: Boolean!) {
  user(id: "123") {
    name
    ... @include(if: $includeDetails) {
      email
      phone
      address
    }
  }
}
```

## Variables

Variables make operations reusable and type-safe.

### Variable Declaration

```graphql
query GetPosts(
  $first: Int!              # Required
  $after: String            # Optional (nullable)
  $status: PostStatus = PUBLISHED  # With default
) {
  posts(first: $first, after: $after, status: $status) {
    title
  }
}
```

### Variable Types

Variables can be:
- Scalars: `$id: ID!`, `$name: String`
- Enums: `$status: PostStatus!`
- Input types: `$input: CreateUserInput!`
- Lists: `$ids: [ID!]!`

### Variables vs Literal Arguments

| Aspect | Variables | Literals |
|--------|-----------|----------|
| Reusability | Yes | No |
| Dynamic values | Yes | No |
| Type checking | At operation level | Inline |
| Use case | Client input | Fixed values |

```graphql
# Variable (dynamic)
query GetUser($id: ID!) {
  user(id: $id) { name }
}

# Literal (fixed)
query GetAdmin {
  user(id: "admin") { name }
}
```

### Default Values

```graphql
query GetPosts(
  $first: Int = 10,
  $orderBy: PostOrder = CREATED_AT_DESC
) {
  posts(first: $first, orderBy: $orderBy) {
    title
  }
}
```

If variable is omitted, default is used. If explicitly set to `null`, null is used (for nullable variables).

## Operation Naming

### Why Name Operations?

1. **Debugging**: Errors reference operation name
2. **Logging**: Server can log by operation
3. **Tooling**: IDEs and tools use names
4. **Persisted queries**: Required for query persistence
5. **Multiple operations**: Required when document has multiple

### Naming Conventions

| Pattern | Example | Use Case |
|---------|---------|----------|
| Verb + Noun | `GetUser` | Queries |
| Verb + Noun | `CreatePost` | Mutations |
| On + Event | `OnPostCreated` | Subscriptions |

```graphql
# Queries: Get, List, Search
query GetUser($id: ID!) { ... }
query ListPosts($first: Int!) { ... }
query SearchUsers($query: String!) { ... }

# Mutations: Create, Update, Delete, action verbs
mutation CreateUser($input: CreateUserInput!) { ... }
mutation UpdatePost($id: ID!, $input: UpdatePostInput!) { ... }
mutation DeleteComment($id: ID!) { ... }
mutation PublishPost($id: ID!) { ... }

# Subscriptions: On + event
subscription OnPostCreated { ... }
subscription OnUserStatusChanged($userId: ID!) { ... }
```

## Multiple Operations

A document can contain multiple operations:

```graphql
query GetUser($id: ID!) {
  user(id: $id) { name }
}

query GetPosts($userId: ID!) {
  posts(authorId: $userId) { title }
}

mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) { id }
}
```

Use `operationName` in request to specify which to execute:

```json
{
  "query": "...",
  "operationName": "GetUser",
  "variables": { "id": "123" }
}
```

## Arguments vs Variables

| Concept | Purpose | Defined In |
|---------|---------|------------|
| Arguments | Parameters to fields | Schema |
| Variables | Dynamic values from client | Operation |

```graphql
# Schema defines arguments
type Query {
  user(id: ID!): User                    # 'id' is an argument
  posts(first: Int, after: String): [Post!]!
}

# Operation uses variables to provide argument values
query GetUser($userId: ID!) {            # $userId is a variable
  user(id: $userId) {                    # passed to 'id' argument
    name
  }
}
```
