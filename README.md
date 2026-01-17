# Relay GraphQL Plugin for Claude Code

A Claude Code plugin that teaches Claude how to work effectively with [Relay](https://relay.dev/) (Facebook's GraphQL client) in React/TypeScript applications.

## Features

- **Decision Trees**: Quickly determine which Relay hook to use for your use case
- **Fragment Patterns**: Co-located data requirements, composition, and data masking
- **Query Strategies**: Lazy loading, preloading, and Suspense integration
- **Mutations**: Optimistic updates, store manipulation, and connection handling
- **Pagination**: Cursor-based pagination with `@connection` and infinite scroll
- **TypeScript**: Full type safety with generated types and proper configuration
- **Troubleshooting**: Common issues and debugging techniques

## Installation

### Option 1: Direct Install (if published to a marketplace)

```bash
/plugin install relay-graphql@your-marketplace
```

### Option 2: From GitHub Repository

First, add the marketplace:

```bash
/plugin marketplace add yourusername/claude-relay-plugin
```

Then install the plugin:

```bash
/plugin install relay-graphql
```

### Option 3: Local Development

```bash
claude --plugin-dir /path/to/claude-relay-plugin
```

## Usage

Once installed, Claude will automatically activate this skill when you ask about:

- Relay fragments, queries, or mutations
- GraphQL data fetching in React
- Cursor-based pagination
- Optimistic updates
- Data masking
- TypeScript types for GraphQL

### Example Prompts

```
How do I create a paginated list with Relay?
```

```
Write a mutation with optimistic updates for liking a post
```

```
Set up a fragment for a UserCard component with TypeScript
```

```
My mutation completes but the UI doesn't update. What's wrong?
```

## Skill Contents

| File | Description |
|------|-------------|
| `SKILL.md` | Main entry point with decision trees and essential patterns |
| `fragments.md` | Fragment arguments, composition, and data masking |
| `queries.md` | Query loading strategies and Suspense integration |
| `mutations.md` | Optimistic updates and connection manipulation |
| `pagination.md` | `@connection` directive and pagination hooks |
| `typescript.md` | Compiler configuration and type patterns |
| `troubleshooting.md` | Common issues and debugging guide |

## Requirements

- Claude Code CLI
- A React/TypeScript project using Relay

## License

MIT
