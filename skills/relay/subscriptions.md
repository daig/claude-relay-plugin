# Relay Subscriptions

[← Back to SKILL.md](./SKILL.md)

## Basic Subscription with useSubscription

```typescript
import { graphql, useSubscription } from 'react-relay';
import type { GraphQLSubscriptionConfig } from 'relay-runtime';
import type { MessageListSubscription } from './__generated__/MessageListSubscription.graphql';

function MessageList({ channelId }: { channelId: string }) {
  const subscriptionConfig = useMemo<GraphQLSubscriptionConfig<MessageListSubscription>>(() => ({
    subscription: graphql`
      subscription MessageListSubscription($channelId: ID!) {
        messageCreated(channelId: $channelId) {
          id
          content
          createdAt
          author { id name }
        }
      }
    `,
    variables: { channelId },
    onError: (error) => {
      console.error('Subscription error:', error);
    },
  }), [channelId]);

  useSubscription(subscriptionConfig);

  // ... render messages
}
```

**Key points:**
- Wrap config in `useMemo` to prevent re-subscribing on every render
- Include `onError` handler for connection issues
- Subscription auto-unsubscribes when component unmounts

## Updating Connections with Declarative Directives

Just like mutations, subscriptions support `@appendNode`, `@prependNode`, and `@deleteEdge` directives for declarative store updates.

### Using @appendNode (Most Common)

When the subscription returns a node that should be added to a connection:

```typescript
const subscriptionConfig = useMemo<GraphQLSubscriptionConfig<ChannelSubscription>>(() => ({
  subscription: graphql`
    subscription ChannelSubscription($topic: String!, $connections: [ID!]!) {
      messageCreated(topic: $topic) {
        message @appendNode(connections: $connections, edgeTypeName: "MessagesEdge") {
          id
          content
          createdAt
          author { id name avatarUrl }
        }
      }
    }
  `,
  variables: {
    topic: `channel:${channelId}`,
    connections: connectionId ? [connectionId] : [],
  },
}), [channelId, connectionId]);

useSubscription(subscriptionConfig);
```

**Requirements:**
- Pass `connections` as a variable (array of connection IDs)
- Specify `edgeTypeName` matching your schema's edge type
- Relay automatically wraps the node in an edge and appends to connection

### Getting the Connection ID

Use `__id` in your query or pagination fragment:

```typescript
const { data } = usePaginationFragment(
  graphql`
    fragment ChannelMessages_channel on Channel
    @refetchable(queryName: "ChannelMessagesPaginationQuery") {
      messagesConnection(first: $count, after: $cursor)
        @connection(key: "ChannelMessages_messagesConnection") {
        __id  # This is the connection ID
        edges {
          node { id ...Message_message }
        }
      }
    }
  `,
  channelRef
);

const connectionId = data?.messagesConnection?.__id;
```

### Using @appendEdge

When the subscription returns an edge type (with `cursor` and `node`):

```typescript
graphql`
  subscription ChannelSubscription($topic: String!, $connections: [ID!]!) {
    messageCreated(topic: $topic) {
      messageEdge @appendEdge(connections: $connections) {
        cursor
        node {
          id
          content
          ...Message_message
        }
      }
    }
  }
`;
```

### Using @deleteEdge

For subscriptions that notify about deletions:

```typescript
graphql`
  subscription MessageDeletedSubscription($channelId: ID!, $connections: [ID!]!) {
    messageDeleted(channelId: $channelId) {
      deletedMessageId @deleteEdge(connections: $connections)
    }
  }
`;
```

## PostGraphile listen() Pattern

PostGraphile uses a generic `listen(topic)` subscription with `pg_notify`. The pattern:

```typescript
// Subscription using PostGraphile's listen + relatedNode
const ChannelSubscription = graphql`
  subscription ChannelViewSubscription($topic: String!, $connections: [ID!]!) {
    listen(topic: $topic) {
      relatedNode @appendNode(connections: $connections, edgeTypeName: "MessageChannelsEdge") {
        ... on MessageChannel {
          id
          postedAt
          message {
            id
            content
            user { id displayName avatarUrl }
          }
        }
      }
    }
  }
`;

// Usage - topic format depends on your pg_notify trigger
const subscriptionConfig = useMemo(() => ({
  subscription: ChannelSubscription,
  variables: {
    topic: `msg:${channelId}`,
    connections: connectionId ? [connectionId] : [],
  },
}), [channelId, connectionId]);
```

**How it works:**
1. PostgreSQL trigger calls `pg_notify('postgraphile:msg:<channelId>', payload)`
2. Payload includes `__node__` array for PostGraphile to resolve `relatedNode`
3. PostGraphile fetches the node data and returns it via WebSocket
4. Relay's `@appendNode` directive inserts it into the connection

## Avoiding Manual Store Updates

**Prefer declarative directives over manual updaters:**

```typescript
// ❌ Verbose: Manual updater
const subscriptionConfig = useMemo(() => ({
  subscription: ChannelSubscription,
  variables: { topic: `msg:${channelId}` },
  updater: (store) => {
    const payload = store.getRootField('listen');
    const newNode = payload?.getLinkedRecord('relatedNode');
    if (!newNode) return;

    const channel = store.get(channelStoreId);
    const connection = ConnectionHandler.getConnection(channel, 'ChannelView_messages');
    if (!connection) return;

    // Check for duplicates
    const edges = connection.getLinkedRecords('edges') || [];
    const exists = edges.some(e => e?.getLinkedRecord('node')?.getValue('id') === newNode.getValue('id'));
    if (exists) return;

    const newEdge = ConnectionHandler.createEdge(store, connection, newNode, 'MessagesEdge');
    ConnectionHandler.insertEdgeAfter(connection, newEdge);
  },
}), [channelId, channelStoreId]);

// ✅ Clean: Declarative directive
const subscriptionConfig = useMemo(() => ({
  subscription: graphql`
    subscription ChannelSubscription($topic: String!, $connections: [ID!]!) {
      listen(topic: $topic) {
        relatedNode @appendNode(connections: $connections, edgeTypeName: "MessagesEdge") {
          ... on Message { id content }
        }
      }
    }
  `,
  variables: {
    topic: `msg:${channelId}`,
    connections: connectionId ? [connectionId] : [],
  },
}), [channelId, connectionId]);
```

**When manual updaters ARE needed:**
- Complex conditional logic (add to multiple connections based on data)
- Updating fields not directly related to the subscription payload
- Custom deduplication beyond what Relay handles

## Combining Subscriptions with Pagination

When using `usePaginationFragment`, place the subscription in the same component:

```typescript
function ChannelMessages({ channel, channelId }: Props) {
  const { data, loadPrevious, hasPrevious } = usePaginationFragment(
    ChannelMessagesFragment,
    channel
  );

  const connectionId = data?.messagesConnection?.__id;

  // Subscription adds new messages to the same connection
  const subscriptionConfig = useMemo(() => ({
    subscription: ChannelSubscription,
    variables: {
      topic: `msg:${channelId}`,
      connections: connectionId ? [connectionId] : [],
    },
  }), [channelId, connectionId]);

  useSubscription(subscriptionConfig);

  return (
    <>
      {hasPrevious && <button onClick={() => loadPrevious(20)}>Load earlier</button>}
      {data?.messagesConnection?.edges?.map(({ node }) => (
        <Message key={node.id} message={node} />
      ))}
    </>
  );
}
```

## Handling Duplicates

When both mutations and subscriptions can add to the same connection, Relay handles duplicates automatically based on node `id`. However, ensure:

1. **Mutation uses `@appendEdge`/`@appendNode`** - So your own messages appear immediately
2. **Subscription uses same directive** - So others' messages appear via WebSocket
3. **Both select the same fields** - Relay merges based on `id`

```typescript
// Mutation for sending your own message
const SendMessageMutation = graphql`
  mutation SendMessageMutation($input: SendMessageInput!, $connections: [ID!]!) {
    sendMessage(input: $input) {
      messageEdge @appendEdge(connections: $connections) {
        node { id content createdAt author { id name } }
      }
    }
  }
`;

// Subscription for receiving others' messages
const MessageSubscription = graphql`
  subscription MessageSubscription($channelId: ID!, $connections: [ID!]!) {
    messageCreated(channelId: $channelId) {
      message @appendNode(connections: $connections, edgeTypeName: "MessagesEdge") {
        id content createdAt author { id name }
      }
    }
  }
`;
```

When you send a message:
1. Mutation's `@appendEdge` adds it to the connection immediately
2. Subscription receives the same message moments later
3. Relay sees the same `id` and merges (no duplicate)

## Subscription Configuration Options

```typescript
const config: GraphQLSubscriptionConfig<MySubscription> = {
  // Required
  subscription: MySubscriptionDocument,
  variables: { ... },

  // Store updates (prefer declarative directives instead)
  updater: (store, data) => { ... },

  // Callbacks
  onCompleted: () => { ... },  // When subscription ends normally
  onError: (error) => { ... }, // Connection or server errors
  onNext: (response) => { ... }, // Each subscription event (rarely needed)
};
```

## Best Practices

1. **Use declarative directives** - `@appendNode`/`@appendEdge` over manual updaters
2. **Memoize config** - Prevent re-subscribing on every render
3. **Handle errors** - WebSocket connections can drop
4. **Co-locate with connection** - Put subscription where you have `connectionId`
5. **Match mutation fields** - Subscription should select same fields as related mutations
6. **Use empty array fallback** - `connections: connectionId ? [connectionId] : []`
