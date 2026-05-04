---
name: workflow-creation
description: >
   This skill should be used when the user asks to "create a workflow",
   "build a domain model", "set up a new Qlerify workflow", "add events and lanes",
   "create BPMN diagram", "model a business process", "set up domain events",
   "add commands and read models to workflow", or any request involving building
   a Qlerify workflow from scratch or adding structural elements to an existing one.
   If the user wants to model from an existing or legacy codebase, first extract
   one aggregate at a time (use the extract-aggregate skill, typically one
   workflow per aggregate), then continue with this skill.
   Provides the full creation sequence, tool ordering, and domain modeling guidance.
allowed-tools: Read, Glob, Grep
---

# Workflow Creation Guide

Build well-structured Qlerify workflows by following a specific tool sequence. Skipping steps
or calling tools out of order leads to broken references and incomplete diagrams.

## Reverse-engineering from existing code

When the source of truth is an existing or legacy codebase, do not jump directly into workflow creation.
First extract one aggregate at a time (use the extract-aggregate skill):

1. Isolate one aggregate at a time
2. Use one Qlerify workflow per aggregate
3. Review the extracted aggregate plan with the user
4. Then return to this skill to model the workflow

## Core Concepts

A Qlerify workflow is a BPMN-style diagram combined with domain-driven design (DDD) elements:

- **Lanes** — Horizontal swim lanes representing real-world actor roles or automation (e.g., "Customer", "Hotel Staff", "Automation")
- **Domain Events** — The core building blocks placed in lanes. Each event represents something that happens in the business process (e.g., "Order Created", "Payment Received")
- **Entities** — Persistent domain objects (Aggregate Roots) with typed fields and relationships
- **Commands** — State-changing operations attached to events, representing API request payloads
- **Read Models** — Data queries/views attached to events, representing API response payloads
- **Domain Event Schemas** — Data payloads carried when events occur, capturing the essential facts about what happened
- **Bounded Contexts** — Logical boundaries grouping related entities, mapping to deployment/service boundaries
- **Groups** — Optional vertical columns representing phases or stages on the diagram (e.g., "Order Placement", "Fulfillment"). **Never create them unless the user explicitly asks** — see Phase 5.

### How elements reference each other

Tools use `$ref` paths to link elements together. The `get_workflow` tool returns the full workflow specification
where each element has a `$ref` path:

- Domain events: `#/domainEvents/OrderPlaced`
- Entities: `#/schemas/entities/Order`
- Commands: `#/schemas/commands/CreateOrder`
- Read models: `#/schemas/queries/GetOrderDetails`
- Domain event schemas: `#/schemas/domainEvents/OrderPlaced`

Use these paths when creating commands, read models, domain event schemas, or referencing entities in fields.

**Tip:** The workflow specification returned by `get_workflow` includes a `$schema` URL (e.g., `https://app.qlerify.com/schemas/domain-model/v1`). Fetch this JSON Schema to understand the full structure of the spec — field types, allowed values, and relationships between elements.

### Event naming and `$ref` path generation

Event names are converted to PascalCase `$ref` keys:

- Spaces are removed, each word capitalized: `"Order Placed"` → `OrderPlaced`
- Hyphens are removed, next letter capitalized: `"Check-in Completed"` → `CheckInCompleted`
- **Special characters (`?`, `!`, `&`, `#`, `/`) break `$ref` path resolution** — avoid them entirely

Use only alphanumeric characters and spaces in event names. If you use hyphens, call `get_workflow`
afterward to verify the actual `$ref` key before referencing it in subsequent calls.

**Note:** Gateway events (`decision`) may not appear in the `get_workflow` domainEvents section.
If you need to reference a gateway, call `get_workflow` to discover its actual `$ref` path, or use a clean
name without special characters and infer the PascalCase key.

## Creation Sequence

Follow these steps in order. Each step depends on the previous one.

### Phase 1: Foundation

**Step 1 — Identify or create the workflow**

For an existing workflow, call `list_workflows` to find it, then `get_workflow` to understand
its current state. For a new workflow, call `create_workflow` with a descriptive name.

**Step 2 — Create lanes**

Every event must belong to a lane. Lanes represent **real-world actor roles** (people) or
**Automation** (system-triggered actions). Do NOT create lanes for internal services or technical
components.

Call `create_lane` for each actor. Aim for 2-4 lanes.

**Lane design rules:**

- Lanes are **people/roles** (Customer, Admin, Hotel Staff, Warehouse Worker) or **Automation** (any action triggered by the system without human intervention)
- Do NOT create lanes for internal services (Payment Service, Notification Service, Order Service, etc.)
- System-triggered actions (sending emails, processing payments, generating invoices, checking availability) belong in the **Automation** lane
- Ask: "Is this a person who performs an action, or a system that runs automatically?" If the latter → Automation

**Common patterns:**

- E-commerce: Customer, Warehouse Staff, Automation
- Hotel Booking: Guest, Hotel Staff, Automation
- HR Onboarding: Candidate, HR Manager, Automation
- Order Management: Customer, Admin, Automation

### Phase 2: Event Flow

**Step 3 — Create domain events** *(see `references/event-generation.md` for naming and chronology rules)*

Build the event flow by chaining calls to `create_domain_event`. Each event needs:

- `description` — Aggregate Name + Space + Past Tense Verb
- `lane` — The lane name (e.g., "Customer")
- `follows` — A `$ref` path to the preceding event, or `"start"` for flow entry points
- `type` — Either `domainEvent` (regular event) or `decision` (decision diamond)

Optional parameters:

- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings

Build the flow left-to-right, top-to-bottom, creating events in the order they occur in the
business process. Do NOT set `aggregateRoot` yet — entities don't exist at this point.
Do NOT set `group` either — groups are a cosmetic polish step handled later (Phase 5) and only if the user explicitly asks for them.

### Phase 3: Domain Model

**Step 4 — Create bounded contexts**

Create bounded contexts BEFORE entities, so entities can be assigned during creation. Call
`create_bounded_context` for each logical boundary.

**Bounded context design rules:**

- A bounded context often maps to a **microservice or to a module inside a modular monolith**
- Only split into multiple BCs when there are genuinely independent domains with clear interaction boundaries
- Ask: "Could a team realistically build and deploy this as a separate service?" If not → same BC
- Over-splitting creates unnecessary inter-service complexity (distributed transactions, API contracts, eventual consistency)
- Common pattern: start with 1 BC, split later when the domain grows and clear boundaries emerge

**Examples:**

- Hotel Booking (6 entities): 1 BC — "Hotel Booking"
- Large E-commerce Platform (20+ entities): 3-4 BCs — "Order Management", "Inventory", "Customer Management", "Payments"

**Step 5 — Generate entity and value object names without attributes**

Create entities and value objects with just their name and bounded context — **no fields yet**. This establishes
`$ref` paths so commands, read models, and domain event schemas can reference them.

```
create_entity(workflowId: "wf-1", name: "Order", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/Order" }

create_entity(workflowId: "wf-1", name: "Order Item", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/OrderItem" }

create_entity(workflowId: "wf-1", name: "Address", boundedContext: "Order Management")
-> { $ref: "#/schemas/valueObjects/Address" }
```

Identify ALL entities and value objects the domain needs — aggregate root entities, related or associated entities,
and value objects. Create them all now so their `$ref` paths are available for subsequent steps.

**Step 6 — Link aggregate roots to events**

Now that entities exist, link each event to its aggregate root entity:

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", aggregateRoot: "#/schemas/entities/Order")
```

**Every event must have an aggregate root before proceeding.** This identifies which entity each event primarily affects.

**Step 7 — Create commands on events** *(see `references/command-generation.md` for detailed field rules)*

Call `create_command` for each state-changing operation. Commands and domain events have a one-to-one relationship. Associate a command with an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).
A command is attached to a domain event using a card on that event.

- Name commands using action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Use `relatedEntity` when an attribute holds related entities or value objects

For each domain event, specify the name and arguments (also referred to as fields, attributes or properties) of the command that produces the event. Think of the command attributes as information fields that the actor (human or automation) fills in and submits. Each field name must be in camelCase. Commands are always invoked on aggregates through the aggregate root. The command attributes on the first level correspond to attributes on the aggregate root. Fields can be nested up to two levels down from the aggregate root and should be nested when updating underlying related entities or value objects. The command's argument structure must always mirror the structure of the entities and VOs inside the aggregate and their attribute names. Even if the user input explicitly prescribes a flat command structure, make sure to mirror the nested structure of the aggregate. Remember that VOs always carry all their attributes and have no id field. (While mirroring the internal aggregate structure in the command might not be recommended as a best practice in DDD, we choose this tradeoff to increase transparency when modeling.) Model each command attribute according to one of the following 4 attribute categories and rules:

1. Simple Attributes (Primitive Values): Implicitly typed as string, number, or boolean. Use this for basic input values. No nested sub-fields. Examples: comment, quantity, isApproved. Usually mapped to simple attributes of the aggregate root.
2. References (ID Only): Implicitly typed as string. Use this when referencing an existing entity from another bounded context. Field name must end with Id. No nested sub-fields. Represents a reference only (no mutation of referenced entity). Examples: userId, paymentId, productId. Usually a string attribute on the aggregate root.
3. Referenced Entity or Value Object (same bounded context): Implicitly typed as object but must never contain any nested sub-fields. Can be a single reference or a collection; set `cardinality` to `"one-to-one"` for a single reference and `"one-to-many"` for a collection. Use this when the referenced entity or value object lives within the same bounded context but outside the current aggregate's consistency boundary, and is not mutated. Unlike a Category 2 reference, the structural attributes of the referenced entity or value object are known (although not included here in the command); unlike Category 4, it is not mutated. Field name must be in camelCase without Id suffix. Do not model any nested sub-fields in your reply. The absence of nested sub-fields shows that we are not mutating the related entity or value object (mutations use Category 4). Omit the Id suffix, since the technical decision of how the reference is implemented is a later concern. Examples: currency, country, shippingMethod.
4. Nested Related Entity or Value Object to Be Mutated: Implicitly typed as an object. It must always contain at least one nested sub-field. It may be a single object or a collection; set `cardinality` to `"one-to-one"` for a single object and `"one-to-many"` for a collection. Use this when creating or updating one or more nested entities or VOs inside the aggregate. This field name must be identical to the camelCase form of the targeted nested entity or VO name. Never use generic or implementation-specific command argument names such as options, data, params, or payload when targeting an underlying entity or VO. The nested sub-field names must be identical to the attribute names of the targeted related entity or VO. Nested attributes may themselves be Category 4 fields, with their own second level of nested sub-fields. For example, if the aggregate has an entity/VO structure with three levels — order => orderItems => taxRate — then the command setTaxRate should mirror all levels in its argument structure: setTaxRate({ id: "order-123", orderItems: [{ id: "item-456", taxRate: { percentage: 25, countryCode: "SE" } }] }). Note that the id fields are named exactly id and that the value object has no id. Follow this pattern even for batch operations that update multiple nested items: mirror all nested levels in the command structure.

**Every event should have a command.** After creating commands, call `get_workflow` and verify there
are no events without a command card. Events without commands represent gaps in the business process.

- **Search/filter parameters do NOT belong on commands** — they belong on Read Models with `isFilter: true`

**Step 8 — Create read models on events** *(see `references/read-model-generation.md` for detailed field rules)*

Call `create_read_model` for each data retrieval view. Each read model is automatically attached
to an event via the required `domainEvent` parameter. This auto-creates the Read Model card.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Requires `cardinality`: `"one-to-one"` for single-record queries, `"one-to-many"` for list queries
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 5

**Reuse read models across events when it makes sense.**

Multiple events in the same workflow often need the same query — for example, several events along an Order's lifecycle all need "Get Order
Details". Instead of creating a separate near-duplicate read model per event, just call`create_read_model` again with the **same name** on
the new event. The backend recognizes the existing query, attaches the new event's card to the same shared schema, and merges any new fields
you passed into that schema. Judge reuse case-by-case: if a new event genuinely needs the same data shape and the same queried entity as an
existing one, reuse it (same name); if the shape or entity differs, pick a different name and create a distinct read model.

**Read model field rules (API response payloads):**

Read models represent what the API returns. Fields can be richer than command fields:

- Set `isFilter: true` for query parameters (what you search/filter by). Filter fields can be cross-entity parameters (e.g., `checkInDate`, `priceMin` on a hotel search) — this is expected and valid
- Omit `isFilter` for returned data fields
- **Use `relatedEntity` for composed response data** — nested objects make sense in responses (e.g., `guest` field with relatedEntity Guest containing `firstName`, `lastName`, `email`)
- Name nested object fields as the entity name (`guest`, `hotel`, `room`), NOT with "Id" suffix

**Step 9 — Create domain event schemas on events (optional)** *(see `references/domain-event-generation.md` for detailed field rules)*

> **Skip this step by default.** Domain event schemas describe the payload published when an event fires, which is only meaningful in **Event Sourcing** architectures where events are the source of truth. Only perform this step if:
>
> - The user explicitly asks for it (e.g., "add domain event schemas", "set up event payloads", "we need the event contracts"), OR
> - You are reverse-engineering a codebase that already uses Event Sourcing (events are persisted and replayed to rebuild state)
>
> For regular CRUD/state-based applications, leave events without schemas — the workflow is complete without them. If unsure, ask the user before creating schemas.

Call `create_domain_event_schema` for each event to define the data payload published when
the event occurs. Each schema is automatically attached to an event via the required `domainEvent`
parameter. This auto-creates the Domain Event card.

- Name matching the event (e.g., "Order Created", "Payment Processed")
- Link to the aggregate root entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Include fields that capture the essential facts about the state change
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 5

**Domain event schema field rules (event payload):**

Domain event schemas represent what is published when the event fires. Fields should capture what
happened, not the full entity state:

- Include identifier fields (e.g., `orderId`, `customerId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- Use nested fields with `relatedEntity` for embedded event data (e.g., order items in the event)
- Keep it focused — usually 3 to 8 fields are sufficient
- Field names should be consistent with command and entity field names

**Step 10 — Update entities with full fields** *(see `references/entity-generation.md` for detailed derivation rules)*

Now that all commands and read models exist (plus any domain event schemas, if Step 9 was performed),
update each entity with its real fields. At this point you have full visibility into every schema
that references each entity, so you can derive the correct set of fields.

```
update_entity(
  workflowId: "wf-1",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "id", dataType: "string", description: "Unique identifier of the order", exampleData: ["ord-001", "ord-002", "ord-003"], isRequired: true },
    { name: "customerId", dataType: "string", description: "Customer who placed the order", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
    { name: "status", dataType: "string", description: "Current fulfillment status", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
    { name: "orderItems", dataType: "object", description: "Line items in the order", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many", exampleData: ["Object", "Object", "Object"] },
    { name: "createdAt", dataType: "string", description: "Timestamp when the order was placed", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
  ]
)
```

**Entity field design rules:**

- Include ALL unique fields from all commands that target this entity — merge fields across commands
- Include `relatedEntity` + `cardinality` for relationships to other entities
- **Always include `exampleData`** (3 realistic values per field). For fields with `relatedEntity`, use the placeholder `["Object", "Object", "Object"]`
- **Always include `description`** — a concise one-sentence explanation in business language
- Include `dataType`: `string`, `number`, `boolean`, `object`
- Mark fields essential for creation with `isRequired: true`
- Fields populated only during specific lifecycle transitions (cancel, archive, complete) should NOT be required

### Phase 4: Validation Loop

**Step 11 — Validate and fix the domain model**

Run `validate_domain_model` to check for structural issues. This catches field mismatches between
commands/read models and their aggregate root entities.

**This is an iterative loop, not a one-shot check:**

1. Run `validate_domain_model`
2. For each issue (MAJOR or MINOR), **judge whether it genuinely indicates a problem** or is a legitimate domain pattern:
   - **If it's a real problem** → fix it
   - **If it's a valid domain pattern** → leave it as-is
3. Re-run `validate_domain_model`
4. **Repeat until all remaining issues are legitimate patterns you've consciously decided to keep**

**Do NOT use severity (MAJOR vs MINOR) as the sole deciding factor.** Both need judgment. Some "MAJOR" issues can be valid; some "MINOR" issues should be fixed.

**Common real problems to fix:**

- **Command field not on entity when it should be stored** → Add the missing field to the entity via `update_entity`, or remove the field from the command via `update_command`
- **Missing entity relationship** that the business domain requires → Add `relatedEntity` field to entity via `update_entity`
- **Denormalized fields on commands** (e.g., `guestEmail` when `guestId` already exists) → Replace with flat ID ref and let the service look up related data internally
- **Typos or inconsistent naming** between command/read model and entity fields → Rename for consistency

**Common legitimate patterns to leave as-is:**

- **Calculated/derived fields on read models** (e.g., `total`, `subtotal`, `taxTotal`, `shippingTotal` on a "Get Cart Totals" query) — these are computed at runtime from entity fields, not stored on the entity. `FIELD_NOT_IN_ENTITY` warnings on these are expected and correct
- **Aggregated fields on read models** (e.g., `orderCount`, `averageRating`, `totalSpent`) — same principle: computed from queries over other data
- **Cross-entity filter parameters** on read models (e.g., `checkInDate` on a Hotel search, `minPrice` on a Product search) — filter fields for search/query parameters don't need to exist on the entity
- **Formatted/presentation fields** on read models (e.g., `fullName` when entity has `firstName` + `lastName`, `displayAddress` when entity has separate address components) — these are projections for display, not storage
- **Status/computed flags on read models** (e.g., `isOverdue`, `isExpired`) — derived from date comparisons at query time

**How to judge:** Ask yourself "In a real application, would this field be **stored** on the database table for this entity, or would it be **calculated on the fly** when the query runs?" If the answer is "calculated at runtime" or "joined from another table at query time", the field is a legitimate read model field and the warning can be ignored.

**Important:** Do not consider the workflow complete until every remaining issue has been consciously reviewed and either fixed or judged as a legitimate pattern.

### Phase 5: Polish (optional)

These steps are cosmetic and can be skipped if not needed.

**Step 12 — Create groups (optional, only on explicit user request)**

> **Do NOT create groups as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "group the events into phases", "add groups for the checkout/fulfillment stages", "organize events into stages". A newly generated workflow should have zero groups by default.

Groups organize events into visual vertical phases on the diagram. When the user asks for them:

1. Call `create_group` for each phase, in the order they appear on the timeline.
2. Assign events to groups using `update_domain_event` with the `group` parameter. Only set `group` on the **first event** that starts a new group — subsequent events in the same group inherit it automatically based on their position on the timeline.

Common phase patterns (use these as inspiration only when the user asks):

- E-commerce: "Browse & Cart", "Checkout", "Fulfillment"
- Onboarding: "Registration", "Verification", "Setup", "Activation"

**Step 13 — Set colors (optional, only on explicit user request)**

> **Do NOT set colors as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "color-code the events by lane", "make the automation events blue", "highlight the checkout events in green". A newly generated workflow should leave every event on its default peach color.

When the user asks, call `update_domain_event` with the `color` parameter. Available colors: peach, yellow, green, teal, blue, lavender, pink, gray.

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", color: "blue")
```

## Constraints and Rules

- **One Aggregate Root card per event** — An event can only have one entity linked as aggregate root
- **One Command card per event** — An event can only have one command
- **One Domain Event card per event** — An event can only have one domain event schema (when one is created)
- **Every event should have an aggregate root and a command** — No naked events. Domain event schemas are optional and only expected for Event Sourcing workflows (see Step 9)
- **Read Model cards require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes cannot be deleted if they contain events** — Move or delete events first
- **Domain events require a lane** — Every event must be assigned to a lane
- **Chain events via follows** — Use "start" for root events, otherwise reference the parent via `$ref` path
- **Bounded contexts must exist before referencing them** — Create BCs before assigning entities to them

## `relatedEntity` Usage Summary


| Context                       | Use `relatedEntity`?                          | Example                                     |
| ----------------------------- | --------------------------------------------- | ------------------------------------------- |
| Command: simple ID lookup     | **NO** — use flat string field                | `bookingId: "bk-001"`                       |
| Command: embedded collection  | **YES** — multiple fields needed              | `orderItems: [{ productName, qty, price }]` |
| Read Model: composed response | **YES** — nested joined data                  | `guest: { firstName, lastName, email }`     |
| Read Model: filter parameter  | **NO** — use flat field with `isFilter: true` | `checkInDate` (isFilter)                    |
| Domain Event: embedded data   | **YES** — nested event payload data           | `orderItems: [{ productId, qty, price }]`   |
| Domain Event: ID reference    | **NO** — use flat field                       | `orderId`, `customerId`                     |
| Entity: relationship          | **YES** — defines data model links            | `order → OrderItem (one-to-many)`           |


**Naming rule:** If using `relatedEntity`, name the field as the entity (`guest`, `hotel`, `orderItems`).
If it's a flat ID reference, name it with "Id" suffix (`guestId`, `hotelId`). Never combine "Id" suffix
with `relatedEntity`.

## Common Workflow Patterns

### CRUD Service

Lanes: User, Automation
Flow: User action event → Automation processing event per operation

### Approval Pipeline

Lanes: Requester, Approver, Automation
Flow: Request submitted → Review pending → Approved/Rejected gateway → Executed

### Event-Driven Saga

Lanes: Customer, Admin, Automation
Flow: Customer/Admin actions trigger events, Automation handles cross-service coordination via decision gateways

## Tips for Well-Structured Workflows

- **Start with events, not entities.** Map the business process flow first, then identify what data each step needs.
- **Use decision gateways sparingly.** Only for genuine branching logic, not optional steps.
- **2-4 lanes is ideal.** Typically, 1-2 human roles + Automation. More than 4 makes the diagram hard to read.
- **Name events as past-tense occurrences.** "Order Created", not "Create Order" — commands go on cards.
- **Avoid special characters in event names.** Use only alphanumeric characters and spaces. No `?`, `!`, `&`, `#`.
- **Include realistic example data.** 3 values per field helps stakeholders understand the model.
- **After completing the workflow, use the `/download` skill** to save the specification to a file. This is much faster than fetching via MCP tools for large workflows.

## Additional Resources

### Reference Files

Consult these for detailed rules when creating specific element types:

- **`references/system-prompt.md`** — DDD/event modeling expert context, naming conventions, character limits, quality checks
- **`references/event-generation.md`** — Event naming rules, role conventions, chronology, gateway patterns
- **`references/command-generation.md`** — The 4 command field categories, naming rules, 3-level nesting guidelines
- **`references/read-model-generation.md`** — Read model field design, filter vs display fields, cardinality rules
- **`references/domain-event-generation.md`** — Domain event payload rules, identifier/timestamp conventions
- **`references/entity-generation.md`** — Entity field derivation from commands, merge strategy, entity vs value object rules
- **`references/tools.md`** — Complete MCP tool reference with parameters and usage tips

### Example Files

For a complete worked example showing all steps with realistic tool calls and data:

- **`examples/ecommerce-workflow.md`** — End-to-end e-commerce order workflow creation: empty entities first, then commands/read models/domain events referencing them, entity fields derived last, validation loop

