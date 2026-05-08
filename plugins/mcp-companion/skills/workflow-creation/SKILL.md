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

A Qlerify workflow is a structured (Software Design Level) Event Storming diagram combined with detailed elements and concepts from domain-driven design (DDD):

- **Lanes** — Horizontal swimlanes representing **roles** — actors that invoke commands. Either a human role (e.g., "Customer", "Hotel Staff") or the exact word "Automation" for any system-triggered step. All automated steps share the single Automation lane — do NOT create lanes for internal services (Payment Service, Notification Service, etc.); those belong in bounded contexts. See "Lane Tools" in `references/tools.md` for full naming rules.
- **Domain Events** — Always placed in sequence within lanes. Each event represents something that happened in a business process (e.g., "Order Created", "Payment Received"). Contains a verb in past tense. A domain event is the result of a role invoking a state-changing command on an aggregate. Valid: Order Created, Invoice Approved, Payment Cancelled. Invalid (no state change): Page Viewed, Form Opened. Domain Events are the starting point of the modeling process — they are usually created before entities, commands, or read models, which are all later linked or attached to events (through Cards).
- **Entities** — An Entity is a domain object that is defined by its identity, not by its attributes. An entity must have an id attribute and a life cycle. Entities can have attributes and other related entities. Some entities play the role of aggregate root from the perspective of a given command. Examples: Order, Customer, Product.
- **Value Objects** — A VO is a domain object that is defined by its attributes. A VO has no identity and is immutable (replaced as a whole when updated). Examples: Money (amount + currency), Address (street + city + postal code), Date Range (start date + end date). A VO must not have an id attribute.
- **Commands** — A Command is a state-changing operation invoked on an aggregate and can trigger a domain event. Commands and domain events have a one-to-one relationship. Commands have arguments (also referred to as fields, attributes or properties) representing data that the actor (human or automation) submits. Examples: "Create Order", "Cancel Subscription", "Add Item to Cart".
- **Read Models** — A Read Model represents the queries, projections or views used to provide the actor with the information needed before invoking a command (used for data retrieval and not state change). A Read Model can have attributes, including filter/search parameters and computed fields, and can have nested related entities or value objects as attributes. Examples: "Get Order Details", "Search Available Rooms", "List Customer Orders".
- **Domain Event Schemas** — Define the structure and meaning of domain event payloads, capturing the essential facts about what happened in the domain.
- **Bounded Contexts** — Logical boundaries that group related domain objects and business rules around a specific capability. They usually map to modules, services, or deployment boundaries. Examples: Order Management, Inventory, Customer Management, Payments.
- **Groups** — Optional visual columns used only to help human viewers organize the diagram into phases or stages, such as Order Placement or Fulfillment. Groups carry no domain, ownership, or architectural meaning. Do not create groups unless the user explicitly asks for them — see Phase 5.

### How elements reference each other

Tools use `$ref` paths to link elements together. The `get_workflow` tool returns the full workflow specification
where each element has a `$ref` path:

- Domain events: `#/domainEvents/OrderPlaced`
- Entities: `#/schemas/entities/Order`
- Value Objects: `#/schemas/valueObjects/Address`
- Commands: `#/schemas/commands/CreateOrder`
- Read models: `#/schemas/queries/GetOrderDetails`
- Domain event schemas: `#/schemas/domainEvents/OrderPlaced`

Use these paths when creating commands, read models, entities, value objects, domain event schemas, or referencing entities in fields.

**Tip:** The workflow specification returned by `get_workflow` includes a `$schema` URL (e.g., `https://app.qlerify.com/schemas/domain-model/v1`). Fetch this JSON Schema to understand the full structure of the spec — field types, allowed values, and relationships between elements.

### Event naming and `$ref` path generation

Event names are converted to PascalCase `$ref` keys:

- Spaces are removed, each word capitalized: `"Order Placed"` → `OrderPlaced`
- Hyphens are removed, next letter capitalized: `"Check-in Completed"` → `CheckInCompleted`
- **Special characters (`?`, `!`, `&`, `#`, `/`) break `$ref` path resolution** — avoid them entirely

Use only alphanumeric characters and spaces in event names. If you use hyphens, call `get_workflow`
afterward to verify the actual `$ref` key before referencing it in subsequent calls.

**Note:** Decisions (`decision`) are visualised as a separate shape on the event storming board taking up the same space as other events, but don't appear in the `get_workflow` domainEvents section. How to find a decision: a domain event is preceded by a decision shape if the domain event has a conditions attribute set. The name of the shape is hidden under the if property ("conditions":[{"after":{"$ref":"#/domainEvents/CustomerCreated"},"if":"Large Customer?","is":"YES"}]). If you need to reference a decision, call `get_workflow`, take the description from the "if" property, PascalCase it (drop spaces and punctuation, capitalize each word), and use it as the $ref segment. For example, a decision described as "Is order paid?" is referenced as #/domainEvents/IsOrderPaid.


## Creation Sequence

Follow these steps in order. Each step depends on the previous one.

### Phase 1: Foundation

**Step 1 — Identify or create the workflow**

For an existing workflow, call `list_workflows` to find it, then `get_workflow` to understand
its current state. For a new workflow, call `create_workflow` with a descriptive name.

### Phase 2: Event Flow

**Step 2 — Create domain events** *(see `references/event-generation.md` for naming and chronology rules)*

Build the event flow by chaining calls to `create_domain_event`. Each event needs:

- `description` — Aggregate Name + Space + Past Tense Verb
- `lane` — The role/actor the event belongs to (e.g., "Customer", "Automation"). Auto-created on the fly the first time an unfamiliar name is referenced, so you typically don't need `create_lane` when building a workflow from scratch. Pass exactly the same name (case-sensitive) across events that share a lane.
- `follows` — A `$ref` path to the preceding event, or `"start"` for flow entry points
- `type` — Either `domainEvent` (regular event) or `decision` (decision diamond)

**Common lane patterns:**

- E-commerce: Customer, Warehouse Worker, Automation
- Hotel Booking: Guest, Hotel Staff, Automation
- HR Onboarding: Candidate, HR Manager, Automation

Optional parameters:

- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings

Build the flow left-to-right, creating events in the order they occur in the business process. Do NOT set `aggregateRoot` yet — entities don't exist at this point.
Do NOT set `group` either — groups are a cosmetic polish step handled later (Phase 5) and only if the user explicitly asks for them.

### Phase 3: Domain Model

**Step 3 — Create bounded contexts**

Create bounded contexts BEFORE entities, so entities can be assigned during creation. Call
`create_bounded_context` for each logical boundary.

**Bounded context design rules:**

- Ask: "Could a team realistically build and deploy this as a separate service?" If not → same BC
- Over-splitting creates unnecessary inter-service complexity (distributed transactions, API contracts, eventual consistency)
- Common pattern: start with 1 BC, split later when the domain grows and clear boundaries emerge

**Examples:**

- Hotel Booking (6 entities): 1 BC — "Hotel Booking"
- Large E-commerce Platform (20+ entities): 4-5 BCs — "Product Catalog", "Order Management", "Inventory", "Customer Management", "Payments"

**Step 4 — Generate entity and value object names, and link aggregate roots to events**

Create entities and value objects with just their name and bounded context, and in the same call link each entity to the events it is the aggregate root for. This establishes `$ref` paths so commands, read models, and domain event schemas can reference them. Classification is determined by the presence of an `id` field — entities have one, value objects do not. Setting this at creation time avoids a follow-up `update_entities` call to convert misclassified value objects.

**Aggregate roots** are linked via `aggregateRootFor` on each entity. The domain event name often hints at its aggregate root — "Order Created" indicates the Order entity. Each event can have only one aggregate root, and value objects cannot be aggregate roots. **Every event must have an aggregate root before proceeding.**

Use `create_entities` to create the schemas. It is a bulk tool: each call accepts an array of entities and value objects, creates them, and links the aggregate roots in one atomic workflow write — so prefer batching when several can be created together.

```
create_entities(workflowId: "wf-1", entities: [
  {
    name: "Order",
    boundedContext: "Order Management",
    fields: [{ name: "id" }],
    aggregateRootFor: ["#/domainEvents/OrderPlaced", "#/domainEvents/OrderCancelled"]
  },
  { name: "Order Item", boundedContext: "Order Management", fields: [{ name: "id" }] },
  { name: "Address", boundedContext: "Order Management" }
])
-> Order, Order Item are entities ($ref under #/schemas/entities/...)
-> Address is a value object ($ref under #/schemas/valueObjects/...)
-> Order is set as aggregate root for OrderPlaced and OrderCancelled
```

Identify ALL entities and value objects the domain needs — aggregate root entities, related or associated entities, and value objects. Use DDD judgment to decide which get the `id` field: things with their own lifecycle and identity are entities; things defined purely by their attributes (Money, Address, DateRange) are value objects.

> If you later need to reassign an aggregate root for a single event, use `update_domain_event(domainEvent: "...", aggregateRoot: "...")`. For initial workflow creation, always set it via `aggregateRootFor` on `create_entities`.

**Step 5 — Create commands on events** *(see `references/command-generation.md` for detailed field rules)*

Use `create_commands` to create the commands. It is a bulk tool: each call accepts an array of commands and attaches them to their events in one atomic workflow write — so prefer batching when several commands can be created together. Commands and domain events have a one-to-one relationship — each command in the batch must target a different event. Associate a command with an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).

- Name commands using action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Use `relatedEntity` when an attribute holds related entities or value objects

For each domain event, specify the name and arguments (also referred to as fields, attributes or properties) of the command that produces the event. Think of the command attributes as information fields that the actor (human or automation) fills in and submits. Each field name must be in camelCase. Commands are always invoked on aggregates through the aggregate root. The command attributes on the first level correspond to attributes on the aggregate root. Fields can be nested up to two levels down from the aggregate root and should be nested when updating underlying related entities or value objects. The command's argument structure must always mirror the structure of the entities and VOs inside the aggregate and their attribute names. Even if the user input explicitly prescribes a flat command structure, make sure to mirror the nested structure of the aggregate. Remember that VOs always carry all their attributes and have no id field. (While mirroring the internal aggregate structure in the command might not be recommended as a best practice in DDD, we choose this tradeoff to increase transparency when modeling.) Model each command attribute according to one of the following 4 attribute categories and rules:

1. Simple Attributes (Primitive Values): Implicitly typed as string, number, or boolean. Use this for basic input values. No nested sub-fields. Examples: comment, quantity, isApproved. Usually mapped to simple attributes of the aggregate root.
2. References (ID Only): Implicitly typed as string. Use this when referencing an existing entity from another bounded context. Field name must end with Id. No nested sub-fields. Represents a reference only (no mutation of referenced entity). Examples: userId, paymentId, productId. Usually a string attribute on the aggregate root.
3. Referenced Entity or Value Object (same bounded context): Implicitly typed as object but must never contain any nested sub-fields. Can be a single reference or a collection; set `cardinality` to `"one-to-one"` for a single reference and `"one-to-many"` for a collection. Use this when the referenced entity or value object lives within the same bounded context but outside the current aggregate's consistency boundary, and is not mutated. Unlike a Category 2 reference, the structural attributes of the referenced entity or value object are known (although not included here in the command); unlike Category 4, it is not mutated. Field name must be in camelCase without Id suffix. Do not model any nested sub-fields in your reply. The absence of nested sub-fields shows that we are not mutating the related entity or value object (mutations use Category 4). Omit the Id suffix, since the technical decision of how the reference is implemented is a later concern. Examples: currency, country, shippingMethod.
4. Nested Related Entity or Value Object to Be Mutated: Implicitly typed as an object. It must always contain at least one nested sub-field. It may be a single object or a collection; set `cardinality` to `"one-to-one"` for a single object and `"one-to-many"` for a collection. Use this when creating or updating one or more nested entities or VOs inside the aggregate. This field name should be the camelCase form of the targeted nested entity or VO name. Exception: when the same Entity/VO type fills multiple semantic roles on the aggregate, name each field after the role and let multiple fields point to the same `relatedEntity` (e.g., `shippingAddress` and `billingAddress` both referencing a VO named "Address"). Never use generic or implementation-specific command argument names such as options, data, params, or payload when targeting an underlying entity or VO. The nested sub-field names must be identical to the attribute names of the targeted related entity or VO. Nested attributes may themselves be Category 4 fields, with their own second level of nested sub-fields. For example, if the aggregate has an entity/VO structure with three levels — order => orderItems => taxRate — then the command setTaxRate should mirror all levels in its argument structure: setTaxRate({ id: "order-123", orderItems: [{ id: "item-456", taxRate: { percentage: 25, countryCode: "SE" } }] }). Note that the id fields are named exactly id and that the value object has no id. Follow this pattern even for batch operations that update multiple nested items: mirror all nested levels in the command structure.

**Every event should have a command.** After creating commands, call `get_workflow` and verify there
are no events without a command card. Events without commands represent gaps in the business process.

- **Search/filter parameters do NOT belong on commands** — they belong on Read Models with `isFilter: true`

**Step 6 — Create read models on events** *(see `references/read-model-generation.md` for detailed field rules)*

Use `create_read_models` to create the read models. It is a bulk tool: each call accepts an array of read models and attaches them to their events in one atomic workflow write — so prefer batching when several can be created together. Each read model is linked to an event via the required `domainEvent` parameter. The read model represents the input data needed by the actor BEFORE triggering the Command that in turn triggers the event.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)

**Reuse read models across events when it makes sense.**

Multiple events in the same workflow often need the same query — for example, several events along an Order's lifecycle all need "Get Order
Details". Instead of creating a separate near-duplicate read model per event, include another entry in the `create_read_models` array with the **same `name`** on the new event. The backend recognizes the shared name, links each event to the same underlying read model, and merges any new fields you passed into it. Judge reuse case-by-case: if a new event genuinely needs the same data shape and the same queried entity as an existing one, reuse the same name; if the shape or entity differs, pick a different name and create a distinct read model.

**Step 7 — Create domain event schemas on events (optional)** *(see `references/domain-event-generation.md` for detailed field rules)*

> **Skip this step by default.** Domain event schemas describe the payload published when an event fires, which is only meaningful in **Event Sourcing** architectures where events are the source of truth. Only perform this step if:
>
> - The user explicitly asks for it (e.g., "add domain event schemas", "set up event payloads", "we need the event contracts"), OR
> - You are reverse-engineering a codebase that already uses Event Sourcing (events are persisted and replayed to rebuild state)
>
> For regular CRUD/state-based applications, leave events without schemas — the workflow is complete without them. If unsure, ask the user before creating schemas.

Use `create_domain_event_schemas` to create the schemas. It is a bulk tool: each call accepts an array of schemas and attaches them to their events in one atomic workflow write — so prefer batching when several can be created together. Each schema is linked to an event via the required `domainEvent` parameter.

- Name matching the event (e.g., "Order Created", "Payment Processed")
- Link to the aggregate root entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Include fields that capture the essential facts about the state change
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 4
- Each schema targets a different event — an event can only have one domain event schema

**Domain event schema field rules (event payload):**

Domain event schemas represent what is published when the event fires. Fields should capture what
happened, not the full entity state:

- Include identifier fields (e.g., `orderId`, `customerId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- Use nested fields with `relatedEntity` for embedded event data (e.g., order items in the event)
- Keep it focused — usually 3 to 8 fields are sufficient
- Field names should be consistent with command and entity field names

**Step 8 — Update entities with full fields** *(see `references/entity-generation.md` for detailed derivation rules)*

Now that all commands and read models exist (plus any domain event schemas, if Step 7 was performed), update each entity with its real fields. At this point you have full visibility into every schema that references each entity, so you can derive the correct set of fields.

Use `update_entities` to apply the updates. It is a bulk tool: each call accepts an array of entity updates and applies them atomically in one workflow write, so prefer batching when several entities can be updated together. Entities created in Step 4 already have an `id` field, so use `addFields` to layer on the rest.

```
update_entities(workflowId: "wf-1", entities: [
  {
    entity: "#/schemas/entities/Order",
    addFields: [
      { name: "customerId", dataType: "string", description: "Customer who placed the order", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
      { name: "status", dataType: "string", description: "Current fulfillment status", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
      { name: "orderItems", dataType: "object", description: "Line items in the order", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many", exampleData: ["Object", "Object", "Object"] },
      { name: "createdAt", dataType: "string", description: "Timestamp when the order was placed", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
    ]
  }
])
```

**Entity field design rules:**

- Include ALL unique fields from all commands that target this entity — merge fields across commands
- Include `relatedEntity` + `cardinality` for relationships to other entities
- **Always include `exampleData`** (3 realistic values per field). For fields with `relatedEntity`, use the placeholder `["Object", "Object", "Object"]`
- **Always include `description`** — a concise one-sentence explanation in business language
- Include `dataType`: `string`, `number`, `boolean`, `object`
- Mark fields essential for creation with `isRequired: true`
- Fields populated only during specific lifecycle transitions (cancel, archive, complete) should NOT be set as required fields

### Phase 4: Validation Loop

**Step 9 — Validate and fix the domain model**

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

- **Command field not on entity when it should be stored** → Add the missing field to the entity via `update_entities`, or remove the field from the command via `update_commands`
- **Missing entity relationship** that the business domain requires → Add `relatedEntity` field to entity via `update_entities`
- **Denormalized fields on commands** (e.g., `guestEmail` when `guestId` already exists) → Replace with flat ID ref and let the service look up related data internally
- **Typos or inconsistent naming** between command/read model and entity fields → Rename for consistency

**Important:** Do not consider the workflow complete until every remaining issue has been consciously reviewed and either fixed or judged as a legitimate pattern.

### Phase 5: Polish (optional)

These steps are cosmetic and can be skipped if not needed.

**Step 10 — Create groups (optional, only on explicit user request)**

> **Do NOT create groups as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "group the events into phases", "add groups for the checkout/fulfillment stages", "organize events into stages". A newly generated workflow should have zero groups by default.

Groups split the workflow into phases seen on the diagram with labels spread out horizontally from left to right at the top of the diagram and vertical dividers splitting the flow into phases. When the user asks for them:

1. Call `create_group` for each phase, in the order they appear on the timeline.
2. Assign events to groups using `update_domain_event` with the `group` parameter. Only set `group` on the **first event** that starts a new group — subsequent events in the same group inherit it automatically based on their position on the timeline.

Common phase patterns (use these as inspiration only when the user asks):

- E-commerce: "Browse & Cart", "Checkout", "Fulfillment"
- Onboarding: "Registration", "Verification", "Setup", "Activation"

**Step 11 — Set colors (optional, only on explicit user request)**

> **Do NOT set colors as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "color-code the events by lane", "make the automation events blue", "highlight the checkout events in green". A newly generated workflow should leave every event on its default peach color.

When the user asks, call `update_domain_event` with the `color` parameter. Available colors: peach, yellow, green, teal, blue, lavender, pink, gray.

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", color: "blue")
```

## Constraints and Rules

- **Every domain event should have a command and an aggregate root** — No naked events. Domain event schemas are optional and only expected for Event Sourcing workflows (see Step 7)
- **One Command card per event** — An event can only have one command
- **One Aggregate Root card per event** — An event can only have one entity linked as aggregate root
- **One Domain Event card per event** — An event can only have one domain event schema (when one is created)
- **Read Model cards require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes cannot be deleted if they contain events** — Move or delete events first
- **Domain events require a lane** — Every event must be assigned to an actor
- **Chain events via follows** — Use "start" for root events, otherwise reference the parent via `$ref` path
- **Bounded contexts must exist before referencing them** — Create BCs before assigning entities to them

## `relatedEntity` Usage Summary

| Context                               | Use `relatedEntity`?                          | Example                                     |
| ------------------------------------- | --------------------------------------------- | ------------------------------------------- |
| Command: primitive type               | **NO** — use flat primitive field             | `bookingId: "bk-001"`                       |
| Command: nested structure             | **YES** — multiple fields needed              | `orderItems: [{ productName, qty, price }]` |
| Read Model: nested structure          | **YES** — nested joined data                  | `guest: { firstName, lastName, email }`     |
| Read Model: filter parameter          | **NO** — use flat field with `isFilter: true` | `checkInDate` (isFilter)                    |
| Domain Event: nested structure        | **YES** — nested event payload data           | `orderItems: [{ productId, qty, price }]`   |
| Domain Event: ID reference            | **NO** — use flat field                       | `orderId`, `customerId`                     |
| Entity: related entity                | **YES** — defines data model links            | `order → OrderItem (one-to-many)`           |


**Naming rule:** If using `relatedEntity`, name the field as the entity (`guest`, `hotel`, `orderItems`).
When one Entity/VO type plays multiple semantic roles on the same aggregate, use role-specific field names that all point to the same `relatedEntity` — e.g., `shippingAddress` and `billingAddress` both referencing a VO named "Address".
If it's a flat ID reference, name it with "Id" suffix (`guestId`, `hotelId`). Never combine "Id" suffix
with `relatedEntity`.

**Attribute-name mirroring (strict):** Across a command, its aggregate root entity, and any read model that exposes the same data, a shared attribute MUST use the same name in all three places. The attribute name is independent of the related Entity/VO **type** name — multiple attributes may reference the same Entity/VO type under different role names (see the Address example above).

## Common Workflow Patterns

### CRUD Service

- Lanes: User, Automation
- Flow: User action event → Automation processing event per operation

### Approval Pipeline

- Lanes: Requester, Approver, Automation
- Flow: Request submitted → Review pending → Approved/Rejected gateway → Executed

### Event-Driven Saga

- Lanes: Customer, Admin, Automation
- Flow: Customer/Admin actions trigger events, Automation handles cross-service coordination via decision gateways

## Tips for Well-Structured Workflows

- **Start with events, not entities.** Map the business process flow first, then identify what data each step needs.
- **Use decision gateways sparingly.** Only for genuine branching logic, not optional steps.
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

