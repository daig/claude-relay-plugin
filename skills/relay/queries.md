# Relay Queries and Data Loading

[← Back to SKILL.md](./SKILL.md)

## Query Loading Strategies

| Strategy | Hook | When to Use | Trade-off |
|----------|------|-------------|-----------|
| Lazy load | `useLazyLoadQuery` | Simple cases, prototypes | Triggers Suspense after render |
| Preloaded | `usePreloadedQuery` | Production routes | Requires loader setup |
| Imperative | `loadQuery` | Event handlers, prefetch | Manual query reference management |

## useLazyLoadQuery (Simple)

Fetches data when component renders. Simple but not optimal - Suspense triggers after React starts rendering.

```typescript
import { graphql, useLazyLoadQuery } from 'react-relay';
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';

function UserPage({ userId }: { userId: string }) {
  const data = useLazyLoadQuery<UserPageQuery>(
    graphql`
      query UserPageQuery($userId: ID!) {
        user(id: $userId) {
          name
          bio
          ...UserProfile_user
        }
      }
    `,
    { userId },
    { fetchPolicy: 'store-or-network' }
  );

  if (!data.user) return <NotFound />;

  return <UserProfile user={data.user} />;
}
```

## usePreloadedQuery + useQueryLoader (Preferred)

Start fetching before navigation/render for better perceived performance.

```typescript
// UserPage.tsx
import { graphql, usePreloadedQuery, PreloadedQuery } from 'react-relay';
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';

// Export the query for the loader
export const userPageQuery = graphql`
  query UserPageQuery($userId: ID!) {
    user(id: $userId) {
      name
      ...UserProfile_user
    }
  }
`;

interface Props {
  queryRef: PreloadedQuery<UserPageQuery>;
}

export function UserPage({ queryRef }: Props) {
  const data = usePreloadedQuery(userPageQuery, queryRef);

  if (!data.user) return <NotFound />;

  return <UserProfile user={data.user} />;
}
```

```typescript
// UserPageLoader.tsx (wrapper component)
import { useQueryLoader } from 'react-relay';
import { UserPage, userPageQuery } from './UserPage';
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';
import { useEffect } from 'react';

export function UserPageLoader({ userId }: { userId: string }) {
  const [queryRef, loadQuery] = useQueryLoader<UserPageQuery>(userPageQuery);

  useEffect(() => {
    loadQuery({ userId });
  }, [loadQuery, userId]);

  if (!queryRef) return <Loading />;

  return (
    <Suspense fallback={<Loading />}>
      <UserPage queryRef={queryRef} />
    </Suspense>
  );
}
```

## Route-Level Preloading

For routing libraries, preload data when the route matches:

```typescript
// routes.ts
import { loadQuery } from 'react-relay';
import { environment } from './RelayEnvironment';
import { userPageQuery } from './UserPage';

export const routes = [
  {
    path: '/user/:userId',
    component: UserPage,
    prepare: (params: { userId: string }) => ({
      queryRef: loadQuery(environment, userPageQuery, { userId: params.userId }),
    }),
  },
];
```

```typescript
// Router component
function Route({ route, params }) {
  const prepared = useMemo(() => route.prepare(params), [route, params]);
  const Component = route.component;

  return (
    <Suspense fallback={<Loading />}>
      <Component {...prepared} />
    </Suspense>
  );
}
```

## Imperative Loading with loadQuery

For event-driven preloading (hover, focus, button click):

```typescript
import { loadQuery } from 'react-relay';
import { environment } from './RelayEnvironment';
import { userPageQuery } from './UserPage';
import type { UserPageQuery } from './__generated__/UserPageQuery.graphql';

function UserLink({ userId, children }: { userId: string; children: React.ReactNode }) {
  const queryRef = useRef<PreloadedQuery<UserPageQuery> | null>(null);

  const handleMouseEnter = () => {
    // Start fetching on hover
    queryRef.current = loadQuery(environment, userPageQuery, { userId });
  };

  const handleClick = () => {
    // Navigate with preloaded data
    navigate(`/user/${userId}`, { state: { queryRef: queryRef.current } });
  };

  return (
    <a onMouseEnter={handleMouseEnter} onClick={handleClick}>
      {children}
    </a>
  );
}
```

## Fetch Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `store-or-network` | Use cache if available, else fetch | Default, good for most cases |
| `store-and-network` | Return cache immediately, fetch in background | Show stale data while refreshing |
| `network-only` | Always fetch, ignore cache | After mutations, pull-to-refresh |
| `store-only` | Only use cache, never fetch | Offline mode, guaranteed-cached data |

```typescript
useLazyLoadQuery(query, variables, {
  fetchPolicy: 'network-only',
});

// Or with useQueryLoader
loadQuery({ userId }, { fetchPolicy: 'store-and-network' });
```

## Query Options Reference

```typescript
useLazyLoadQuery(query, variables, {
  // Fetch policy (see table above)
  fetchPolicy: 'store-or-network',

  // Network cache config (distinct from Relay store)
  networkCacheConfig: {
    force: true, // Bypass HTTP cache
  },

  // Skip Suspense for this query (not recommended)
  // fetchKey: string | number, // Force refetch when key changes
});
```

## Suspense Integration Patterns

### Single Loading State

```typescript
<Suspense fallback={<PageSkeleton />}>
  <UserPage queryRef={queryRef} />
</Suspense>
```

### Nested Suspense (Progressive Loading)

```typescript
<Suspense fallback={<HeaderSkeleton />}>
  <Header />

  <Suspense fallback={<ContentSkeleton />}>
    <MainContent queryRef={queryRef} />

    <Suspense fallback={<CommentsSkeleton />}>
      <Comments />
    </Suspense>
  </Suspense>
</Suspense>
```

### SuspenseList for Coordinated Loading

```typescript
import { SuspenseList, Suspense } from 'react';

<SuspenseList revealOrder="forwards" tail="collapsed">
  <Suspense fallback={<UserCardSkeleton />}>
    <UserCard user={user1} />
  </Suspense>
  <Suspense fallback={<UserCardSkeleton />}>
    <UserCard user={user2} />
  </Suspense>
  <Suspense fallback={<UserCardSkeleton />}>
    <UserCard user={user3} />
  </Suspense>
</SuspenseList>
```

## Multiple Queries in One Component

If a component needs data from multiple queries:

```typescript
function Dashboard({ userQueryRef, statsQueryRef }: Props) {
  // These can load in parallel if both refs are available
  const userData = usePreloadedQuery(userQuery, userQueryRef);
  const statsData = usePreloadedQuery(statsQuery, statsQueryRef);

  return (
    <div>
      <UserInfo user={userData.user} />
      <StatsPanel stats={statsData.stats} />
    </div>
  );
}
```

## Refetching a Query

For pull-to-refresh or polling:

```typescript
function UserPage({ queryRef }: Props) {
  const data = usePreloadedQuery(query, queryRef);
  const [, loadQuery] = useQueryLoader(query);

  const handleRefresh = () => {
    loadQuery(
      { userId: data.user.id },
      { fetchPolicy: 'network-only' }
    );
  };

  return (
    <div>
      <button onClick={handleRefresh}>Refresh</button>
      <UserProfile user={data.user} />
    </div>
  );
}
```

## Query Best Practices

1. **Preload at route boundaries** - Start fetching before component mounts
2. **Use fragments for component data** - Queries should spread fragments, not define all fields
3. **One query per route/page** - Avoid multiple queries that could be combined
4. **Handle loading states** - Proper Suspense boundaries with meaningful fallbacks
5. **Handle null/error cases** - Data might be null, network might fail
6. **Avoid query waterfalls** - Don't wait for one query to start another

## Common Query Patterns

### Viewer Query (Current User)

```typescript
graphql`
  query AppQuery {
    viewer {
      id
      name
      email
      ...Sidebar_viewer
      ...Header_viewer
    }
  }
`;
```

### List Query

```typescript
graphql`
  query PostsPageQuery($first: Int!, $after: String) {
    posts(first: $first, after: $after) {
      edges {
        node {
          id
          ...PostCard_post
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;
```

### Detail Query

```typescript
graphql`
  query PostDetailQuery($postId: ID!) {
    node(id: $postId) {
      ... on Post {
        id
        title
        content
        ...PostMeta_post
        ...PostComments_post
      }
    }
  }
`;
```
