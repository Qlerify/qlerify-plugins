---
name: workflow-creation
description: >
  This skill should be used when the user asks to "create a workflow",
  "build a domain model", "set up a new Qlerify workflow", "add events and lanes",
  "create BPMN diagram", "model a business process", "set up domain events",
  "add commands and read models to workflow", or any request involving building
  a Qlerify workflow from scratch or adding structural elements to an existing one.
  Provides the full creation sequence, tool ordering, and domain modeling guidance.
allowed-tools: Read, Glob, Grep
---

# Workflow Creation Guide

Build well-structured Qlerify workflows by following a specific tool sequence. Skipping steps
or calling tools out of order leads to broken references and incomplete diagrams.

## Core Concepts

A Qlerify workflow is a BPMN-style diagram combined with domain-driven design (DDD) elements:

- **Lanes** — Horizontal swim lanes representing real-world actor roles or automation (e.g., "Customer", "Hotel Staff", "Automation")
- **Groups** — Vertical columns representing phases or stages (e.g., "Order Placement", "Fulfillment")
- **Domain Events** — The core building blocks placed at lane/group intersections. Each event represents something that happens in the business process (e.g., "Order Created", "Payment Received")
- **Entities** — Persistent domain objects (Aggregate Roots) with typed fields and relationships
- **Commands** — State-changing operations attached to events, representing API request payloads
- **Read Models** — Data queries/views attached to events, representing API response payloads
- **Domain Event Schemas** — Data payloads carried when events occur, capturing the essential facts about what happened
- **Bounded Contexts** — Logical boundaries grouping related entities, mapping to deployment/service boundaries

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

**Note:** Gateway events (`bpmn:ExclusiveGateway`) may not appear in the `get_workflow` domainEvents section.
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

**Step 3 — Create groups**

Groups organize events into phases. Call `create_group` for each phase in chronological order.
Groups are positioned automatically — just create them in the order they should appear. Common patterns:
- E-commerce: "Browse", "Cart", "Checkout", "Fulfillment", "Post-delivery"
- Onboarding: "Registration", "Verification", "Setup", "Activation"

### Phase 2: Event Flow

**Step 4 — Create domain events**

Build the event flow by chaining calls to `create_domain_event`. Each event needs:
- `lane` — The lane name (e.g., "Customer")
- `follows` — A `$ref` path to the preceding event, or `"start"` for flow entry points
- `type` — Either `bpmn:Task` (regular event) or `bpmn:ExclusiveGateway` (decision diamond)

Optional parameters:
- `group` — Sets a group boundary starting at this event. Only set on the **first** event of a new group. Do not set group on subsequent events in the same group.
- `aggregateRoot` — A `$ref` path to an entity (e.g., `#/schemas/entities/Order`). Links the entity as aggregate root on this event. **Set this for every event** — see note below.
- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings.
- `color` — Visual color: peach, yellow, green, teal, blue, lavender, pink, gray

Build the flow left-to-right, top-to-bottom, creating events in the order they occur in the
business process.

**Aggregate root requirement:** Every domain event SHOULD have an aggregate root — it identifies which
entity this event primarily affects. If the entity doesn't exist yet when creating the event, set the
aggregate root later in Step 5b using `update_domain_event` after entities are created. Do not leave
events without aggregate roots.

### Phase 3: Domain Model

**Step 5a — Create bounded contexts**

Create bounded contexts BEFORE entities, so entities can be assigned during creation. Call
`create_bounded_context` for each logical boundary.

**Bounded context design rules:**

- A bounded context maps to a **microservice or standalone service deployment boundary** — don't create more than you would actually deploy as separate services
- For small/medium workflows (< 10 entities, single team), **one bounded context wrapping everything is fine**
- Only split into multiple BCs when there are genuinely independent domains with clear interaction boundaries
- Ask: "Would a team realistically build and deploy this as a separate service?" If not → same BC
- Over-splitting creates unnecessary inter-service complexity (distributed transactions, API contracts, eventual consistency)
- Common pattern: start with 1 BC, split later when the domain grows and clear boundaries emerge

**Examples:**

- Hotel Booking (6 entities): 1 BC — "Hotel Booking"
- Large E-commerce Platform (20+ entities): 3-4 BCs — "Order Management", "Inventory", "Customer Management", "Payments"

**Step 5b — Create entities and link aggregate roots**

Call `create_entity` for each core domain object. Entities must be created before commands and
read models so they can be referenced.

- Include fields with `name`, `dataType`, `exampleData` (3 realistic values), `isRequired`
- Use `relatedEntity` (`$ref` path like `#/schemas/entities/OrderItem`) and `cardinality` to express entity relationships from the owning entity's perspective
- Assign to a bounded context by name using the `boundedContext` parameter (the BC must already exist from Step 5a)
- Field types: `string`, `number`, `boolean`, `object`

After creating entities, update any events that don't have aggregate roots yet:

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", aggregateRoot: "#/schemas/entities/Order")
```

**Every event must have an aggregate root before proceeding.**

**Step 6 — Create commands on events**

Call `create_command` for each state-changing operation. Each command is automatically attached
to an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).
This auto-creates the Command card on that event.

- Name with action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Command fields should correspond to fields on the aggregate root entity they modify
- Mark auto-generated fields (IDs, timestamps) with `hideInForm: true`

**Every event should have a command.** After creating commands, call `get_workflow` and verify there
are no events without a command card. Events without commands represent gaps in the business process.

**Command field rules (API request payloads):**

Commands represent what a caller sends to perform an action. Fields should be flat and simple:

- **Use plain fields for ID references:** `bookingId` (string), `hotelId` (string), `customerId` (string). Do NOT use `relatedEntity` for simple ID lookups — it creates nonsensical nested structures like `{ hotelId: { id: "123" } }` instead of the correct `{ hotelId: "123" }`
- **Only use `relatedEntity` on commands for embedded collections** where you need to send multiple fields from a related entity (e.g., `orderItems` with `productName`, `quantity`, `unitPrice`)
- **Search/filter parameters do NOT belong on commands** — they belong on Read Models with `isFilter: true`
- **NEVER combine an "Id" suffix field name with `relatedEntity`** — if the field ends in "Id", it's a flat string reference, not a nested object

**Step 7 — Create read models on events**

Call `create_read_model` for each data retrieval view. Each read model is automatically attached
to an event via the required `domainEvent` parameter. This auto-creates the Read Model card.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Requires `cardinality`: `"one-to-one"` for single-record queries, `"one-to-many"` for list queries

**Read model field rules (API response payloads):**

Read models represent what the API returns. Fields can be richer than command fields:

- Set `isFilter: true` for query parameters (what you search/filter by). Filter fields can be cross-entity parameters (e.g., `checkInDate`, `priceMin` on a hotel search) — this is expected and valid
- Omit `isFilter` for returned data fields
- **Use `relatedEntity` for composed response data** — nested objects make sense in responses (e.g., `guest` field with relatedEntity Guest containing `firstName`, `lastName`, `email`)
- Name nested object fields as the entity name (`guest`, `hotel`, `room`), NOT with "Id" suffix

**Step 8 — Create domain event schemas on events**

Call `create_domain_event_schema` for each event to define the data payload published when
the event occurs. Each schema is automatically attached to an event via the required `domainEvent`
parameter. This auto-creates the Domain Event card.

- Name matching the event (e.g., "Order Created", "Payment Processed")
- Link to the aggregate root entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Include fields that capture the essential facts about the state change

**Domain event schema field rules (event payload):**

Domain event schemas represent what is published when the event fires. Fields should capture what
happened, not the full entity state:

- Include identifier fields (e.g., `orderId`, `customerId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- Use nested fields with `relatedEntity` for embedded event data (e.g., order items in the event)
- Keep it focused — usually 3 to 8 fields are sufficient
- Field names should be consistent with command and entity field names

### Phase 4: Validation

**Step 9 — Validate the domain model**

Run `validate_domain_model` to check for structural issues. This catches field mismatches between
commands/read models and their aggregate root entities.

- Fix all **MAJOR** issues before considering the workflow complete
- **MINOR** `FIELD_NOT_IN_ENTITY` issues on read model filter fields are often expected — cross-entity query parameters (e.g., `checkInDate` on a Hotel read model) are valid filter fields that don't need to exist on the entity

Common validation issues and fixes:

- Command field not on entity → Remove from command or add to entity
- Missing entity relationship → Add `relatedEntity` field to entity
- Denormalized fields on commands (e.g., `guestEmail`) → Replace with flat ID ref (`guestId`) and let the service look up related data internally

## Constraints and Rules

- **One Aggregate Root card per event** — An event can only have one entity linked as aggregate root
- **One Command card per event** — An event can only have one command
- **One Domain Event card per event** — An event can only have one domain event schema
- **Every event should have an aggregate root, a command, and a domain event schema** — No naked events
- **Read Model cards require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes and groups cannot be deleted if they contain events** — Move or delete events first
- **Domain events require a lane** — Every event must be assigned to a lane
- **Chain events via follows** — Use "start" for root events, otherwise reference the parent via `$ref` path
- **Bounded contexts must exist before referencing them** — Create BCs before assigning entities to them

## `relatedEntity` Usage Summary

| Context                       | Use `relatedEntity`?                          | Example                                     |
|-------------------------------|-----------------------------------------------|---------------------------------------------|
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

Lanes: User, Automation | Groups: Create, Read, Update, Delete
Flow: User action event → Automation processing event per operation

### Approval Pipeline

Lanes: Requester, Approver, Automation | Groups: Submit, Review, Execute
Flow: Request submitted → Review pending → Approved/Rejected gateway → Executed

### Event-Driven Saga

Lanes: Customer, Admin, Automation | Groups per saga step
Flow: Customer/Admin actions trigger events, Automation handles cross-service coordination via decision gateways

## Tips for Well-Structured Workflows

- **Start with events, not entities.** Map the business process flow first, then identify what data each step needs.
- **Use decision gateways sparingly.** Only for genuine branching logic, not optional steps.
- **Color-code by domain.** Use consistent colors for events in the same bounded context.
- **2-4 lanes is ideal.** Typically, 1-2 human roles + Automation. More than 4 makes the diagram hard to read.
- **Name events as past-tense occurrences.** "Order Created", not "Create Order" — commands go on cards.
- **Avoid special characters in event names.** Use only alphanumeric characters and spaces. No `?`, `!`, `&`, `#`.
- **Include realistic example data.** 3 values per field helps stakeholders understand the model.
- **After completing the workflow, use the `/download` skill** to save the specification to a file. This is much faster than fetching via MCP tools for large workflows.

## Additional Resources

### Reference Files

For detailed natural-language descriptions of every MCP tool, parameters, and usage tips:
- **`references/tools.md`** — Complete tool reference with natural-language descriptions, parameters, and usage tips

### Example Files

For a complete worked example showing all 9 steps with realistic tool calls and data:
- **`examples/ecommerce-workflow.md`** — End-to-end e-commerce order workflow creation with 3 lanes, 7 events, 2 entities, 4 commands, 2 read models, domain event schemas, a decision gateway, and validation
