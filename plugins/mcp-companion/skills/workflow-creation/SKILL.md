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

- **Lanes** — Horizontal swim lanes representing actors or systems (e.g., "Customer", "Payment Service")
- **Groups** — Vertical columns representing phases or stages (e.g., "Order Placement", "Fulfillment")
- **Domain Events** — The core building blocks placed at lane/group intersections. Each event represents something that
  happens in the business process (e.g., "Order Created", "Payment Received")
- **Entities** — Persistent domain objects (Aggregate Roots) with typed fields and relationships
- **Commands** — State-changing operations attached to events, with input fields
- **Read Models** — Data queries/views attached to events, with filter and output fields
- **Bounded Contexts** — Logical boundaries grouping related entities

### How elements reference each other

Tools use `$ref` paths to link elements together. The `get_workflow` tool returns the full workflow specification
where each element has a `$ref` path:

- Domain events: `#/domainEvents/OrderPlaced`
- Entities: `#/schemas/entities/Order`
- Commands: `#/schemas/commands/CreateOrder`
- Read models: `#/schemas/readModels/GetOrderDetails`

Use these paths when creating commands, read models, or referencing entities in fields.

## Creation Sequence

Follow these steps in order. Each step depends on the previous one.

### Phase 1: Foundation

**Step 1 — Identify or create the workflow**

For an existing workflow, call `list_workflows` to find it, then `get_workflow` to understand
its current state. For a new workflow, call `create_workflow` with a descriptive name.

**Step 2 — Create lanes**

Every event must belong to a lane. Call `create_lane` for each actor or system involved in the process.
Aim for 2-5 lanes. Common patterns:
- User-facing: "Customer", "Admin", "Manager"
- System: "Order Service", "Payment Gateway", "Notification System"
- External: "Third-party API", "Bank"

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
- `group` — Sets a group boundary starting at this event. Only set on the **first** event of a new group.
  Do not set group on subsequent events in the same group.
- `aggregateRoot` — A `$ref` path to an entity (e.g., `#/schemas/entities/Order`). Links the entity as
  aggregate root on this event.
- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings.
- `color` — Visual color: peach, yellow, green, teal, blue, lavender, pink, gray

Build the flow left-to-right, top-to-bottom, creating events in the order they occur in the
business process.

### Phase 3: Domain Model

**Step 5 — Create entities**

Call `create_entity` for each core domain object. Entities must be created before commands and
read models so they can be referenced.

- Include fields with `name`, `dataType`, `exampleData` (3 realistic values), `isRequired`
- Use `relatedEntity` (`$ref` path like `#/schemas/entities/OrderItem`) and `cardinality` to express
  entity relationships from the owning entity's perspective
- Optionally assign to a bounded context by name using the `boundedContext` parameter
- Field types: `string`, `number`, `boolean`, `object`

**Step 6 — Create commands on events**

Call `create_command` for each state-changing operation. Each command is automatically attached
to an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).
This auto-creates the Command card on that event.

- Name with action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Command fields should correspond to fields on the aggregate root entity they modify
- Mark auto-generated fields (IDs, timestamps) with `hideInForm: true`
- Use `relatedEntity` ($ref path) and `cardinality` on fields to reference related entities,
  with nested `fields` containing field names from that entity

**Step 7 — Create read models on events**

Call `create_read_model` for each data retrieval view. Each read model is automatically attached
to an event via the required `domainEvent` parameter. This auto-creates the Read Model card.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Requires `cardinality`: `"one-to-one"` for single-record queries, `"one-to-many"` for list queries
- Fields represent the full query contract: both inputs and outputs
- Set `isFilter: true` for query parameters (what you search by), omit for returned data fields
- Use `relatedEntity` on return fields that reference other entities, with nested `fields` for their field names

### Phase 4: Organization

**Step 8 — Create bounded contexts**

Group related entities into bounded contexts for logical separation. Call `create_bounded_context`
and then update entities to assign them to contexts (if not already assigned in Step 5).
Required before generating OpenAPI specs.

## Constraints and Rules

- **One Aggregate Root card per event** — An event can only have one entity linked as aggregate root
- **One Command card per event** — An event can only have one command
- **Read Model cards require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes and groups cannot be deleted if they contain events** — Move or delete events first
- **Domain events require a lane** — Every event must be assigned to a lane
- **Chain events via follows** — Use "start" for root events, otherwise reference the parent via `$ref` path

## Common Workflow Patterns

### CRUD Service
Lanes: User, Service | Groups: Create, Read, Update, Delete
Flow: User action event → Service processing event per operation

### Approval Pipeline
Lanes: Requester, Approver, System | Groups: Submit, Review, Execute
Flow: Request submitted → Review pending → Approved/Rejected gateway → Executed

### Event-Driven Saga
Lanes: Service A, Service B, Orchestrator | Groups per saga step
Flow: Each service emits events, orchestrator coordinates via decision gateways

## Tips for Well-Structured Workflows

- **Start with events, not entities.** Map the business process flow first, then identify what data each step needs.
- **Use decision gateways sparingly.** Only for genuine branching logic, not optional steps.
- **Color-code by domain.** Use consistent colors for events in the same bounded context.
- **3-5 lanes is ideal.** More than 5 makes the diagram hard to read.
- **Name events as past-tense occurrences.** "Order Created", not "Create Order" — commands go on cards.
- **Include realistic example data.** 3 values per field helps stakeholders understand the model.

## Additional Resources

### Reference Files

For detailed natural-language descriptions of every MCP tool, parameters, and usage tips:
- **`references/tools.md`** — Complete tool reference with natural-language descriptions, parameters, and usage tips

### Example Files

For a complete worked example showing all 8 steps with realistic tool calls and data:
- **`examples/ecommerce-workflow.md`** — End-to-end e-commerce order workflow creation with 3 lanes, 6 events, 2
  entities, 3 commands, 2 read models, and a decision gateway
