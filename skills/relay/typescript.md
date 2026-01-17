# Relay TypeScript Integration

[← Back to SKILL.md](./SKILL.md)

## Compiler Configuration

### package.json Configuration

```json
{
  "relay": {
    "src": "./src",
    "schema": "./schema.graphql",
    "language": "typescript",
    "artifactDirectory": "./src/__generated__",
    "eagerEsModules": true,
    "excludes": ["**/node_modules/**", "**/__mocks__/**", "**/__generated__/**"]
  }
}
```

### relay.config.js (Alternative)

```javascript
module.exports = {
  src: './src',
  schema: './schema.graphql',
  language: 'typescript',
  artifactDirectory: './src/__generated__',
  eagerEsModules: true,
  excludes: ['**/node_modules/**', '**/__mocks__/**', '**/__generated__/**'],

  // Optional: custom scalars
  customScalars: {
    DateTime: 'string',
    JSON: 'Record<string, unknown>',
  },
};
```

## Why artifactDirectory Matters

**Without `artifactDirectory`:** Types are generated next to source files
```
src/
  components/
    UserCard.tsx
    UserCard_user.graphql.ts  # Generated here
    __generated__/
      UserCardQuery.graphql.ts
```

**With `artifactDirectory`:** All types in one location
```
src/
  __generated__/
    UserCard_user.graphql.ts
    UserCardQuery.graphql.ts
  components/
    UserCard.tsx
```

**Benefits of centralized `artifactDirectory`:**
- Cleaner source directories
- Easier to gitignore all generated files
- Simpler import paths
- Better IDE performance (fewer files to watch)

## Generated Type Patterns

### Query Types

```typescript
// Generated: UserPageQuery.graphql.ts
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';

// Contains:
// - UserPageQuery (full query type with variables and response)
// - UserPageQuery$variables (just the variables)
// - UserPageQuery$data (just the response data)
```

### Fragment Types

```typescript
// Generated: UserCard_user.graphql.ts
import type {
  UserCard_user$key,   // Fragment reference (for props)
  UserCard_user$data,  // Resolved data shape
} from './__generated__/UserCard_user.graphql';
```

### Mutation Types

```typescript
// Generated: CreatePostMutation.graphql.ts
import type {
  CreatePostMutation,           // Full mutation type
  CreatePostMutation$variables, // Input variables
  CreatePostMutation$data,      // Response data
} from './__generated__/CreatePostMutation.graphql';
```

## Component Typing Patterns

### Fragment Component

```typescript
import { graphql, useFragment } from 'react-relay';
import type { UserCard_user$key } from './__generated__/UserCard_user.graphql';

interface Props {
  user: UserCard_user$key;
  className?: string;
}

export function UserCard({ user, className }: Props) {
  const data = useFragment(
    graphql`
      fragment UserCard_user on User {
        id
        name
        avatarUrl
      }
    `,
    user
  );

  // data is fully typed: { id: string, name: string, avatarUrl: string | null }
  return (
    <div className={className}>
      <img src={data.avatarUrl ?? '/default.png'} alt={data.name} />
      <span>{data.name}</span>
    </div>
  );
}
```

### Query Component

```typescript
import { graphql, useLazyLoadQuery } from 'react-relay';
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';

interface Props {
  userId: string;
}

export function UserPage({ userId }: Props) {
  const data = useLazyLoadQuery<UserPageQuery>(
    graphql`
      query UserPageQuery($userId: ID!) {
        user(id: $userId) {
          ...UserCard_user
        }
      }
    `,
    { userId }
  );

  // data.user is typed as the fragment reference or null
  if (!data.user) return <NotFound />;

  return <UserCard user={data.user} />;
}
```

### Mutation Component

```typescript
import { graphql, useMutation } from 'react-relay';
import type { CreatePostMutation } from './__generated__/CreatePostMutation.graphql';

export function CreatePostButton() {
  const [commit, isInFlight] = useMutation<CreatePostMutation>(graphql`
    mutation CreatePostMutation($input: CreatePostInput!) {
      createPost(input: $input) {
        post {
          id
          title
        }
      }
    }
  `);

  const handleCreate = () => {
    commit({
      variables: {
        input: { title: 'New Post', content: 'Hello' },
      },
      onCompleted: (response) => {
        // response.createPost?.post?.id is typed
        console.log('Created:', response.createPost?.post?.id);
      },
    });
  };

  return (
    <button onClick={handleCreate} disabled={isInFlight}>
      Create
    </button>
  );
}
```

## Nullable Types Handling

GraphQL nullable fields become `T | null` in TypeScript:

```typescript
// GraphQL: avatarUrl: String (nullable)
// TypeScript: avatarUrl: string | null

// Always handle nulls
<img src={data.avatarUrl ?? '/default-avatar.png'} />

// Or with optional chaining
{data.user?.name}
```

## Common TypeScript Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Types not found | Compiler not run | Run `relay-compiler` |
| Type mismatch after change | Stale generated types | Re-run compiler |
| `$key` type errors | Wrong prop passed | Pass fragment ref, not data |
| Generic type errors | Missing type parameter | Add `<QueryType>` to hook |
| Import errors | Wrong path | Import from `__generated__/` |
| Enum not recognized | Custom scalar | Add to `customScalars` config |

## Type-Safe Optimistic Updates

```typescript
commit({
  variables: { postId },
  optimisticResponse: {
    // TypeScript enforces correct shape
    likePost: {
      post: {
        id: postId,
        likeCount: 42,
        // ❌ TypeScript error if field missing or wrong type
      },
    },
  },
});
```

## Module Aliasing for Cleaner Imports

### tsconfig.json

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@generated/*": ["src/__generated__/*"]
    }
  }
}
```

### Usage

```typescript
// Before
import type { UserCard_user$key } from '../../../__generated__/UserCard_user.graphql';

// After
import type { UserCard_user$key } from '@generated/UserCard_user.graphql';
```

### Vite/Webpack Configuration

```typescript
// vite.config.ts
export default {
  resolve: {
    alias: {
      '@generated': path.resolve(__dirname, 'src/__generated__'),
    },
  },
};
```

## Strict Mode Considerations

With `strict: true` in tsconfig:

```typescript
// Nullable checks are enforced
const data = useFragment(fragment, user);

// ❌ Error: Object is possibly null
console.log(data.posts.length);

// ✅ Handle null case
console.log(data.posts?.length ?? 0);
```

## Custom Scalar Types

Define custom scalar mappings:

```javascript
// relay.config.js
module.exports = {
  customScalars: {
    DateTime: 'string',
    Date: 'string',
    Time: 'string',
    JSON: 'Record<string, unknown>',
    BigInt: 'string',
    UUID: 'string',
  },
};
```

For more complex types, create type definitions:

```typescript
// types/graphql-scalars.d.ts
declare module '@generated/*.graphql' {
  // DateTime is ISO 8601 string
  type DateTime = string;

  // JSON is parsed object
  type JSON = Record<string, unknown>;
}
```

## Enum Handling

GraphQL enums become TypeScript string unions:

```graphql
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

```typescript
// Generated type
type PostStatus = 'DRAFT' | 'PUBLISHED' | 'ARCHIVED';

// Usage
if (data.status === 'PUBLISHED') {
  // ...
}
```

## IDE Integration

### VS Code Extensions

- **GraphQL** - Syntax highlighting and validation
- **Relay** - Go to definition, autocomplete for Relay

### ESLint

```json
{
  "plugins": ["relay"],
  "rules": {
    "relay/graphql-syntax": "error",
    "relay/graphql-naming": "error",
    "relay/unused-fields": "warn"
  }
}
```
