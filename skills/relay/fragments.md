# Relay Fragments Deep Dive

[← Back to SKILL.md](./SKILL.md)

## Fragment Arguments with @argumentDefinitions

Fragments can accept arguments, making them reusable with different configurations:

```typescript
// Avatar.tsx
import { graphql, useFragment } from 'react-relay';
import type { Avatar_user$key } from './__generated__/Avatar_user.graphql';

interface Props {
  user: Avatar_user$key;
}

export function Avatar({ user }: Props) {
  const data = useFragment(
    graphql`
      fragment Avatar_user on User
      @argumentDefinitions(
        size: { type: "Int", defaultValue: 48 }
        showBorder: { type: "Boolean", defaultValue: false }
      ) {
        name
        avatarUrl(size: $size)
        isPremium @include(if: $showBorder)
      }
    `,
    user
  );

  return (
    <img
      src={data.avatarUrl}
      alt={data.name}
      className={data.isPremium ? 'premium-border' : undefined}
    />
  );
}
```

## Passing Arguments with @arguments

Parent fragments or queries pass values using `@arguments`:

```typescript
// UserCard.tsx
graphql`
  fragment UserCard_user on User {
    ...Avatar_user @arguments(size: 96, showBorder: true)
    name
    bio
  }
`;

// Or in a query with variables
graphql`
  query UserProfileQuery($userId: ID!, $avatarSize: Int!) {
    user(id: $userId) {
      ...Avatar_user @arguments(size: $avatarSize)
    }
  }
`;
```

## Fragment Composition Patterns

### Nested Fragments

Components compose fragments from their children:

```typescript
// PostCard.tsx
graphql`
  fragment PostCard_post on Post {
    id
    title
    content
    createdAt
    author {
      ...UserAvatar_user
      ...UserName_user
    }
    ...PostActions_post
    ...PostComments_post
  }
`;
```

### Conditional Fragments with @include/@skip

```typescript
graphql`
  fragment PostCard_post on Post
  @argumentDefinitions(
    includeComments: { type: "Boolean", defaultValue: false }
  ) {
    id
    title
    ...PostComments_post @include(if: $includeComments)
  }
`;
```

### Type-Conditional Fragments

For interface/union types, use inline fragments:

```typescript
graphql`
  fragment NotificationItem_notification on Notification {
    id
    createdAt
    ... on FollowNotification {
      follower {
        ...UserCard_user
      }
    }
    ... on LikeNotification {
      post {
        ...PostCard_post
      }
      liker {
        name
      }
    }
    ... on CommentNotification {
      comment {
        content
        author {
          ...UserCard_user
        }
      }
    }
  }
`;
```

In the component:

```typescript
function NotificationItem({ notification }: Props) {
  const data = useFragment(/* fragment */, notification);

  switch (data.__typename) {
    case 'FollowNotification':
      return <FollowNotification follower={data.follower} />;
    case 'LikeNotification':
      return <LikeNotification post={data.post} liker={data.liker} />;
    case 'CommentNotification':
      return <CommentNotification comment={data.comment} />;
    default:
      return null;
  }
}
```

## Data Masking

Relay enforces **data masking**: a component can only access data from its own fragment, not data from child fragments.

```typescript
// Parent component
const data = useFragment(
  graphql`
    fragment UserProfile_user on User {
      id
      ...UserCard_user  # This data is NOT accessible here
    }
  `,
  user
);

// ❌ This will be undefined - data masking prevents access
console.log(data.name);

// ✅ Pass the ref to the child component
<UserCard user={data} />
```

**Why data masking?**
- Components are self-contained and reusable
- Changing a child's data requirements doesn't affect parents
- TypeScript enforces correct data flow

### Breaking Data Masking with @inline

Sometimes you need to read child fragment data (e.g., for logging, analytics, or complex conditionals):

```typescript
// Define fragment with @inline
graphql`
  fragment UserAnalytics_user on User @inline {
    id
    name
    signupDate
  }
`;

// Read it imperatively
import { readInlineData } from 'react-relay';
import type { UserAnalytics_user$key } from './__generated__/UserAnalytics_user.graphql';

function trackUser(userRef: UserAnalytics_user$key) {
  const user = readInlineData(
    graphql`
      fragment UserAnalytics_user on User @inline {
        id
        name
        signupDate
      }
    `,
    userRef
  );

  analytics.track('user_viewed', { userId: user.id, name: user.name });
}
```

**Use @inline sparingly** - it breaks the encapsulation that makes Relay powerful.

## TypeScript Types: $key vs $data

Relay generates two types for each fragment:

| Type | Usage | Example |
|------|-------|---------|
| `$key` | Fragment reference (opaque) | `UserCard_user$key` |
| `$data` | Resolved data (readable) | `UserCard_user$data` |

```typescript
// Props use $key - the opaque reference
interface Props {
  user: UserCard_user$key;
}

// useFragment returns $data - the actual values
function UserCard({ user }: Props) {
  const data: UserCard_user$data = useFragment(fragment, user);
  // data.name, data.avatarUrl, etc. are typed
}
```

### When to Use $data Directly

For utility functions that need resolved data:

```typescript
import type { UserCard_user$data } from './__generated__/UserCard_user.graphql';

function formatUserDisplayName(user: UserCard_user$data): string {
  return user.displayName || user.username || 'Anonymous';
}
```

## Fragment with Multiple Keys

For components that accept multiple fragment refs:

```typescript
interface Props {
  user: UserCard_user$key;
  viewer: UserCard_viewer$key;
}

function UserCard({ user, viewer }: Props) {
  const userData = useFragment(userFragment, user);
  const viewerData = useFragment(viewerFragment, viewer);

  const isCurrentUser = userData.id === viewerData.id;
  // ...
}
```

## Common Fragment Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Fragment not spread in parent | Data never fetched | Add `...FragmentName` to parent query/fragment |
| Passing resolved data instead of ref | TypeScript error, data masking broken | Pass the fragment key object, not `data.field` |
| Wrong naming convention | Compiler warnings, confusion | Use `ComponentName_propName` |
| Forgetting to run compiler | Types out of sync | Run `relay-compiler` after changes |
| Duplicate field selections | Compiler error | Remove duplicate fields, keep in one place |
| Circular fragment spreads | Compiler error | Restructure component hierarchy |

## Fragment Best Practices

1. **One fragment per component** - Keep data requirements co-located
2. **Name fragments after component + prop** - `UserCard_user`, not `UserFragment`
3. **Keep fragments minimal** - Only request fields you display
4. **Use arguments for variations** - Don't create multiple similar fragments
5. **Spread child fragments** - Don't duplicate child data requirements
6. **Type all props** - Use generated `$key` types for fragment refs

## Refetchable Fragments

Add `@refetchable` to enable refetching just this fragment's data:

```typescript
graphql`
  fragment UserProfile_user on User
  @refetchable(queryName: "UserProfileRefetchQuery") {
    id
    name
    bio
    followerCount
  }
`;
```

This generates a query that can refetch just this fragment:

```typescript
const [data, refetch] = useRefetchableFragment(fragment, user);

const handleRefresh = () => {
  refetch({}, { fetchPolicy: 'network-only' });
};
```

See [pagination.md](./pagination.md) for `usePaginationFragment` which extends this pattern.
