# GraphQL & Relay Plugin for Claude Code

A Claude Code plugin containing two complementary skills for GraphQL development:

1. **GraphQL Fundamentals** - Core GraphQL concepts, type system, SDL, and codegen
2. **Relay Patterns** - Facebook's Relay client for React/TypeScript applications

## Skills Overview

### GraphQL Skill

Covers foundational GraphQL concepts that are prerequisites for any GraphQL work:

- **Type System**: Scalars, objects, interfaces, unions, enums, input types
- **Operations**: Queries, mutations, subscriptions, fragments, variables
- **Schema Design**: SDL syntax, documentation, directives, evolution strategies
- **Nullability**: Null semantics, error handling, propagation rules
- **TypeScript**: GraphQL Code Generator setup and configuration

### Relay Skill

Covers Relay-specific patterns for React/TypeScript applications:

- **Fragments**: Co-located data requirements, composition, data masking
- **Queries**: Lazy loading, preloading, Suspense integration
- **Mutations**: Optimistic updates, store manipulation, connection handling
- **Pagination**: Cursor-based pagination with `@connection` and infinite scroll
- **TypeScript**: Relay compiler configuration and type patterns
- **Troubleshooting**: Common issues and debugging techniques

## Installation

### Option 1: From GitHub Repository

First, add the marketplace:

```bash
/plugin marketplace add yourusername/claude-relay-plugin
```

Then install the plugin:

```bash
/plugin install relay-graphql
```

### Option 2: Local Development

```bash
claude --plugin-dir /path/to/claude-relay-plugin
```

## Usage

Claude will automatically activate the appropriate skill based on your questions.

### GraphQL Skill Triggers

- GraphQL type system (interfaces, unions, enums)
- Schema Definition Language (SDL)
- Nullability and error handling
- GraphQL Code Generator
- General query/mutation writing

### Relay Skill Triggers

- Relay fragments and data masking
- Relay hooks (useFragment, usePaginationFragment, etc.)
- Relay-specific directives (@connection, @argumentDefinitions)
- Optimistic updates and store manipulation
- Relay compiler configuration

### Example Prompts

**GraphQL Fundamentals:**
```
What's the difference between interface and union types in GraphQL?
Explain GraphQL nullability and the ! modifier
Set up GraphQL Code Generator for TypeScript
```

**Relay Patterns:**
```
How do I create a paginated list with Relay?
Write a mutation with optimistic updates for liking a post
My mutation completes but the UI doesn't update. What's wrong?
```

## Skill Contents

### GraphQL Skill (`skills/graphql/`)

| File | Description |
|------|-------------|
| `SKILL.md` | Quick syntax reference, directives, field selection |
| `types.md` | Complete type system: scalars, objects, interfaces, unions, enums |
| `operations.md` | Queries, mutations, subscriptions, fragments, variables |
| `schema.md` | SDL syntax, documentation, schema evolution |
| `nullability.md` | Null semantics, propagation rules, error patterns |
| `typescript.md` | GraphQL Code Generator setup |

### Relay Skill (`skills/relay/`)

| File | Description |
|------|-------------|
| `SKILL.md` | Decision trees and essential Relay patterns |
| `fragments.md` | Fragment arguments, composition, data masking |
| `queries.md` | Query loading strategies and Suspense integration |
| `mutations.md` | Optimistic updates and connection manipulation |
| `pagination.md` | `@connection` directive and pagination hooks |
| `typescript.md` | Compiler configuration and type patterns |
| `troubleshooting.md` | Common issues and debugging guide |

## Skill Relationship

The two skills are designed to be **orthogonal**:

- **GraphQL skill** covers concepts that Relay assumes you already know
- **Relay skill** covers Relay-specific patterns and APIs

Together they provide complete coverage for GraphQL + Relay development.

## Requirements

- Claude Code CLI
- For Relay skill: A React/TypeScript project using Relay

## License

MIT
