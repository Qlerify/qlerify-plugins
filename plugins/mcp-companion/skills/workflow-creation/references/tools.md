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

### create_project

Create a top-level project — the container that owns workflows. Only `name` is required.
Returns the new project ID and a URL — show the **full URL as plain text on its own line**
(not wrapped in Markdown like `[text](url)`) so the terminal auto-links it, and it
can be clicked/copied. Use this when the user has no project yet (a fresh account,
or `list_workflows` returns none), then pass the new project ID to `create_workflow`.

- `name` — The project name (e.g., "Acme E-Commerce", "Internal Tools")

### create_workflow

Create a brand-new empty workflow inside a project. Provide a descriptive name — this appears
in the UI and helps users find the workflow later. Returns the new workflow ID.

After creation, give the user the workflow's **full URL as plain text on its own line** so the terminal
auto-links it (clickable + copyable). Do **not** hide it behind link text like `[Open workflow](url)` —
that renders as non-actionable blue text in a console. Use this pattern:
`https://app.qlerify.com/workflow/{projectId}/{workflowId}`

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

Each domain event and decision also carries a **`position: [col, row]`** — its cell in the diagram's
global grid (`col` = column left→right, `row` = row top→bottom; the top-left node is `[0, 0]`, the one
directly below it `[0, 1]`). `lane` and `group` say which band/column an element belongs to
semantically; `position` is its absolute placement. Read it to see how the diagram is actually laid out —
relative order, what sits in which column, whether two nodes share a cell — instead of inferring
layout from `follows` alone.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Optional. Bounded context name to export. If omitted, uses the first bounded context.

### generate_openapi_spec

Generate an OpenAPI/Swagger YAML specification from the workflow's domain model. Requires at least
one bounded context with entities. Useful for bootstrapping API implementations from the domain model.

- `workflowId`, `projectId` — Identifies the workflow
- `boundedContext` — Name of the bounded context to generate the spec for

---

## Lane Tools

Swimlanes (or "lanes") are the horizontal rows on the Event Storming board, each representing a **role** — an actor that invokes commands. Use short business names (Customer, Hotel Staff, Admin, Warehouse Worker) or the exact word **Automation** for any system-triggered action (sending emails, processing payments, generating invoices). Aim for 2–4 lanes — more makes the diagram hard to read.

**Naming rules:**

- Max 18 characters; alphanumerics and spaces only — no `?`, `!`, `&`, `#`, `/` (these characters are not supported in lane names and can cause validation/escaping issues in the UI and API).
- Matching is **exact and case-sensitive** — `Customer` and `customer` produce two separate lanes.
- Use exactly `Automation` for system-triggered steps. Do NOT create lanes for internal services (Payment Service, Notification Service, etc.) — those are bounded contexts.

When `create_domain_event` references an unfamiliar lane name, the lane is auto-created and appended to the bottom of the diagram. Use `get_workflow` to see existing lanes.

### create_lane

Seed an empty lane before any events reference it. Usually unnecessary since `create_domain_event` auto-creates lanes. Duplicate names are not blocked but create a separate lane each time — check `get_workflow` first. By default the lane is appended at the bottom; pass `position` to insert it at a specific vertical slot.

- `workflowId`, `projectId` — Identifies the workflow
- `name` — The lane name. See "Naming rules" above.
- `position` — Optional. 0-based vertical slot for the new lane (`0` = topmost; "first"/"top" → `0`, "second" → `1`). Omit to append at the bottom.

### update_lane

Rename an existing lane. Does not affect events already in the lane.

- `workflowId`, `projectId` — Identifies the workflow
- `lane` — Current lane name
- `name` — The new name

### delete_lane

Remove a lane. The lane must be empty — move or delete its events first.

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

Domain Events are the core building blocks in Event Storming and Event Modeling — they represent things that happen
in the business process. Use `get_workflow` to see existing events with their linked commands, read models, aggregate roots, and schemas.

### create_domain_events

Bulk-create multiple domain events in a single call. Use this when building a new workflow with
the full event flow — typically the entire chain in one call. For inserting one or two events
into an existing workflow, use the singular `create_domain_event`.

Lanes are auto-created when first referenced (same as `create_domain_event`), so you don't need to
create lanes ahead of time. Bounded contexts and aggregate-root entities, if referenced via
`aggregateRoot`, must already exist.

If validation fails on the Nth entry, entries 1..N-1 are already committed. The tool stops at the
first failure and returns which entries succeeded and which failed, so you can fix the failed
entry and call again with only the remaining ones.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvents` — Array of event definitions, each with:
  - `description` — The event name (past-tense). Same rules as `create_domain_event`'s `description` below.
  - `type` — `'domainEvent'` or `'decision'`
  - `lane` — Role/actor name. Auto-created if unfamiliar; pass the same name across events that share a lane.
  - `follows` — One of: `"start"`, a `$ref` path to an existing event, OR (bulk-only) the bare description of an event created earlier in the same array (e.g. `"follows": "Order Placed"`).
  - `parallel` — Optional. Same as `create_domain_event` below. To model concurrent branches off the same non-`decision` parent, give each branch the same `follows` and set `parallel: true` on every branch after the first; otherwise a second entry pointing at the same parent is inserted between the parent and its existing follower(s) — all of the parent's current followers are reparented under the new event. No effect when the parent is a `decision` (decisions branch by default) or has no follower yet.
  - `group`, `color`, `aggregateRoot`, `acceptanceCriteria`, `conditionLabel` — Optional, same as `create_domain_event` below. Set `conditionLabel` on a branch whose `follows` is a decision to label it inline (no separate `update_domain_event` pass needed).

### create_domain_event

Add a single domain event to the workflow. Use this for inserting one or two events into an
existing workflow. For building a new workflow with many events, use `create_domain_events` instead.

- `workflowId`, `projectId` — Identifies the workflow
- `description` — The event name (use past-tense: "Order Created", not "Create Order"). **Avoid special characters** (`?`, `!`, `&`, `#`, `/`) — they break `$ref` path resolution. Hyphens are removed and the next letter capitalized in `$ref` keys (e.g., "Check-in" → `CheckIn`).
- `type` — Either `'domainEvent'` (regular event) or `'decision'`
- `lane` — Name of the role/actor this event belongs to (required). Auto-created on the fly if no lane with that name exists; pass the exact same name across events that share a lane (matching is case-sensitive). See "Lane Tools" above for naming rules.
- `follows` — A `$ref` path to the preceding event (e.g., `#/domainEvents/OrderPlaced`), or `"start"` for flow entry points. The new event is connected after the parent. If the parent is a domain event that already has one or more followers, the new event is **inserted between** the parent and those followers by default (all of them are reparented under the new event); set `parallel: true` to add it as a concurrent branch beside them instead. If the parent is a `decision`, the new event is always added as a new branch (no `parallel` needed).
- `parallel` — Optional. When `true`, and the parent already has one or more followers, the new event is added as a concurrent (AND) branch alongside them rather than inserted between. Use for parallel flows where multiple events genuinely happen after the same event with no condition. For conditional/exclusive (either-or) branching, use a `decision` instead. No effect when the parent is a decision or has no followers yet.
- `group` — Optional. Sets a group boundary starting at this event. Only set on the **first** event of a new group. Do not set group on subsequent events in the same group.
- `aggregateRoot` — Optional. `$ref` path to an entity (e.g., `#/schemas/entities/Order`). Links an entity as aggregate root for the command triggering this event. **Every command / event should have one** — if the entity doesn't exist yet, set it later via `update_domain_event` after creating entities.
- `acceptanceCriteria` — Optional. Array of Given-When-Then acceptance criteria strings.
- `conditionLabel` — Optional. Branch label shown on the arrow when this event `follows` a **decision** (e.g. "Yes", "No", "Approve"). Labels a guard-fork branch at creation; ignored if the parent isn't a decision. (Can also be set later via `update_domain_event`.)
- `color` — Optional. Visual color: peach, yellow, green, teal, blue, lavender, pink, gray

**Note:** Decisions may not appear in the `get_workflow` domainEvents section. Call `get_workflow` to discover their actual `$ref` paths if needed.

### update_domain_event

Modify an existing domain event — change its name, lane, color, condition label, group, aggregate root, or acceptance criteria.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`)
- `description` — New event name (optional)
- `lane` — Lane name to move event to (optional)
- `color` — New color (optional)
- `conditionLabel` — Branch label for events following a gateway, or empty string to clear (optional)
- `group` — Group name to assign this event to, or empty string to remove from its current group (optional). Only set on the first event of a new group — subsequent events inherit via the parent chain.
- `aggregateRoot` — `$ref` path to entity, or empty string to remove (optional)
- `acceptanceCriteria` — Array of GWT strings, replaces all existing (optional)

### delete_domain_event

Remove an event from the workflow. Connected child events are automatically relinked to the
deleted event's parent, preserving the flow.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event

### add_connection

Add an arrow from one existing event/decision to another **without moving either**. The target
gains the source as an *additional* incoming parent; its existing main parent — and therefore its
position and group — stays put. Use it for fan-in joins (a state or decision reached from several
predecessors) and for secondary parents that `create_domain_event` can't express (it only sets a
single `follows`). Build the primary spine with `create_domain_event(s)`; use this only for the
extra incoming edges.

- `workflowId`, `projectId` — Identifies the workflow
- `from` — `$ref` path to the SOURCE the arrow starts from (event or decision)
- `to` — `$ref` path to the TARGET the arrow points to; it gains `from` as an extra parent
- `conditionLabel` — Optional branch label shown when `from` is a decision (e.g. "Yes" / "No")

Errors if the connection already exists or `from` and `to` are the same. Does not create events — both must already exist.

### remove_connection

Remove an arrow between two existing events/decisions — the inverse of `add_connection`. The source is dropped from the target's incoming parents (and that branch's condition label is cleared), applying the same diagram rules as the UI: if the target is left with no parent it becomes a starting point; if its main parent was removed the next parent takes over (the target may shift); groups/positions are reconciled.

- `workflowId`, `projectId` — Identifies the workflow
- `from` — `$ref` path to the SOURCE the arrow starts from
- `to` — `$ref` path to the TARGET the arrow points to; it loses `from` as a parent

Errors if no such connection exists. To delete an event entirely (not just one incoming edge), use `delete_domain_event`.

---

## Entity Tools

Entities are persistent domain objects. Value Objects are attribute-based types typically used within those entities as part of the domain model. Use `get_workflow` to see existing entities and value objects with their fields.

### create_entities

Define one or more domain entities or value objects in a single atomic workflow write. Pass an array; for a single entity, pass an array with one element. One bulk call avoids the version-conflict races that arise when creating entities in parallel. Note: Value Objects are created with the same tool as entities but always without an id attribute. In the domain model, VOs have a separate path from entities based on the existence of an id field.

- `workflowId` — Identifies the workflow
- `entities` — Array of entity / VO definitions, each with:
  - `name` — Entity or VO name (e.g., "Order", "Customer")
  - `description` — One or two concise sentences describing the entity / VO's role in the domain. For aggregate roots, explain what the aggregate represents and its lifecycle responsibility; for value objects, note that it is replaced as a whole.
  - `boundedContext` — Optional. Name of the bounded context to assign this entity / VO to
  - `aggregateRootFor` — Optional. Array of `$ref` paths to events this entity is the aggregate root for (e.g., `["#/domainEvents/OrderPlaced"]`). Only entities with an `id` field can be aggregate roots.
  - `fields` — Optional. Array of field definitions. Use this when you want to create the entity / VO with its fields in the same call; for minimal initial creation, you can omit `fields` here and add or enrich field metadata later via `update_entities`. If you intend to create an entity, include a field named exactly `id` in the initial `create_entities` call. Otherwise the item will be classified as a value object.
    - `name` — Field name in camelCase (entities must always have one attribute named exactly `id` at creation time; value objects must never have a field named `id`)
    - `description` — Required for every field definition you include in this request. Use a short business-language description of the field; for self-explanatory fields like `id`, `name`, or `email`, keep it brief and precise.
    - `dataType` — One of: `string`, `number`, `boolean`, `object`
    - `exampleData` — Array of 3 realistic example values
    - `isRequired` — Whether the field is mandatory (true/false)
    - `isKey` — Whether the field is part of the entity's primary key (true/false). The `id` field is usually the key; mark multiple fields to form a composite key. Omit for value objects.
    - `relatedEntity` — `$ref` path to another entity (e.g., `#/schemas/entities/OrderItem`)
    - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`

The whole batch is atomic: if any entity fails validation (e.g. duplicate name), none are created.

### update_entities

Modify one or more entities in a single atomic workflow write — rename, add/update/remove fields, or change bounded context. Pass an array of entity updates; for a single entity, pass an array with one element. Field operations are applied per entity in order: remove → update → add.

- `workflowId` — Identifies the workflow
- `entities` — Array of entity updates, each with:
  - `entity` — `$ref` path to the entity (e.g., `#/schemas/entities/Order`). Required.
  - `name` — New name (optional)
  - `description` — Updated top-level description for the entity / VO (optional)
  - `boundedContext` — Bounded context name to assign to (optional, empty string to clear)
  - `addFields`, `updateFields`, `removeFields` — Field modification arrays. Field bodies in `addFields` and `updateFields[].updates` accept the same properties as `create_entities` fields, including `description`.

The whole batch is atomic: if any update fails validation, none are applied.

### delete_entity

Remove an entity from the domain model.

- `workflowId` — Identifies the workflow
- `entity` — `$ref` path to the entity

---

## Command Tools

Commands represent state-changing operations — actions that modify data. They correspond to POST/PUT/DELETE API endpoints or write operations. Commands and domain events have a one-to-one relationship. Commands are created on and attached to domain events. Use `get_workflow` to see existing commands.

### create_commands

Define one or more commands in a single atomic workflow write. Pass an array of commands, each bound to a domain event; for a single command, pass an array with one element. Commands with their attributes represent the information an actor submits to perform a state-changing action on an aggregate. Each event can have only one command, so each command in the batch must target a different event.

- `workflowId` — Identifies the workflow
- `commands` — Array of command definitions, each with:
  - `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`). Required.
  - `name` — Command name with verb prefix and spaces (e.g., "Create Order", "Cancel Subscription")
  - `fields` — Array of field definitions or command attributes:
    - `name` — Field name in camelCase
    - `isRequired` — Whether the field is required/mandatory
    - `hideInForm` — Set true for auto-generated fields like IDs and timestamps
    - `relatedEntity` — `$ref` path to a related entity. Use this for attributes implicitly typed as an object. Do NOT use this for primitive types. Do NOT use this for strings holding id references to entities or value objects in other bounded contexts.
    - `cardinality` — `"one-to-one"` or `"one-to-many"` for fields with `relatedEntity`
    - `fields` — Nested field names from the related entity (only for fields with relatedEntity)

**Field design rules for commands:**

- When referencing an entity in an external bounded context (such as a user in the user authentication service), never use nested fields or `relatedEntity`; instead, use a simple string field whose name ends with "Id" (e.g., `userId`).
- Never combine "Id" suffix with `relatedEntity` — that creates `{ hotelId: { id: "123" } }` instead of `{ hotelId: "123" }`
- Search/filter parameters belong on read models, not commands
- Command attributes should match attributes on entities or value objects

### update_commands

Modify one or more commands in a single atomic workflow write — rename or change fields. Pass an array of command updates; for a single command, pass an array with one element. Field operations are applied per command in order: remove → update → add.

- `workflowId` — Identifies the workflow
- `commands` — Array of command updates, each with:
  - `command` — `$ref` path to the command (e.g., `#/schemas/commands/PlaceOrder`). Required.
  - `name` — New name (optional)
  - `addFields`, `updateFields`, `removeFields` — Field modifications

The whole batch is atomic: if any update fails validation, none are applied.

### delete_command

Remove a command from the event.

- `workflowId` — Identifies the workflow
- `command` — `$ref` path to the command

---

## Domain Event Schema Tools

Domain event schemas define the data payload carried when a domain event occurs — the facts
about what happened. They capture the essential state change information (not the full entity
state). Domain event schemas are attached directly to domain events. Use `get_workflow` to see
existing domain event schemas under `schemas/domainEvents/`.

### create_domain_event_schemas

Define one or more domain event schemas in a single atomic workflow write. Pass an array of schemas, each bound to a domain event; for a single schema, pass an array with one element. Schemas represent the data payload published when the event fires. Each event can have only one domain event schema, so each schema in the batch must target a different event.

- `workflowId` — Identifies the workflow
- `domainEventSchemas` — Array of schema definitions, each with:
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

### update_domain_event_schemas

Modify one or more domain event schemas in a single atomic workflow write — rename, change entity, or modify fields. Pass an array of schema updates; for a single schema, pass an array with one element. Field operations are applied per schema in order: remove → update → add.

- `workflowId` — Identifies the workflow
- `domainEventSchemas` — Array of schema updates, each with:
  - `domainEventSchema` — `$ref` path to the schema (e.g., `#/schemas/domainEvents/OrderPlaced`). Required.
  - `name` — New name (optional)
  - `entity` — Updated aggregate root entity `$ref` path (optional, empty string to remove)
  - `addFields`, `updateFields`, `removeFields` — Field modifications

The whole batch is atomic: if any update fails validation, none are applied.

### delete_domain_event_schema

Remove a domain event schema from the event.

- `workflowId` — Identifies the workflow
- `domainEventSchema` — `$ref` path to the schema

---

## Read Model Tools

Read models represent data queries and views — what the API returns to callers (GET endpoints / query operations). Use `get_workflow` to see existing read models. See `references/read-model-generation.md` for field design rules.

### create_read_models

Define one or more read models/queries in a single atomic workflow write. Pass an array of read models, each bound to a domain event; for a single read model, pass an array with one element. An event can carry several different read models (e.g. "Get Order Details" + "List Orders" both on `OrderPlaced`); only attaching the same read model name to the same event twice is rejected.

**Reuse:** if two entries in the batch (or an entry and an existing workflow query) share the same `name`, both events are linked to the same underlying read model and their fields are unioned. Use this to share a query across multiple events (e.g., "Get Order Details" on several events along an Order's lifecycle).

- `workflowId` — Identifies the workflow
- `readModels` — Array of read model definitions, each with:
  - `domainEvent` — Required. `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`)
  - `name` — Read model name with spaces (e.g., "Get Order Details")
  - `description` — Optional one-sentence description of what the query returns and when it is used. Omit when the name already makes the intent obvious.
  - `cardinality` — Required. `"one-to-one"` or `"one-to-many"`
  - `entity` — Recommended. `$ref` path to the source entity (e.g., `#/schemas/entities/Order`)
  - `fields` — Array of field definitions. Each field:
    - `name` — camelCase
    - `description` — Short description of the field's meaning, purpose, or filter/computed intent. Add when non-obvious from the name; omit on self-explanatory fields.
    - `isFilter` — Boolean; mark query/filter parameters
    - `computed` — Boolean; mark fields calculated at runtime
    - `relatedEntity` — `$ref` path; use for nested composed data
    - `cardinality` — `"one-to-one"` or `"one-to-many"` for `relatedEntity` fields
    - `fields` — Nested sub-fields when `relatedEntity` is set

### update_read_models

Modify one or more read models in a single atomic workflow write — rename, change source entity, or modify fields. Pass an array of read model updates; for a single read model, pass an array with one element. Field operations are applied per read model in order: remove → update → add.

- `workflowId` — Identifies the workflow
- `readModels` — Array of read model updates, each with:
  - `readModel` — `$ref` path to the read model (e.g., `#/schemas/queries/GetOrderDetails`). Required.
  - `name` — New name (optional)
  - `entity` — Updated source entity `$ref` path (optional)
  - `addFields`, `updateFields`, `removeFields` — Field modifications

The whole batch is atomic: if any update fails validation, none are applied.

### delete_read_model

Remove a read model from the event.

- `workflowId` — Identifies the workflow
- `readModel` — `$ref` path to the read model

---

## Card Tools

Cards attach domain model elements and requirements to events. Most domain model cards are
created automatically by dedicated tools (`create_commands`, `create_read_models`,
`create_domain_event` with `aggregateRoot`). Use `create_card` only for other card types
like User Story.

### list_card_types

Get the available card type definitions for a workflow. Each card type has an ID, name, and
role. Call this before creating cards to get the correct `cardTypeId` values.

Common card types:

- **Command** — Created automatically via `create_commands`
- **AggregateRoot** — Created automatically via `create_domain_event` with `aggregateRoot` parameter
- **ReadModel** — Created automatically via `create_read_models`
- **DomainEvent** — Created automatically via `create_domain_event_schemas`
- **GivenWhenThen** — Created automatically via `create_domain_event` with `acceptanceCriteria` parameter
- **UserStory** — Use `create_card` for this type

Use `get_workflow` to see existing cards on events.

### create_card

Attach a card to an event. Only needed for card types not managed by dedicated tools (e.g., UserStory).

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event (e.g., `#/domainEvents/OrderPlaced`)
- `cardTypeId` — The card type (from `list_card_types`)
- `description` — Card content/description (supports HTML)

### update_card

Modify a card — change its content or card type. Cards are identified by their event and current text.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event the card is on
- `currentDescription` — The card's existing description, used to locate it
- `description` — New content (optional, supports HTML)
- `cardTypeId` — Change the card type (optional)

### delete_card

Remove a card from an event. Cards are identified by their event and text.

- `workflowId`, `projectId` — Identifies the workflow
- `domainEvent` — `$ref` path to the event the card is on
- `description` — The card's description, used to locate it

---

## Bounded Context Tools

Bounded contexts are logical boundaries within a domain model. They represent
separate areas of the domain that could potentially be separate microservices. Creating a
bounded context auto-assigns any unassigned entities to it.

### create_bounded_context

Create a new bounded context. Any existing entities without a bounded context are automatically
assigned to the new one. **Create bounded contexts before entities** so you can assign entities
during creation.

- `workflowId`, `projectId` — Identifies the workflow
- `name` — Context name (e.g., "Order Management", "Hotel Booking")

**Design guidance:** A bounded context maps to a deployment/service boundary. For small or medium-sized
domains, one BC is usually sufficient. Only split into multiple BCs when domains
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
- `boundedContext` — Optional. Restrict validation to a single bounded context. If omitted, validates all bounded contexts plus global/unassigned elements.
- `scope` — Optional. Filter issues by type: `"all"` (default), `"entities"`, `"commands"`, or `"readModels"`.

Run this after creating all entities, commands, read models, and domain event schemas to catch field mismatches. **MAJOR** issues should be
fixed. **MINOR** `FIELD_NOT_IN_ENTITY` issues on read model filter fields are often expected — cross-entity query
parameters (e.g., `checkInDate` on a Hotel search) don't need to exist on the entity.
