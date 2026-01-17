---
name: relay
description: >
  Relay GraphQL client for React/TypeScript applications. Use when working with
  Relay fragments, queries, mutations, or pagination. Covers useFragment,
  useLazyLoadQuery, usePreloadedQuery, useMutation, usePaginationFragment,
  @connection directive, optimistic updates, and TypeScript type generation.
  Apply when user mentions Relay, GraphQL fragments, data masking, or cursor-based
  pagination in React.
---

# Relay React/TypeScript Skill

## Quick Reference: Which Hook Do I Need?

| Scenario | Hook | Notes |
|----------|------|-------|
| Display data from parent component | `useFragment` | Most common - compose from parent's query |
| Load data at component mount | `useLazyLoadQuery` | Simple but triggers Suspense late |
| Load data before navigation | `usePreloadedQuery` + `useQueryLoader` | Preferred for routes |
| Paginated/infinite list | `usePaginationFragment` | Requires `@connection` directive |
| Search/filter with refetch | `useRefetchableFragment` | Fragment must have `@refetchable` |
| Create/Update/Delete | `useMutation` | With optimistic updates when needed |

## Core Principle: Fragment-First Development

Relay's power comes from **co-located data requirements**. Each component declares exactly what data it needs via a fragment.

```typescript
// UserCard.tsx
import { graphql, useFragment } from 'react-relay';
import type { UserCard_user$key } from './__generated__/UserCard_user.graphql';

interface Props {
  user: UserCard_user$key;
}

export function UserCard({ user }: Props) {
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

  return (
    <div>
      <img src={data.avatarUrl} alt={data.name} />
      <h3>{data.name}</h3>
    </div>
  );
}
```

**Key points:**
- Fragment name format: `ComponentName_propName`
- Import the generated `$key` type for the prop
- `useFragment` returns the resolved data with full TypeScript typing

## Composing Fragments in Queries

Parent components spread child fragments into their queries:

```typescript
// UserProfile.tsx
import { graphql, useLazyLoadQuery } from 'react-relay';
import type { UserProfileQuery } from './__generated__/UserProfileQuery.graphql';
import { UserCard } from './UserCard';
import { UserPosts } from './UserPosts';

export function UserProfile({ userId }: { userId: string }) {
  const data = useLazyLoadQuery<UserProfileQuery>(
    graphql`
      query UserProfileQuery($userId: ID!) {
        user(id: $userId) {
          ...UserCard_user
          ...UserPosts_user
        }
      }
    `,
    { userId }
  );

  if (!data.user) return <NotFound />;

  return (
    <div>
      <UserCard user={data.user} />
      <UserPosts user={data.user} />
    </div>
  );
}
```

## Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Fragment | `ComponentName_propName` | `UserCard_user` |
| Query | `ComponentNameQuery` | `UserProfileQuery` |
| Mutation | `ComponentNameMutationName` | `UserCardFollowMutation` |
| Subscription | `ComponentNameSubscription` | `MessageListSubscription` |

## Basic Mutation Pattern

```typescript
import { graphql, useMutation } from 'react-relay';
import type { UserCardFollowMutation } from './__generated__/UserCardFollowMutation.graphql';

function FollowButton({ userId }: { userId: string }) {
  const [commit, isInFlight] = useMutation<UserCardFollowMutation>(graphql`
    mutation UserCardFollowMutation($userId: ID!) {
      followUser(userId: $userId) {
        user {
          id
          isFollowedByViewer
          followerCount
        }
      }
    }
  `);

  const handleFollow = () => {
    commit({
      variables: { userId },
      optimisticResponse: {
        followUser: {
          user: {
            id: userId,
            isFollowedByViewer: true,
            followerCount: currentCount + 1, // from component state/props
          },
        },
      },
    });
  };

  return (
    <button onClick={handleFollow} disabled={isInFlight}>
      {isInFlight ? 'Following...' : 'Follow'}
    </button>
  );
}
```

## Suspense and Error Boundaries

Relay requires Suspense boundaries. Wrap data-fetching components:

```typescript
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<LoadingSpinner />}>
        <UserProfile userId="123" />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**Placement strategy:**
- One Suspense per distinct loading state the user should see
- Error boundaries at route level or around critical sections
- Nest Suspense for incremental loading (shell → content → details)

## Component Development Workflow

When creating a new data-connected component:

1. **Define the fragment** with exactly the fields needed
2. **Add TypeScript types** using the generated `$key` type
3. **Spread fragment** in parent query or fragment
4. **Run the compiler** (`relay-compiler` or via build)
5. **Import generated types** from `__generated__/` directory

When modifying data requirements:
1. Update the fragment
2. Run compiler
3. TypeScript will flag any breaking changes

## Relay Store Basics

The Relay store is a normalized cache keyed by `id`:

```
Store: {
  "user:123": { id: "user:123", name: "Alice", ... },
  "post:456": { id: "post:456", title: "Hello", author: { __ref: "user:123" } },
}
```

**Key behaviors:**
- Objects with `id` field are automatically normalized
- Mutations returning objects with `id` auto-update the store
- Connections (paginated lists) need explicit handling via `@connection`

## Anti-Patterns to Avoid

| Don't | Do Instead |
|-------|------------|
| Fetch all data in root query | Use fragments at each component |
| Pass raw query data down props | Pass fragment refs (`$key` types) |
| Refetch entire query for updates | Use mutations with proper `id` returns |
| Skip TypeScript types | Always use generated types |
| Inline GraphQL strings without `graphql` tag | Always use the `graphql` tagged template |
| Forget `@connection` on paginated fields | Add directive for pagination to work |

## Quick Debugging Checklist

**Data not updating after mutation?**
- [ ] Mutation returns object with `id` field
- [ ] Field names match exactly (case-sensitive)
- [ ] For connections: using `@appendEdge`/`@prependEdge` or updater function

**Fragment returns null?**
- [ ] Fragment is spread in parent query
- [ ] Parent passes correct prop (the fragment ref, not resolved data)
- [ ] Query actually fetches data (check Network tab)

**TypeScript errors?**
- [ ] Run `relay-compiler` after GraphQL changes
- [ ] Import types from `__generated__/` directory
- [ ] Use `$key` type for fragment ref props

## Supporting Documentation

| Topic | File | When to Use |
|-------|------|-------------|
| Fragment arguments, composition, data masking | [fragments.md](./fragments.md) | Complex fragment patterns |
| Query loading strategies, preloading, fetch policies | [queries.md](./queries.md) | Route-level data loading |
| Optimistic updates, updaters, connections | [mutations.md](./mutations.md) | Complex mutations |
| `@connection`, `usePaginationFragment`, cursors | [pagination.md](./pagination.md) | Lists and infinite scroll |
| Compiler config, generated types, aliasing | [typescript.md](./typescript.md) | Setup and type issues |
| Common errors, debugging tools | [troubleshooting.md](./troubleshooting.md) | When things break |

## Environment Setup Reference

Minimal `package.json` relay config:

```json
{
  "relay": {
    "src": "./src",
    "schema": "./schema.graphql",
    "language": "typescript",
    "artifactDirectory": "./src/__generated__"
  }
}
```

Run compiler:
```bash
# Watch mode during development
relay-compiler --watch

# Single run for CI/build
relay-compiler
```

## Essential Imports

```typescript
// Hooks
import {
  useFragment,
  useLazyLoadQuery,
  usePreloadedQuery,
  useQueryLoader,
  useMutation,
  usePaginationFragment,
  useRefetchableFragment,
} from 'react-relay';

// GraphQL tag
import { graphql } from 'react-relay';

// Types - always from __generated__ directory
import type { ComponentQuery } from './__generated__/ComponentQuery.graphql';
import type { Component_fragment$key } from './__generated__/Component_fragment.graphql';
```

## Common Directives Quick Reference

| Directive | Purpose | Example |
|-----------|---------|---------|
| `@refetchable` | Enable refetching a fragment | `@refetchable(queryName: "UserRefetchQuery")` |
| `@connection` | Mark a paginated field | `@connection(key: "UserPosts_posts")` |
| `@argumentDefinitions` | Define fragment variables | `@argumentDefinitions(count: {type: "Int", defaultValue: 10})` |
| `@arguments` | Pass values to fragment | `...UserPosts_user @arguments(count: 5)` |
| `@appendEdge` | Add to connection end | `@appendEdge(connections: $connections)` |
| `@prependEdge` | Add to connection start | `@prependEdge(connections: $connections)` |
| `@deleteEdge` | Remove from connection | `@deleteEdge(connections: $connections)` |
| `@deleteRecord` | Remove from store entirely | `@deleteRecord` |
| `@inline` | Break data masking | `@inline` on fragment |
