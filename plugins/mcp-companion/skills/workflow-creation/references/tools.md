# Qlerify MCP Tool Reference

Natural-language descriptions of every Qlerify MCP tool, organized by category.
Each entry describes what the tool does, when to use it, and key parameters.

All tools that reference workflow elements use `$ref` paths from the `get_workflow` specification
(e.g., `#/domainEvents/OrderPlaced`, `#/schemas/entities/Order`).

---

## Workflow Tools

### list_workflows
Browse all workflows accessible to the current API key. Returns workflow names, IDs, and
project IDs. Supports pagination for accounts with many workflows. Use this as the starting
point to find the right workflow before doing anything else.

### create_workflow
Create a brand-new empty workflow inside a project. Provide a descriptive name — this appears
in the UI and helps users find the workflow later. Returns the new workflow ID.

- `projectId` — The project to create the workflow in
- `name` — A descriptive name for the workflow

### get_workflow
Retrieve a clean, DDD-friendly specification of the workflow. Returns domain events with semantic
fields (command, domainEventSchema, aggregateRoot, readModels, acceptanceCriteria), and schemas
organized by type (entities, commands, queries, domainEvents, valueObjects). Each element includes
a `$ref` path for use with other tools. This is the **primary tool for viewing existing data** —
there are no separate list tools for lanes, groups, events, entities, commands, or read models.

The returned specification includes a `$schema` URL pointing to a JSON Schema that describes the
full structure. Fetch this URL to understand field types, allowed values, and relationships.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Optional. Bounded context name to export. If omitted, uses the first bounded context.

### generate_openapi_spec
Generate an OpenAPI/Swagger YAML specification from the workflow's domain model. Requires at least
one bounded context with entities. Useful for bootstrapping API implementations from the domain model.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Name of the bounded context to generate the spec for

---

## Lane Tools

Lanes are the horizontal swim lanes in the BPMN diagram. Each lane represents a **real-world actor
role** (person) or **Automation** (system-triggered actions). Do NOT create lanes for internal
services or technical components. Use `get_workflow` to see existing lanes.

### create_lane

Add a new swim lane to the workflow. Name it after the actor role or "Automation" for system actions.

- `workflowId`, `projectId` — Identifies the workflow
- `name` — The lane name (e.g., "Customer", "Hotel Staff", "Automation"). NOT service names like "Payment Service"

### update_lane
Rename an existing lane. Does not affect events already in the lane.

- `workflowId`, `projectId` — Identifies the workflow
- `lane` — Current lane name
- `name` — The new name

### delete_lane
Remove a lane from the workflow. The lane must be empty — move or delete all events in it first.

- `workflowId`, `projectId` — Identifies the workflow
- `lane` — Lane name to delete

---

## Group Tools

Groups are the vertical columns in the BPMN diagram. Each group represents a phase, stage,
or logical section of the business process. Use `get_workflow` to see existing groups.

### create_group
Add a new phase/stage column to the workflow. Groups are positioned automatically — just
create them in the order they should appear left-to-right.

- `workflowId`, `projectId` — Identifies the workflow
- `name` — The group name (e.g., "Checkout", "Fulfillment")

### update_group
Rename an existing group.

- `workflowId`, `projectId` — Identifies the workflow
- `group` — Current group name
- `name` — The new name

### delete_group
Remove a group. The group must be empty — move or delete all events in it first.

- `workflowId`, `projectId` — Identifies the workflow
- `group` — Group name to delete

---

## Domain Event Tools

Domain events are the core elements of the BPMN diagram — they represent things that happen
in the business process. Events are placed at the intersection of a lane and optionally a group.
Use `get_workflow` to see existing events with their linked cards and schemas.

### create_domain_event
Add a new event to the workflow. This is the most important creation tool — events are what
make up the process flow.

- `workflowId`, `projectId` — Identifies the workflow
- `description` — The event name (use past-tense: "Order Created", not "Create Order"). **Avoid special characters** (`?`, `!`, `&`, `#`) — they break `$ref` path resolution. Hyphens are removed and the next letter capitalized in `$ref` keys (e.g., "Check-in" → `CheckIn`).
- `type` — Either `bpmn:Task` (regular event) or `bpmn:ExclusiveGateway` (decision diamond)
- `lane` — Name of the lane/actor this event belongs to (required)
- `follows` — A `$ref` path to the preceding event (e.g., `#/domainEvents/OrderPlaced`), or `"start"` for flow entry points. The new event is inserted after the parent in the flow sequence.
- `group` — Optional. Sets a group boundary starting at this event. Only set on the **first** event of a new group. Do not set group on subsequent events in the same group.
- `aggregateRoot` — `$ref` path to an entity (e.g., `#/schemas/entities/Order`). Links the entity as aggregate root on this event. **Every event should have one** — if the entity doesn't exist yet, set it later via `update_domain_event` after creating entities.
- `acceptanceCriteria` — Optional. Array of Given-When-Then acceptance criteria strings.
- `color` — Optional. Visual color: peach, yellow, green, teal, blue, lavender, pink, gray

**Note:** Gateway events (`bpmn:ExclusiveGateway`) may not appear in the `get_workflow` domainEvents section. Call `get_workflow` to discover their actual `$ref` paths if needed.

### update_domain_event
Modify an existing event — change its name, lane, color, aggregate root, or acceptance criteria.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`)
- `description` — New event name (optional)
- `lane` — Lane name to move event to (optional)
- `color` — New color (optional)
- `conditionLabel` — Branch label for events following a gateway (optional)
- `aggregateRoot` — `$ref` path to entity, or empty string to remove (optional)
- `acceptanceCriteria` — Array of GWT strings, replaces all existing (optional)

### delete_domain_event
Remove an event from the workflow. Connected child events are automatically relinked to the
deleted event's parent, preserving the flow.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event

---

## Entity Tools

Entities represent persistent domain objects — the data structures at the heart of the domain
model. Use `get_workflow` to see existing entities with their fields.

### create_entity
Define a new domain entity with typed fields. Each field needs a name, data type, and ideally
example data to make the model concrete.

- `workflowId` — Identifies the workflow
- `name` — Entity name (e.g., "Order", "Customer")
- `boundedContext` — Optional. Name of the bounded context to assign this entity to
- `fields` — Array of field definitions:
  - `name` — Field name in camelCase
  - `dataType` — One of: `string`, `number`, `boolean`, `object`
  - `exampleData` — Array of 3 realistic example values
  - `isRequired` — Whether the field is mandatory (true/false)
  - `relatedEntity` — `$ref` path to another entity (e.g., `#/schemas/entities/OrderItem`)
  - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`

### update_entity
Modify an entity — rename it, add/update/remove fields, or change its bounded context assignment.
Field operations are applied in order: remove -> update -> add.

- `workflowId` — Identifies the workflow
- `entity` — `$ref` path to the entity (e.g., `#/schemas/entities/Order`)
- `name` — New name (optional)
- `boundedContext` — Bounded context name to assign to (optional, empty string to clear)
- `addFields`, `updateFields`, `removeFields` — Field modification arrays

### delete_entity
Remove an entity from the domain model.

- `workflowId` — Identifies the workflow
- `entity` — `$ref` path to the entity

---

## Command Tools

Commands represent state-changing operations — actions that modify data. They correspond to
POST/PUT/DELETE API endpoints or write operations. Commands are created directly on domain events
and automatically create a Command card on that event. Use `get_workflow` to see existing commands.

### create_command
Define a new command with input fields, attached to a specific domain event. Auto-creates
a Command card on the event. Commands represent **API request payloads** — what a caller sends
to perform an action.

- `workflowId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`). Required.
- `name` — Command name with verb prefix and spaces (e.g., "Create Order", "Cancel Subscription")
- `fields` — Array of field definitions:
  - `name` — Field name in camelCase
  - `isRequired` — Whether the field is required/mandatory
  - `hideInForm` — Set true for auto-generated fields like IDs and timestamps
  - `relatedEntity` — `$ref` path to a related entity. **Only use for embedded collections** where multiple fields from the related entity are needed (e.g., `orderItems`). Do NOT use for simple ID references — use a plain string field instead (e.g., `customerId`, `bookingId`).
  - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`
  - `fields` — Nested field names from the related entity (for reference fields)

**Field design rules for commands:**

- Use flat string fields for ID references: `hotelId`, `bookingId`, `customerId`
- Never combine "Id" suffix with `relatedEntity` — that creates `{ hotelId: { id: "123" } }` instead of `{ hotelId: "123" }`
- Search/filter parameters belong on read models, not commands
- Command fields should match fields on the aggregate root entity

### update_command
Modify a command — rename or change fields. Field operations are applied in order: remove -> update -> add.

- `workflowId` — Identifies the workflow
- `command` — `$ref` path to the command (e.g., `#/schemas/commands/PlaceOrder`)
- `name` — New name (optional)
- `addFields`, `updateFields`, `removeFields` — Field modifications

### delete_command
Remove a command and its card from the event.

- `workflowId` — Identifies the workflow
- `command` — `$ref` path to the command

---

## Domain Event Schema Tools

Domain event schemas define the data payload carried when a domain event occurs — the facts
about what happened. They capture the essential state change information (not the full entity
state). Domain event schemas are created directly on domain events and automatically create
a Domain Event card on that event. Use `get_workflow` to see existing domain event schemas
under `schemas/domainEvents/`.

### create_domain_event_schema
Define a new domain event schema with payload fields, attached to a specific domain event.
Auto-creates a Domain Event card on the event. The schema represents the data that is published
when the event fires.

- `workflowId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`). Required.
- `name` — Schema name with spaces (e.g., "Order Placed", "Payment Processed")
- `entity` — `$ref` path to the aggregate root entity that produces this event (e.g., `#/schemas/entities/Order`). Recommended.
- `fields` — Array of field definitions representing the event payload:
  - `name` — Field name in camelCase
  - `relatedEntity` — `$ref` path to a related entity. Use for nested event data (e.g., order items carried in the event).
  - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`
  - `fields` — Nested field names from the related entity (for reference fields)

**Field design rules for domain event schemas:**

- Include identifier fields (e.g., `orderId`, `customerId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- Capture what happened, not the full entity state — usually 3 to 8 fields
- Field names should be consistent with command and entity field names where applicable

### update_domain_event_schema
Modify a domain event schema — rename, change entity, or modify fields. Field operations are
applied in order: remove -> update -> add.

- `workflowId` — Identifies the workflow
- `domainEventSchema` — `$ref` path to the schema (e.g., `#/schemas/domainEvents/OrderPlaced`)
- `name` — New name (optional)
- `entity` — Updated aggregate root entity `$ref` path (optional, empty string to remove)
- `addFields`, `updateFields`, `removeFields` — Field modifications

### delete_domain_event_schema
Remove a domain event schema and its card from the event.

- `workflowId` — Identifies the workflow
- `domainEventSchema` — `$ref` path to the schema

---

## Read Model Tools

Read models represent data queries and views — how data is retrieved and displayed. They
correspond to GET API endpoints or query operations. Read models are created directly on domain
events and automatically create a Read Model card on that event. Use `get_workflow` to see
existing read models.

### create_read_model

Define a new read model/query, attached to a specific domain event. Auto-creates a Read Model
card on the event. Read models represent **API response payloads** — what the API returns to callers.

- `workflowId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`). Required.
- `name` — Read model name with spaces (e.g., "Get Order Details", "List Active Products", "Search Customers")
- `cardinality` — Required. `"one-to-one"` for single-record queries, `"one-to-many"` for list queries.
- `entity` — `$ref` path to the source entity (e.g., `#/schemas/entities/Order`). Recommended.
- `fields` — Array of field definitions representing the full query contract (both inputs and outputs):
  - `name` — Field name in camelCase
  - `isFilter` — Set true for query parameters/filters, omit for returned data fields. Filter fields can be cross-entity parameters (e.g., `checkInDate`, `priceMin`) — this is valid.
  - `relatedEntity` — `$ref` path to a related entity. Use for **composed response data** — nested objects in API responses (e.g., `guest` with `firstName`, `lastName`, `email` from Guest entity). Name these fields as the entity name (`guest`, `hotel`), not with "Id" suffix.
  - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`
  - `fields` — Nested field names from the related entity (for reference fields)

### update_read_model
Modify a read model — rename, change source entity, or modify fields. Field operations are
applied in order: remove -> update -> add.

- `workflowId` — Identifies the workflow
- `readModel` — `$ref` path to the read model (e.g., `#/schemas/queries/GetOrderDetails`)
- `name` — New name (optional)
- `entity` — Updated source entity `$ref` path (optional)
- `addFields`, `updateFields`, `removeFields` — Field modifications

### delete_read_model
Remove a read model and its card from the event.

- `workflowId` — Identifies the workflow
- `readModel` — `$ref` path to the read model

---

## Card Tools

Cards attach domain model elements and requirements to events. Most domain model cards are
created automatically by dedicated tools (`create_command`, `create_read_model`,
`create_domain_event` with `aggregateRoot`). Use `create_card` only for other card types
like User Story.

### list_card_types
Get the available card type definitions for a workflow. Each card type has an ID, name, and
role. Call this before creating cards to get the correct `cardTypeId` values.

Common card types:
- **Command** — Created automatically via `create_command`
- **AggregateRoot** — Created automatically via `create_domain_event` with `aggregateRoot` parameter
- **ReadModel** — Created automatically via `create_read_model`
- **DomainEvent** — Created automatically via `create_domain_event_schema`
- **GivenWhenThen** — Created automatically via `create_domain_event` with `acceptanceCriteria` parameter
- **UserStory** — Use `create_card` for this type

Use `get_workflow` to see existing cards on events.

### create_card
Attach a card to an event. Only needed for card types not managed by dedicated tools (e.g., UserStory).

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`)
- `cardTypeId` — The card type (from `list_card_types`)
- `description` — Card content/description

### update_card
Modify a card's content.

- `workflowId`, `projectId` — Identifies the workflow
- `cardId` — Identifies the card
- `description` — Updated content

### delete_card
Remove a card from an event.

- `workflowId`, `projectId` — Identifies the workflow
- `cardId` — Identifies the card

---

## Bounded Context Tools

Bounded contexts are logical boundaries that group related entities together. They represent
separate areas of the domain that could potentially be separate microservices. Creating a
bounded context auto-assigns any unassigned entities to it.

### create_bounded_context
Create a new bounded context. Any existing entities without a bounded context are automatically
assigned to the new one. **Create bounded contexts before entities** so you can assign entities
during creation.

- `workflowId`, `projectId` — Identifies the workflow
- `name` — Context name (e.g., "Order Management", "Hotel Booking")

**Design guidance:** A bounded context maps to a deployment/service boundary. For small/medium
workflows (< 10 entities), one BC is usually sufficient. Only split into multiple BCs when domains
are genuinely independent with clear interaction boundaries.

### update_bounded_context
Rename a bounded context.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Current name of the context
- `name` — New name

### delete_bounded_context
Remove a bounded context. Entities in it become unassigned.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Name of the context to delete

---

## Validation Tools

### validate_domain_model
Check the workflow's domain model for structural issues. Returns a list of problems such as command/read model fields
that don't match their aggregate root entity, missing relationships, and naming mismatches.

- `workflowId`, `projectId` — Identifies the workflow

Run this after creating all entities, commands, read models, and domain event schemas to catch field mismatches. **MAJOR** issues should be
fixed. **MINOR** `FIELD_NOT_IN_ENTITY` issues on read model filter fields are often expected — cross-entity query
parameters (e.g., `checkInDate` on a Hotel search) don't need to exist on the entity.
