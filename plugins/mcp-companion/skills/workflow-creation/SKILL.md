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
- **Domain Events** — The core building blocks placed in lanes. Each event represents something that happens in the business process (e.g., "Order Created", "Payment Received")
- **Entities** — Persistent domain objects (Aggregate Roots) with typed fields and relationships
- **Commands** — State-changing operations attached to events, representing API request payloads
- **Read Models** — Data queries/views attached to events, representing API response payloads
- **Domain Event Schemas** — Data payloads carried when events occur, capturing the essential facts about what happened
- **Bounded Contexts** — Logical boundaries grouping related entities, mapping to deployment/service boundaries
- **Groups** — Optional vertical columns representing phases or stages (e.g., "Order Placement", "Fulfillment")

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

### Phase 2: Event Flow

**Step 3 — Create domain events**

Build the event flow by chaining calls to `create_domain_event`. Each event needs:
- `lane` — The lane name (e.g., "Customer")
- `follows` — A `$ref` path to the preceding event, or `"start"` for flow entry points
- `type` — Either `bpmn:Task` (regular event) or `bpmn:ExclusiveGateway` (decision diamond)

Optional parameters:
- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings

Build the flow left-to-right, top-to-bottom, creating events in the order they occur in the
business process. Do NOT set `aggregateRoot` yet — entities don't exist at this point.
Do NOT set `group` yet — groups are optional and created at the end.

### Phase 3: Domain Model

**Step 4 — Create bounded contexts**

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

**Step 5 — Create empty entities**

Create entities with just their name and bounded context — **no fields yet**. This establishes
`$ref` paths so commands, read models, and domain event schemas can reference them.

```
create_entity(workflowId: "wf-1", name: "Order", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/Order" }

create_entity(workflowId: "wf-1", name: "Order Item", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/OrderItem" }
```

Identify ALL entities and value objects the domain needs — aggregate roots, child entities,
and value objects. Create them all now so their `$ref` paths are available for subsequent steps.

**Step 6 — Link aggregate roots to events**

Now that entities exist, link each event to its aggregate root entity:

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", aggregateRoot: "#/schemas/entities/Order")
```

**Every event must have an aggregate root before proceeding.** This identifies which entity each event primarily affects.

**Step 7 — Create commands on events**

Call `create_command` for each state-changing operation. Each command is automatically attached
to an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).
This auto-creates the Command card on that event.

- Name with action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Mark auto-generated fields (IDs, timestamps) with `hideInForm: true`
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 5

**Every event should have a command.** After creating commands, call `get_workflow` and verify there
are no events without a command card. Events without commands represent gaps in the business process.

**Command field rules (API request payloads):**

Commands represent what a caller sends to perform an action. Fields should be flat and simple:

- **Use plain fields for ID references:** `bookingId` (string), `hotelId` (string), `customerId` (string). Do NOT use `relatedEntity` for simple ID lookups — it creates nonsensical nested structures like `{ hotelId: { id: "123" } }` instead of the correct `{ hotelId: "123" }`
- **Only use `relatedEntity` on commands for embedded collections** where you need to send multiple fields from a related entity (e.g., `orderItems` with `productName`, `quantity`, `unitPrice`)
- **Commands support up to 3 levels of nesting** — a field can have nested `fields` with `relatedEntity`, and those nested fields can also have their own `fields` with `relatedEntity` for deeper structures
- **Search/filter parameters do NOT belong on commands** — they belong on Read Models with `isFilter: true`
- **NEVER combine an "Id" suffix field name with `relatedEntity`** — if the field ends in "Id", it's a flat string reference, not a nested object

**Step 8 — Create read models on events**

Call `create_read_model` for each data retrieval view. Each read model is automatically attached
to an event via the required `domainEvent` parameter. This auto-creates the Read Model card.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Requires `cardinality`: `"one-to-one"` for single-record queries, `"one-to-many"` for list queries
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 5

**Read model field rules (API response payloads):**

Read models represent what the API returns. Fields can be richer than command fields:

- Set `isFilter: true` for query parameters (what you search/filter by). Filter fields can be cross-entity parameters (e.g., `checkInDate`, `priceMin` on a hotel search) — this is expected and valid
- Omit `isFilter` for returned data fields
- **Use `relatedEntity` for composed response data** — nested objects make sense in responses (e.g., `guest` field with relatedEntity Guest containing `firstName`, `lastName`, `email`)
- Name nested object fields as the entity name (`guest`, `hotel`, `room`), NOT with "Id" suffix

**Step 9 — Create domain event schemas on events**

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

**Step 10 — Update entities with full fields**

Now that all commands, read models, and domain event schemas exist, update each entity with its
real fields. At this point you have full visibility into every schema that references each entity,
so you can derive the correct set of fields.

```
update_entity(
  workflowId: "wf-1",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "id", dataType: "string", exampleData: ["ord-001", "ord-002", "ord-003"], isRequired: true },
    { name: "customerId", dataType: "string", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
    { name: "status", dataType: "string", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
    { name: "orderItems", dataType: "object", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many" },
    { name: "createdAt", dataType: "string", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
  ]
)
```

**Entity field design rules:**

- Include ALL unique fields from all commands that target this entity — merge fields across commands
- Include `relatedEntity` + `cardinality` for relationships to other entities
- Include `exampleData` (3 realistic values per field)
- Include `dataType`: `string`, `number`, `boolean`, `object`
- Mark fields essential for creation with `isRequired: true`
- Fields populated only during specific lifecycle transitions (cancel, archive, complete) should NOT be required

### Phase 4: Validation Loop

**Step 11 — Validate and fix the domain model**

Run `validate_domain_model` to check for structural issues. This catches field mismatches between
commands/read models and their aggregate root entities.

**This is an iterative loop, not a one-shot check:**

1. Run `validate_domain_model`
2. If there are **MAJOR** issues, fix them:
   - **Command field not on entity** → Add the missing field to the entity via `update_entity`, or remove the field from the command via `update_command`
   - **Missing entity relationship** → Add `relatedEntity` field to entity via `update_entity`
   - **Denormalized fields on commands** (e.g., `guestEmail`) → Replace with flat ID ref (`guestId`) and let the service look up related data internally
3. Re-run `validate_domain_model`
4. **Repeat until no MAJOR issues remain**

**MINOR issues:**
- `FIELD_NOT_IN_ENTITY` on read model filter fields are often expected — cross-entity query parameters (e.g., `checkInDate` on a Hotel read model) are valid filter fields that don't need to exist on the entity
- Review MINOR issues but don't blindly fix all of them

**Important:** Do not consider the workflow complete until the validation loop produces zero MAJOR issues.

### Phase 5: Polish (optional)

These steps are cosmetic and can be skipped if not needed.

**Step 12 — Create groups (optional)**

Groups organize events into visual phases on the diagram. Call `create_group` for each phase,
then assign events to groups using `update_domain_event` with the `group` parameter.

Only set `group` on the **first event** that starts a new group — subsequent events in the same
group inherit it automatically based on their position.

Common patterns:
- E-commerce: "Browse & Cart", "Checkout", "Fulfillment"
- Onboarding: "Registration", "Verification", "Setup", "Activation"

**Step 13 — Set colors (optional)**

Events default to peach color. Only set colors if the user specifically requests color-coding.
Available colors: peach, yellow, green, teal, blue, lavender, pink, gray.

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", color: "blue")
```

## Constraints and Rules

- **One Aggregate Root card per event** — An event can only have one entity linked as aggregate root
- **One Command card per event** — An event can only have one command
- **One Domain Event card per event** — An event can only have one domain event schema
- **Every event should have an aggregate root, a command, and a domain event schema** — No naked events
- **Read Model cards require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes cannot be deleted if they contain events** — Move or delete events first
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

For a complete worked example showing all steps with realistic tool calls and data:
- **`examples/ecommerce-workflow.md`** — End-to-end e-commerce order workflow creation: empty entities first, then commands/read models/domain events referencing them, entity fields derived last, validation loop, and optional groups
