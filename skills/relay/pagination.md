# Relay Pagination

[← Back to SKILL.md](./SKILL.md)

## Connection Specification

Relay pagination follows the [GraphQL Cursor Connections Specification](https://relay.dev/graphql/connections.htm):

```graphql
type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

**Key concepts:**
- `edges` - Array of edge objects containing nodes and cursors
- `node` - The actual data object
- `cursor` - Opaque string for pagination position
- `pageInfo` - Pagination state information

## Setting Up a Pagination Fragment

Three directives are essential:

```typescript
graphql`
  fragment UserPosts_user on User
  @argumentDefinitions(
    count: { type: "Int", defaultValue: 10 }
    cursor: { type: "String" }
  )
  @refetchable(queryName: "UserPostsPaginationQuery") {
    posts(first: $count, after: $cursor)
      @connection(key: "UserPosts_posts") {
      edges {
        node {
          id
          ...PostCard_post
        }
      }
    }
  }
`;
```

| Directive | Purpose |
|-----------|---------|
| `@argumentDefinitions` | Define `count` and `cursor` variables |
| `@refetchable` | Generate pagination query automatically |
| `@connection` | Mark field as paginated, enable automatic merging |

## usePaginationFragment Hook

```typescript
import { graphql, usePaginationFragment } from 'react-relay';
import type { UserPosts_user$key } from './__generated__/UserPosts_user.graphql';

interface Props {
  user: UserPosts_user$key;
}

export function UserPosts({ user }: Props) {
  const {
    data,
    loadNext,
    loadPrevious,
    hasNext,
    hasPrevious,
    isLoadingNext,
    isLoadingPrevious,
    refetch,
  } = usePaginationFragment(
    graphql`
      fragment UserPosts_user on User
      @argumentDefinitions(
        count: { type: "Int", defaultValue: 10 }
        cursor: { type: "String" }
      )
      @refetchable(queryName: "UserPostsPaginationQuery") {
        posts(first: $count, after: $cursor)
          @connection(key: "UserPosts_posts") {
          edges {
            node {
              id
              ...PostCard_post
            }
          }
        }
      }
    `,
    user
  );

  return (
    <div>
      {data.posts.edges.map(({ node }) => (
        <PostCard key={node.id} post={node} />
      ))}

      {hasNext && (
        <button onClick={() => loadNext(10)} disabled={isLoadingNext}>
          {isLoadingNext ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

## usePaginationFragment Return Values

| Property | Type | Description |
|----------|------|-------------|
| `data` | Fragment data | Current loaded data |
| `loadNext` | `(count: number, options?) => void` | Load next page |
| `loadPrevious` | `(count: number, options?) => void` | Load previous page |
| `hasNext` | `boolean` | More items available after |
| `hasPrevious` | `boolean` | More items available before |
| `isLoadingNext` | `boolean` | Currently loading next |
| `isLoadingPrevious` | `boolean` | Currently loading previous |
| `refetch` | `(variables, options?) => void` | Reset and refetch |

## Infinite Scroll Implementation

```typescript
function InfinitePostList({ user }: Props) {
  const { data, loadNext, hasNext, isLoadingNext } = usePaginationFragment(
    fragment,
    user
  );

  const observerRef = useRef<IntersectionObserver>();
  const loadMoreRef = useCallback(
    (node: HTMLDivElement | null) => {
      if (observerRef.current) observerRef.current.disconnect();

      observerRef.current = new IntersectionObserver((entries) => {
        if (entries[0].isIntersecting && hasNext && !isLoadingNext) {
          loadNext(10);
        }
      });

      if (node) observerRef.current.observe(node);
    },
    [hasNext, isLoadingNext, loadNext]
  );

  return (
    <div>
      {data.posts.edges.map(({ node }) => (
        <PostCard key={node.id} post={node} />
      ))}

      <div ref={loadMoreRef}>
        {isLoadingNext && <Spinner />}
      </div>
    </div>
  );
}
```

## useRefetchableFragment for Search/Filter

When you need to refetch with different variables (not just pagination):

```typescript
function SearchableUserList({ query }: { query: SearchableUserList_query$key }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [isPending, startTransition] = useTransition();

  const [data, refetch] = useRefetchableFragment(
    graphql`
      fragment SearchableUserList_query on Query
      @argumentDefinitions(
        search: { type: "String", defaultValue: "" }
        count: { type: "Int", defaultValue: 20 }
      )
      @refetchable(queryName: "SearchableUserListRefetchQuery") {
        users(search: $search, first: $count)
          @connection(key: "SearchableUserList_users") {
          edges {
            node {
              id
              ...UserCard_user
            }
          }
        }
      }
    `,
    query
  );

  const handleSearch = (term: string) => {
    setSearchTerm(term);
    startTransition(() => {
      refetch({ search: term });
    });
  };

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search users..."
      />

      {isPending && <Spinner />}

      {data.users.edges.map(({ node }) => (
        <UserCard key={node.id} user={node} />
      ))}
    </div>
  );
}
```

## Combining Search and Pagination

```typescript
const { data, loadNext, hasNext, isLoadingNext, refetch } = usePaginationFragment(
  graphql`
    fragment PostList_query on Query
    @argumentDefinitions(
      search: { type: "String", defaultValue: "" }
      count: { type: "Int", defaultValue: 10 }
      cursor: { type: "String" }
    )
    @refetchable(queryName: "PostListPaginationQuery") {
      posts(search: $search, first: $count, after: $cursor)
        @connection(key: "PostList_posts") {
        edges {
          node {
            id
            ...PostCard_post
          }
        }
      }
    }
  `,
  query
);

// Search resets pagination
const handleSearch = (term: string) => {
  refetch({ search: term }); // Resets to first page
};

// Load more continues from cursor
const handleLoadMore = () => {
  loadNext(10);
};
```

## @connection Key Best Practices

The `key` in `@connection` must be unique per parent:

```graphql
# Good - unique per User
fragment UserPosts_user on User {
  posts @connection(key: "UserPosts_posts") { ... }
}

# Good - includes filter in key for separate caches
fragment UserPosts_user on User {
  publishedPosts: posts(status: PUBLISHED)
    @connection(key: "UserPosts_publishedPosts") { ... }
  draftPosts: posts(status: DRAFT)
    @connection(key: "UserPosts_draftPosts") { ... }
}
```

### Connection Key with Filters

The `filters` parameter controls which arguments are part of the connection's identity:

```graphql
# Include category in identity - different categories = different connections
posts(first: $count, after: $cursor, category: $category)
  @connection(key: "UserPosts_posts", filters: ["category"]) {
  ...
}
```

**Important:** By default, ALL non-pagination arguments become part of the connection identity. This affects `ConnectionHandler.getConnection()` and `@appendEdge`:

```graphql
# ⚠️ orderBy is part of connection identity by default
messages(first: 100, orderBy: CREATED_AT_ASC)
  @connection(key: "Chat_messages") {
  ...
}

# To use @appendEdge, you must either:
# 1. Pass filters to ConnectionHandler.getConnection(record, key, { orderBy: "CREATED_AT_ASC" })
# 2. Or exclude orderBy from identity with filters: []
```

**Use `filters: []` to ignore all non-pagination arguments:**

```graphql
# orderBy is NOT part of identity - simpler connection handling
messages(first: 100, orderBy: CREATED_AT_ASC)
  @connection(key: "Chat_messages", filters: []) {
  ...
}

# Now @appendEdge and ConnectionHandler.getConnection() work without specifying orderBy
```

## Important Behaviors

| Behavior | Description |
|----------|-------------|
| Automatic merging | New pages merge with existing data |
| Cursor-based | Uses `after`/`before` cursors, not offsets |
| Refetch resets | Calling `refetch()` clears loaded pages |
| Node deduplication | Same `id` nodes are deduplicated |
| Edge order preserved | Edges maintain server order |

## Bidirectional Pagination

For chat-like interfaces with both directions:

```typescript
graphql`
  fragment MessageList_conversation on Conversation
  @argumentDefinitions(
    first: { type: "Int" }
    after: { type: "String" }
    last: { type: "Int" }
    before: { type: "String" }
  )
  @refetchable(queryName: "MessageListPaginationQuery") {
    messages(first: $first, after: $after, last: $last, before: $before)
      @connection(key: "MessageList_messages") {
      edges {
        node {
          id
          ...Message_message
        }
      }
    }
  }
`;

// Load newer messages
loadNext(20);

// Load older messages
loadPrevious(20);
```

## Streaming Pagination with @stream

For large lists, stream edges as they arrive:

```graphql
fragment UserPosts_user on User {
  posts(first: 50) @connection(key: "UserPosts_posts") {
    edges @stream(initial_count: 10, label: "posts") {
      node {
        id
        ...PostCard_post
      }
    }
  }
}
```

This renders the first 10 immediately, then streams the rest.

## Common Pagination Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Duplicate items | Missing/wrong `id` field | Ensure `id` is selected on nodes |
| Load more not working | Missing `@connection` | Add directive with unique key |
| Data not merging | Different connection keys | Use same key for same connection |
| Stale after mutation | Connection not updated | Use `@appendEdge` or manual updater |
| Wrong order after prepend | Default appends to end | Use `@prependEdge` or `insertEdgeBefore` |
