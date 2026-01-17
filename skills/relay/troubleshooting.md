# Relay Troubleshooting

[← Back to SKILL.md](./SKILL.md)

## Data Not Updating After Mutation

### Checklist

- [ ] **Mutation returns `id` field** - Objects without `id` can't be matched in store
- [ ] **Field names match exactly** - Case-sensitive, check for typos
- [ ] **Returning all changed fields** - Only returned fields update in store
- [ ] **Connection updates** - Using `@appendEdge`/`@deleteEdge` or updater function
- [ ] **Check Network tab** - Verify mutation response contains expected data

### Common Fixes

```typescript
// ❌ Missing id - won't update store
mutation {
  updateUser(input: $input) {
    user {
      name
    }
  }
}

// ✅ Include id for store matching
mutation {
  updateUser(input: $input) {
    user {
      id
      name
    }
  }
}
```

```typescript
// ❌ Connection not updated
mutation CreatePostMutation($input: CreatePostInput!) {
  createPost(input: $input) {
    post { id title }
  }
}

// ✅ Use declarative connection update
mutation CreatePostMutation($input: CreatePostInput!, $connections: [ID!]!) {
  createPost(input: $input) {
    postEdge @appendEdge(connections: $connections) {
      node { id title }
    }
  }
}
```

## Fragment Data is Null

### Checklist

- [ ] **Fragment spread in parent** - `...FragmentName` in parent query/fragment
- [ ] **Parent fetches data** - Check Network tab for query
- [ ] **Correct prop passed** - Fragment ref, not resolved data
- [ ] **Field exists in schema** - Check schema for field availability
- [ ] **Access control** - User might not have permission to view data

### Common Fixes

```typescript
// ❌ Fragment not spread
query UserQuery($id: ID!) {
  user(id: $id) {
    name
  }
}
<UserCard user={data.user} />  // UserCard_user not included!

// ✅ Spread the fragment
query UserQuery($id: ID!) {
  user(id: $id) {
    ...UserCard_user
  }
}
```

```typescript
// ❌ Passing resolved data instead of ref
const data = useFragment(fragment, user);
<ChildComponent user={data} />  // Wrong!

// ✅ Pass the ref, spread child fragment
// In parent:
graphql`
  fragment Parent_user on User {
    ...ChildComponent_user
  }
`;
<ChildComponent user={data} />  // data is still the ref
```

## Compiler Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Unknown fragment` | Fragment not defined or typo | Check spelling, ensure file is saved |
| `Unknown type` | Type not in schema | Update schema, check spelling |
| `Duplicate document` | Same name used twice | Rename one of the duplicates |
| `Cannot spread fragment X on type Y` | Type mismatch | Check the `on Type` clause |
| `Variable $x is not defined` | Missing variable | Add to query/fragment arguments |
| `Circular dependency` | Fragments reference each other | Restructure component hierarchy |

### Fix: Schema Out of Sync

```bash
# Re-fetch schema from server
npx get-graphql-schema http://localhost:4000/graphql > schema.graphql

# Or if using introspection script
npm run update-schema
```

## Network/Suspense Issues

### Infinite Loading

- [ ] **Error boundary present** - Errors without boundary cause infinite suspend
- [ ] **Check console for errors** - Network or GraphQL errors
- [ ] **Verify endpoint** - Server running and accessible

```typescript
// ✅ Always wrap with error boundary
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

### Query Fires Multiple Times

- [ ] **Check component re-renders** - Use React DevTools
- [ ] **Variables changing** - New object reference triggers refetch
- [ ] **Missing useMemo/useCallback** - Stabilize variable objects

```typescript
// ❌ New object every render
<Component query={useLazyLoadQuery(query, { filter: { type: 'all' } })} />

// ✅ Stable reference
const variables = useMemo(() => ({ filter: { type: 'all' } }), []);
<Component query={useLazyLoadQuery(query, variables)} />
```

### Network Errors

```typescript
// Custom network error handling in environment
const network = Network.create(async (request, variables) => {
  const response = await fetch('/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: request.text, variables }),
  });

  if (!response.ok) {
    throw new Error(`Network error: ${response.status}`);
  }

  const json = await response.json();

  if (json.errors) {
    console.error('GraphQL errors:', json.errors);
  }

  return json;
});
```

## Performance Issues

### Large Lists Slow

- [ ] **Use pagination** - Don't fetch all items at once
- [ ] **Virtualize rendering** - Use react-window or react-virtualized
- [ ] **Check fragment size** - Fetching too many fields per item?

### Unnecessary Re-renders

- [ ] **Data masking working** - Only changed fragments should re-render
- [ ] **memo components** - Wrap with React.memo if needed
- [ ] **Check useFragment usage** - Each component should have its own fragment

```typescript
// ✅ Each component reads only its data
const UserCard = memo(function UserCard({ user }: Props) {
  const data = useFragment(fragment, user);
  // Only re-renders when UserCard_user data changes
  return <div>{data.name}</div>;
});
```

### Store Getting Large

```typescript
// Garbage collect old data
environment.getStore().scheduleGC();

// Or configure retention
const store = new Store(new RecordSource(), {
  gcReleaseBufferSize: 10,
});
```

## Debugging Tools

### Relay DevTools

Install browser extension for Chrome/Firefox:
- View store contents
- Track queries and mutations
- Inspect component data requirements

### Network Logging

```typescript
const network = Network.create(async (request, variables) => {
  console.group(`Relay: ${request.name}`);
  console.log('Variables:', variables);

  const response = await fetchGraphQL(request, variables);

  console.log('Response:', response);
  console.groupEnd();

  return response;
});
```

### Store Inspection

```typescript
// In development, expose store
if (process.env.NODE_ENV === 'development') {
  (window as any).__RELAY_STORE__ = environment.getStore();
}

// Then in console:
__RELAY_STORE__.getSource().toJSON()
```

### Component Data Logging

```typescript
function DebugFragment({ fragmentRef }: Props) {
  const data = useFragment(fragment, fragmentRef);

  useEffect(() => {
    console.log('Fragment data:', data);
  }, [data]);

  return <ActualComponent data={data} />;
}
```

## Type Errors After Schema Change

1. Update local schema file
2. Run `relay-compiler`
3. Fix any TypeScript errors from changed types
4. If types still wrong, delete `__generated__` and re-run compiler

```bash
rm -rf src/__generated__
npx relay-compiler
```

## Mutation Not Working

### Checklist

- [ ] **Mutation name matches** - Server expects exact operation name
- [ ] **Input type correct** - Check required vs optional fields
- [ ] **Auth token sent** - Check headers in Network tab
- [ ] **Server returns data** - Not just `{ data: null }`

### Debug Mutation

```typescript
commit({
  variables,
  onCompleted: (response, errors) => {
    console.log('Mutation completed:', response);
    if (errors) {
      console.error('Mutation errors:', errors);
    }
  },
  onError: (error) => {
    console.error('Mutation failed:', error);
  },
});
```
