---
name: sync
description: >
  This skill should be used when the user asks to "sync domain model",
  "update Qlerify", "push changes to Qlerify", "sync schemas", "sync entities",
  "sync domain events", or after implementing features that add or change entities,
  API endpoints, domain events, database schemas, migrations, or Prisma/GraphQL
  types. For brownfield/legacy codebases with unclear boundaries, first extract
  one aggregate at a time (use the extract-aggregate skill) before broad
  sync operations. Syncs the local codebase's domain model with Qlerify.
allowed-tools: Glob, Grep, Read
---

# Sync Domain Model with Qlerify

Sync the local codebase's domain model with Qlerify. Detect entities, commands, read models, and domain event schemas in code and ensure they match Qlerify.

If this is a brownfield or legacy codebase and the aggregate boundaries are unclear, first
extract one aggregate at a time (use the extract-aggregate skill), then run sync.

## Step 1: Identify the workflow

1. Call `list_workflows` to get all accessible workflows.
2. Match by project name or ask the user which workflow to sync with.
3. Call `get_workflow` to understand current state (events, entities, commands, read models, domain event schemas, bounded contexts).

## Step 2: Scan codebase for domain objects

Scan for the following:

- **Entities**: Classes, interfaces, or schemas representing persistent domain objects (database models, ORM entities,
  TypeScript interfaces).
- **Commands**: Functions or DTOs for state-changing operations (`CreateOrder`, `UpdateCustomer`, POST/PUT/DELETE
  endpoints).
- **Read Models**: Query handlers or GET endpoints (`GetOrderById`, `ListProducts`).
- **Domain Event Schemas**: Event classes, records, or DTOs representing published event payloads (`OrderPlacedEvent`, `PaymentConfirmedEvent`).

Search patterns:
- Models: `src/domain/`, `src/models/`, `src/entities/`, `**/entity.ts`, `**/model.ts`
- Commands: `src/commands/`, `src/handlers/`, `**/command.ts`
- Queries: `src/queries/`, `src/read-models/`, `**/query.ts`
- Events: `src/events/`, `src/domain-events/`, `**/event.ts`, `**/*Event.ts`, `**/*Event.java`
- API routes: `src/routes/`, `src/api/`, `src/controllers/`
- Schemas: `prisma/schema.prisma`, `**/schema.graphql`, `**/migrations/`

Also check git diff for recently changed schema files to detect field-level changes.

## Step 3: Compare and sync

1. Call `get_workflow` to get all entities, commands, read models, and domain event schemas from Qlerify.
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
- Domain event schemas created/updated/deleted
- Fields added/modified/removed
- Conflicts needing manual resolution

## Field type mapping

| Code Type                                     | Qlerify Type                          |
|-----------------------------------------------|---------------------------------------|
| `string`, `varchar`, `text`, `char`           | `string`                              |
| `number`, `int`, `float`, `decimal`, `bigint` | `number`                              |
| `boolean`, `bool`                             | `boolean`                             |
| Foreign key, relation, `@relation`            | Set `relatedEntity` + `cardinality`   |
| Nested object, JSON, `jsonb`                  | `object`                              |

## Guidelines

When creating/updating entities:
- Include 3 realistic example data values per field
- Set `isRequired: true` for non-nullable fields
- Use `relatedEntity` ($ref path) and `cardinality` to express entity relationships from the owning entity's perspective

When creating commands:
- Name with verbs: Create, Update, Delete, Submit, Cancel
- Fields should correspond to fields on the aggregate root entity they modify
- Mark auto-generated fields with `hideInForm: true`
- Use nested `fields` with `relatedEntity` ($ref path) to reference related entities

When creating read models:
- Name with Get/List/Search prefixes
- Link to entity via `entity` ($ref path)
- Fields represent both inputs and outputs: set `isFilter: true` for query parameters, omit for returned data fields
- Use nested `fields` with `relatedEntity` ($ref path) for return fields that reference other entities

When creating domain event schemas:
- Name matching the event (e.g., "Order Placed", "Payment Confirmed")
- Link to the aggregate root entity via `entity` ($ref path)
- Include identifier fields (e.g., `orderId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `confirmedAt`)
- Capture essential facts about the state change, not the full entity state
- Use nested `fields` with `relatedEntity` ($ref path) for embedded event data
