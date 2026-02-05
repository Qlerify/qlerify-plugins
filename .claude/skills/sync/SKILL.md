---
name: sync
description: >
  Sync domain model with Qlerify. Use when the user says "sync domain model",
  "update Qlerify", "push changes to Qlerify", "sync schemas", "sync entities",
  or after implementing features that add or change entities, API endpoints,
  database schemas, migrations, or Prisma/GraphQL types.
allowed-tools: Glob, Grep, Read
---

# Sync Domain Model with Qlerify

Sync the local codebase's domain model with Qlerify. Detect entities, commands, and read models in code and ensure they match Qlerify.

## Step 1: Identify the workflow

1. Call `list_workflows` to get all accessible workflows.
2. Match by project name or ask the user which workflow to sync with.
3. Call `get_workflow_overview` to understand current state (lanes, bounded contexts, element counts).

## Step 2: Scan codebase for domain objects

Look for:

- **Entities**: Classes, interfaces, or schemas representing persistent domain objects (database models, ORM entities, TypeScript interfaces).
- **Commands**: Functions or DTOs for state-changing operations (`CreateOrder`, `UpdateCustomer`, POST/PUT/DELETE endpoints).
- **Read Models**: Query handlers or GET endpoints (`GetOrderById`, `ListProducts`).

Search patterns:
- Models: `src/domain/`, `src/models/`, `src/entities/`, `**/entity.ts`, `**/model.ts`
- Commands: `src/commands/`, `src/handlers/`, `**/command.ts`
- Queries: `src/queries/`, `src/read-models/`, `**/query.ts`
- API routes: `src/routes/`, `src/api/`, `src/controllers/`
- Schemas: `prisma/schema.prisma`, `**/schema.graphql`, `**/migrations/`

Also check git diff for recently changed schema files to detect field-level changes.

## Step 3: Compare and sync

1. Call `list_entities`, `list_commands`, and `list_read_models` from Qlerify.
2. Compare code vs Qlerify.
3. For differences:
   - **New in code**: Create in Qlerify with proper fields and example data.
   - **Changed fields**: Update Qlerify definition using `addFields`, `updateFields`, `removeFields`.
   - **Removed from code**: Ask user before deleting from Qlerify.

## Step 4: Report changes

Summarize:
- Entities created/updated/deleted
- Commands created/updated/deleted
- Read models created/updated/deleted
- Fields added/modified/removed
- Conflicts needing manual resolution

## Field type mapping

| Code Type                                     | Qlerify Type                        |
|-----------------------------------------------|-------------------------------------|
| `string`, `varchar`, `text`, `char`           | `string`                            |
| `number`, `int`, `float`, `decimal`, `bigint` | `number`                            |
| `boolean`, `bool`                             | `boolean`                           |
| Foreign key, relation, `@relation`            | `reference` (set `relatedEntityId`) |
| Nested object, JSON, `jsonb`                  | `object`                            |

## Guidelines

When creating/updating entities:
- Include 3 realistic example data values per field
- Mark primary keys with `primaryKey: true`
- Set `isRequired: true` for non-nullable fields
- Use `relatedEntityId` for foreign keys
- Set `cardinality` to `one-to-one` or `one-to-many` for references

When creating commands:
- Name with verbs: Create, Update, Delete, Submit, Cancel
- Mark auto-generated fields with `hideInForm: true`

When creating read models:
- Name with Get/List/Search prefixes
- Link to entity via `entityId`
- Mark filters with `isFilter: true`
