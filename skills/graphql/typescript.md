# GraphQL TypeScript Integration

Generate TypeScript types from GraphQL schemas and operations for type-safe development.

## GraphQL Code Generator

[GraphQL Code Generator](https://the-guild.dev/graphql/codegen) is the standard tool for generating TypeScript types.

### Installation

```bash
npm install -D @graphql-codegen/cli
npm install -D @graphql-codegen/typescript
npm install -D @graphql-codegen/typescript-operations
```

### Configuration

Create `codegen.ts`:

```typescript
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: './schema.graphql',
  documents: ['src/**/*.graphql', 'src/**/*.tsx'],
  generates: {
    './src/__generated__/types.ts': {
      plugins: [
        'typescript',
        'typescript-operations',
      ],
    },
  },
};

export default config;
```

### Running

```bash
npx graphql-codegen
# Or add to package.json scripts
npm run codegen
```

## Generated Types

### Schema Types

From this schema:

```graphql
type User {
  id: ID!
  name: String!
  email: String
  role: Role!
}

enum Role {
  USER
  ADMIN
}
```

Generates:

```typescript
export type User = {
  __typename?: 'User';
  id: string;
  name: string;
  email?: string | null;
  role: Role;
};

export enum Role {
  User = 'USER',
  Admin = 'ADMIN',
}
```

### Operation Types

From this query:

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

Generates:

```typescript
export type GetUserQueryVariables = {
  id: string;
};

export type GetUserQuery = {
  __typename?: 'Query';
  user?: {
    __typename?: 'User';
    id: string;
    name: string;
    email?: string | null;
  } | null;
};
```

## Common Plugins

| Plugin | Purpose |
|--------|---------|
| `typescript` | Base TypeScript types from schema |
| `typescript-operations` | Types for queries/mutations |
| `typescript-resolvers` | Resolver type signatures |
| `typed-document-node` | TypedDocumentNode for clients |
| `typescript-react-apollo` | React Apollo hooks |
| `typescript-urql` | URQL hooks |

### Plugin Configuration

```typescript
const config: CodegenConfig = {
  generates: {
    './src/__generated__/types.ts': {
      plugins: ['typescript', 'typescript-operations'],
      config: {
        // Use 'interface' instead of 'type'
        declarationKind: 'interface',
        // Avoid optional chaining in generated types
        avoidOptionals: false,
        // Add __typename to all types
        addUnderscoreToArgsType: true,
        // Enum as const instead of enum
        enumsAsConst: true,
      },
    },
  },
};
```

## Resolver Types

For GraphQL servers, generate resolver type signatures:

```bash
npm install -D @graphql-codegen/typescript-resolvers
```

Configuration:

```typescript
generates: {
  './src/__generated__/resolvers.ts': {
    plugins: ['typescript', 'typescript-resolvers'],
    config: {
      contextType: '../context#Context',
      mappers: {
        User: '../models#UserModel',
      },
    },
  },
}
```

Generated resolver types:

```typescript
export type QueryResolvers<Context = any> = {
  user?: Resolver<Maybe<User>, {}, Context, QueryUserArgs>;
  users?: Resolver<Array<User>, {}, Context, QueryUsersArgs>;
};

export type UserResolvers<Context = any> = {
  id?: Resolver<string, User, Context>;
  name?: Resolver<string, User, Context>;
  posts?: Resolver<Array<Post>, User, Context>;
};
```

Usage:

```typescript
import { QueryResolvers, UserResolvers } from './__generated__/resolvers';

const Query: QueryResolvers<Context> = {
  user: async (_, { id }, context) => {
    return context.db.users.findById(id);
  },
};

const User: UserResolvers<Context> = {
  posts: async (parent, _, context) => {
    return context.db.posts.findByAuthor(parent.id);
  },
};
```

## Client Integration

### Typed Document Node

```bash
npm install -D @graphql-codegen/typed-document-node
```

```typescript
generates: {
  './src/__generated__/graphql.ts': {
    plugins: [
      'typescript',
      'typescript-operations',
      'typed-document-node',
    ],
  },
}
```

Usage with any client:

```typescript
import { GetUserDocument } from './__generated__/graphql';

// Type-safe variables and response
const result = await client.query({
  query: GetUserDocument,
  variables: { id: '123' }, // Type-checked
});

result.data.user?.name; // Type-safe access
```

### Framework-Specific Hooks

For React Apollo:

```typescript
generates: {
  './src/__generated__/graphql.tsx': {
    plugins: [
      'typescript',
      'typescript-operations',
      'typescript-react-apollo',
    ],
  },
}
```

Generates:

```typescript
export function useGetUserQuery(options: QueryHookOptions<GetUserQuery, GetUserQueryVariables>) {
  return useQuery<GetUserQuery, GetUserQueryVariables>(GetUserDocument, options);
}
```

## Watch Mode

For development:

```bash
npx graphql-codegen --watch
```

Or in `codegen.ts`:

```typescript
const config: CodegenConfig = {
  watch: true,
  // ...
};
```

## Monorepo Setup

For monorepos with shared schema:

```typescript
const config: CodegenConfig = {
  schema: '../../packages/schema/src/**/*.graphql',
  documents: 'src/**/*.graphql',
  generates: {
    './src/__generated__/': {
      preset: 'client',
    },
  },
};
```

## Type Safety Tips

### Handling Nullability

```typescript
// Generated type has optional fields
type GetUserQuery = {
  user?: {
    name: string;
    email?: string | null;
  } | null;
};

// Always check nullability
function UserProfile({ data }: { data: GetUserQuery }) {
  if (!data.user) {
    return <NotFound />;
  }

  return (
    <div>
      <h1>{data.user.name}</h1>
      {data.user.email && <p>{data.user.email}</p>}
    </div>
  );
}
```

### Fragment Types

Use fragments for component props:

```graphql
fragment UserCard on User {
  id
  name
  avatarUrl
}

query GetUsers {
  users {
    ...UserCard
  }
}
```

```typescript
import { UserCardFragment } from './__generated__/graphql';

function UserCard({ user }: { user: UserCardFragment }) {
  return <div>{user.name}</div>;
}
```

## Build Integration

Add to `package.json`:

```json
{
  "scripts": {
    "codegen": "graphql-codegen",
    "codegen:watch": "graphql-codegen --watch",
    "build": "npm run codegen && tsc",
    "dev": "concurrently \"npm run codegen:watch\" \"npm run start:dev\""
  }
}
```
