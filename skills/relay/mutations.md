# Relay Mutations

[← Back to SKILL.md](./SKILL.md)

## Basic Mutation with useMutation

```typescript
import { graphql, useMutation } from 'react-relay';
import type { LikePostMutation } from './__generated__/LikePostMutation.graphql';

function LikeButton({ postId }: { postId: string }) {
  const [commit, isInFlight] = useMutation<LikePostMutation>(graphql`
    mutation LikePostMutation($postId: ID!) {
      likePost(postId: $postId) {
        post {
          id
          likeCount
          isLikedByViewer
        }
      }
    }
  `);

  const handleLike = () => {
    commit({
      variables: { postId },
      onCompleted: (response) => {
        console.log('Liked!', response.likePost?.post?.likeCount);
      },
      onError: (error) => {
        console.error('Failed:', error);
      },
    });
  };

  return (
    <button onClick={handleLike} disabled={isInFlight}>
      {isInFlight ? 'Liking...' : 'Like'}
    </button>
  );
}
```

## Optimistic Updates with optimisticResponse

Immediately update the UI before the server responds:

```typescript
commit({
  variables: { postId },
  optimisticResponse: {
    likePost: {
      post: {
        id: postId,
        likeCount: currentLikeCount + 1,
        isLikedByViewer: true,
      },
    },
  },
});
```

**Requirements for optimisticResponse:**
- Must match the shape of mutation response exactly
- Include all selected fields (even if unchanged)
- Objects must include `id` for store matching
- If server response differs, it automatically replaces optimistic data

### Complete Optimistic Example

```typescript
function FollowButton({ user }: { user: FollowButton_user$key }) {
  const data = useFragment(
    graphql`
      fragment FollowButton_user on User {
        id
        isFollowedByViewer
        followerCount
      }
    `,
    user
  );

  const [commit, isInFlight] = useMutation<FollowUserMutation>(graphql`
    mutation FollowUserMutation($userId: ID!) {
      followUser(userId: $userId) {
        user {
          id
          isFollowedByViewer
          followerCount
        }
      }
    }
  `);

  const handleToggle = () => {
    const willFollow = !data.isFollowedByViewer;

    commit({
      variables: { userId: data.id },
      optimisticResponse: {
        followUser: {
          user: {
            id: data.id,
            isFollowedByViewer: willFollow,
            followerCount: data.followerCount + (willFollow ? 1 : -1),
          },
        },
      },
    });
  };

  return (
    <button onClick={handleToggle} disabled={isInFlight}>
      {data.isFollowedByViewer ? 'Unfollow' : 'Follow'}
    </button>
  );
}
```

## Optimistic Updates with optimisticUpdater

For complex updates that can't be expressed as a simple response:

```typescript
commit({
  variables: { postId },
  optimisticUpdater: (store) => {
    const post = store.get(postId);
    if (post) {
      const currentCount = post.getValue('likeCount') as number;
      post.setValue(currentCount + 1, 'likeCount');
      post.setValue(true, 'isLikedByViewer');
    }
  },
});
```

## Server-Side Updates with updater

Runs after server responds. Use for updates the server doesn't return:

```typescript
commit({
  variables: { postId },
  updater: (store, response) => {
    // response contains the mutation result
    const newLikeCount = response.likePost?.post?.likeCount;

    // Additional store updates beyond what mutation returns
    const viewer = store.getRoot().getLinkedRecord('viewer');
    if (viewer) {
      const likedPosts = viewer.getValue('likedPostsCount') as number;
      viewer.setValue(likedPosts + 1, 'likedPostsCount');
    }
  },
});
```

## Connection Mutations with Declarative Directives

### Choosing Between @appendEdge and @appendNode

Relay provides two sets of directives for adding items to connections:

| Directive | Use when server returns | Required args |
|-----------|------------------------|---------------|
| `@appendEdge` / `@prependEdge` | An **edge** type (has `cursor` + `node`) | `connections` |
| `@appendNode` / `@prependNode` | A **node** directly | `connections` + `edgeTypeName` |

**How to decide:**
1. Check your schema - does the mutation payload have an `*Edge` field? Use `@appendEdge`
2. Does it only return the node type directly? Use `@appendNode` with `edgeTypeName`

Most GraphQL servers with Relay support (PostGraphile, Hasura, Prisma) return edge types, so prefer `@appendEdge`.

### Adding with @appendEdge (Preferred)

Use when the mutation returns an edge type:

```typescript
graphql`
  mutation CreatePostMutation($input: CreatePostInput!, $connections: [ID!]!) {
    createPost(input: $input) {
      postEdge @appendEdge(connections: $connections) {
        cursor
        node {
          id
          title
          ...PostCard_post
        }
      }
    }
  }
`;

// Usage
commit({
  variables: {
    input: { title, content },
    connections: [connectionId],
  },
});
```

### Adding with @appendNode

Use when the mutation returns a node and you need Relay to create the edge wrapper:

```typescript
graphql`
  mutation CreatePostMutation($input: CreatePostInput!, $connections: [ID!]!) {
    createPost(input: $input) {
      post @appendNode(connections: $connections, edgeTypeName: "PostEdge") {
        id
        title
        ...PostCard_post
      }
    }
  }
`;
```

The `edgeTypeName` must match your schema's edge type name exactly.

### Getting Connection ID

Two approaches:

**Option 1: Using `__id` in your query (simpler)**
```typescript
const data = useLazyLoadQuery(graphql`
  query UserPostsQuery($userId: ID!) {
    user(id: $userId) {
      postsConnection {
        __id  # Relay provides this automatically
        edges { node { id ...PostCard_post } }
      }
    }
  }
`, { userId });

// Use directly
const connectionId = data.user?.postsConnection?.__id;
```

**Option 2: Using ConnectionHandler (when you don't have query access)**
```typescript
import { ConnectionHandler } from 'relay-runtime';

const connectionId = ConnectionHandler.getConnectionID(
  userId,           // Parent record ID
  'UserPosts_posts' // Connection key from @connection directive
);
```

### Architectural Pattern: Sibling Components Sharing a Connection

When one component displays a connection and a sibling component mutates it, **lift the query to their shared parent**:

```
// ❌ Anti-pattern: Sibling components each managing connection separately
ParentComponent
├── MessageList (queries messages, has @connection)
└── MessageComposer (queries same connection for __id, does mutation)
    // Problem: Two queries for same data, complex ID passing via callbacks

// ✅ Better: Parent owns the query, children receive what they need
ParentComponent (owns query with @connection, passes data down)
├── MessageList (receives messages as props - pure render)
└── MessageComposer (receives connectionId as prop - does mutation)
```

**Implementation:**

```typescript
// Parent owns the query and subscription
function ChannelView({ channelId }) {
  const data = useLazyLoadQuery(graphql`
    query ChannelViewQuery($channelId: ID!) {
      channel(id: $channelId) {
        messagesConnection @connection(key: "ChannelView_messages") {
          __id
          edges { node { id ...MessageList_message } }
        }
      }
    }
  `, { channelId });

  const connectionId = data.channel?.messagesConnection?.__id;

  return (
    <>
      <MessageList messages={data.channel?.messagesConnection} />
      <MessageComposer connectionId={connectionId} channelId={channelId} />
    </>
  );
}

// Child just renders - no Relay hooks needed
function MessageList({ messages }) {
  return messages?.edges?.map(({ node }) => <Message key={node.id} message={node} />);
}

// Child just mutates - receives connectionId as prop
function MessageComposer({ connectionId, channelId }) {
  const [commit] = useMutation(graphql`
    mutation CreateMessageMutation($input: CreateMessageInput!, $connections: [ID!]!) {
      createMessage(input: $input) {
        messageEdge @appendEdge(connections: $connections) {
          node { id ...MessageList_message }
        }
      }
    }
  `);

  const send = (content) => {
    commit({ variables: { input: { channelId, content }, connections: [connectionId] } });
  };
  // ...
}
```

**Benefits:**
- Single source of truth for the connection
- No callback prop drilling for connection IDs
- Simpler child components (display component has no Relay hooks)
- Subscription logic lives with the query it refreshes

### Prepending to Connection

```typescript
// With edge type
graphql`
  mutation CreatePostMutation($input: CreatePostInput!, $connections: [ID!]!) {
    createPost(input: $input) {
      postEdge @prependEdge(connections: $connections) {
        node {
          id
          ...PostCard_post
        }
      }
    }
  }
`;

// With node type
graphql`
  mutation CreatePostMutation($input: CreatePostInput!, $connections: [ID!]!) {
    createPost(input: $input) {
      post @prependNode(connections: $connections, edgeTypeName: "PostEdge") {
        id
        ...PostCard_post
      }
    }
  }
`;
```

### Deleting from Connection

```typescript
graphql`
  mutation DeletePostMutation($postId: ID!, $connections: [ID!]!) {
    deletePost(postId: $postId) {
      deletedPostId @deleteEdge(connections: $connections)
    }
  }
`;
```

### Deleting Record Entirely

```typescript
graphql`
  mutation DeletePostMutation($postId: ID!) {
    deletePost(postId: $postId) {
      deletedPost {
        id @deleteRecord
      }
    }
  }
`;
```

## Manual Connection Updates with ConnectionHandler

When declarative directives aren't enough:

```typescript
import { ConnectionHandler } from 'relay-runtime';

commit({
  variables: { input: { title, content } },
  updater: (store) => {
    // Get the new post from mutation payload
    const payload = store.getRootField('createPost');
    const newEdge = payload?.getLinkedRecord('postEdge');
    if (!newEdge) return;

    // Get the connection
    const user = store.get(userId);
    const connection = ConnectionHandler.getConnection(
      user!,
      'UserPosts_posts', // @connection key
      { orderBy: 'CREATED_AT' } // connection arguments if any
    );
    if (!connection) return;

    // Insert at beginning
    ConnectionHandler.insertEdgeBefore(connection, newEdge);

    // Or insert at end
    // ConnectionHandler.insertEdgeAfter(connection, newEdge);
  },
  optimisticUpdater: (store) => {
    // Create optimistic records
    const id = `client:newPost:${Date.now()}`;
    const node = store.create(id, 'Post');
    node.setValue(id, 'id');
    node.setValue(title, 'title');
    node.setValue(content, 'content');
    node.setValue(new Date().toISOString(), 'createdAt');

    const newEdge = ConnectionHandler.createEdge(
      store,
      connection!,
      node,
      'PostEdge' // Edge type name
    );

    ConnectionHandler.insertEdgeBefore(connection!, newEdge);
  },
});
```

## Mutation Configuration Options

```typescript
commit({
  // Required
  variables: { ... },

  // Optimistic updates
  optimisticResponse: { ... },
  optimisticUpdater: (store) => { ... },

  // Post-response updates
  updater: (store, response) => { ... },

  // Callbacks
  onCompleted: (response, errors) => { ... },
  onError: (error) => { ... },
  onUnsubscribe: () => { ... },

  // Upload config (for file uploads)
  uploadables: { file: fileObject },
});
```

## TypeScript Types for Mutations

```typescript
import type {
  CreatePostMutation,
  CreatePostMutation$data,
  CreatePostMutation$variables,
} from './__generated__/CreatePostMutation.graphql';

// $variables type for building inputs
const variables: CreatePostMutation$variables = {
  input: { title: 'Hello', content: 'World' },
  connections: [connectionId],
};

// $data type for handling response
const handleComplete = (response: CreatePostMutation$data) => {
  const newPost = response.createPost?.postEdge?.node;
  if (newPost) {
    console.log('Created:', newPost.title);
  }
};
```

## Mutation Best Practices

1. **Return updated fields** - Include all fields that might change in mutation response
2. **Return object IDs** - Enables automatic store updates
3. **Use optimistic updates** - For instant UI feedback
4. **Handle connections explicitly** - Use `@appendEdge`/`@appendNode`/`@deleteEdge` or updaters
5. **Choose the right directive** - `@appendEdge` when server returns edge, `@appendNode` when it returns node
6. **Type your mutations** - Use generated types for variables and responses
7. **Handle errors gracefully** - Both network errors and business logic errors

## Common Mutation Patterns

### Create with Redirect

```typescript
const [commit] = useMutation(createPostMutation);

const handleCreate = () => {
  commit({
    variables: { input },
    onCompleted: (response) => {
      const newId = response.createPost?.post?.id;
      if (newId) {
        navigate(`/post/${newId}`);
      }
    },
  });
};
```

### Update with Current Values

```typescript
commit({
  variables: {
    input: {
      id: postId,
      title: newTitle,
      // Keep other fields
      content: currentContent,
    },
  },
});
```

### Delete with Confirmation

```typescript
const handleDelete = async () => {
  const confirmed = await showConfirmDialog('Delete this post?');
  if (!confirmed) return;

  commit({
    variables: { postId, connections: [connectionId] },
    optimisticResponse: {
      deletePost: {
        deletedPostId: postId,
      },
    },
    onError: () => {
      showToast('Failed to delete post');
    },
  });
};
```

### Form Submission

```typescript
function PostForm() {
  const [commit, isInFlight] = useMutation(createPostMutation);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    setError(null);

    commit({
      variables: { input: formData },
      onCompleted: (_, errors) => {
        if (errors?.length) {
          setError(errors[0].message);
        } else {
          resetForm();
        }
      },
      onError: (err) => {
        setError(err.message);
      },
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {error && <ErrorMessage>{error}</ErrorMessage>}
      {/* form fields */}
      <button type="submit" disabled={isInFlight}>
        {isInFlight ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```
