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

### Adding to a Connection

```typescript
// Mutation with @appendEdge
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
    connections: [connectionId], // Get this from ConnectionHandler
  },
});
```

### Getting Connection ID

```typescript
import { ConnectionHandler } from 'relay-runtime';

// In your component with the connection
const connectionId = ConnectionHandler.getConnectionID(
  userId,           // Parent record ID
  'UserPosts_posts' // Connection key from @connection directive
);
```

### Prepending to Connection

```typescript
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
4. **Handle connections explicitly** - Use `@appendEdge`/`@deleteEdge` or updaters
5. **Type your mutations** - Use generated types for variables and responses
6. **Handle errors gracefully** - Both network errors and business logic errors

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
