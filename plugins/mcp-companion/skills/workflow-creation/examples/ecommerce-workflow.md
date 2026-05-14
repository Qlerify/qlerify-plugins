# Example: Simple Order Management Workflow

The creation sequence of a simple example order management workflow is described below.
All IDs and `$ref` paths below are illustrative — actual calls return real UUIDs and paths.

---

## Phase 1: Foundation

### Step 1 — Create workflow

```
create_workflow(projectId: "proj-1", name: "Simple Order Process")
-> { workflowId: "wf-1" }
```

---

## Phase 2: Event Flow

### Step 2 — Create domain events

Build the whole event flow with one `create_domain_events` bulk call. Entries are processed in
order; each `follows` references either `"start"`, the description of an earlier sibling in the
same array, or a `$ref` to an event that already exists. Lanes ("Customer", "Warehouse Worker",
"Automation") are auto-created the first time each name is referenced. Do NOT set `aggregateRoot`
yet — entities don't exist. Do NOT set `group` — groups are optional and come later.

```
create_domain_events(workflowId: "wf-1", projectId: "proj-1", domainEvents: [
  { description: "Item Added to Order", type: "domainEvent", lane: "Customer", follows: "start" },
  { description: "Order Placed", type: "domainEvent", lane: "Customer", follows: "Item Added to Order" },
  { description: "Payment Outcome", type: "decision", lane: "Automation", follows: "Order Placed" },
  { description: "Payment Confirmed", type: "domainEvent", lane: "Automation", follows: "Payment Outcome" },
  { description: "Payment Failed", type: "domainEvent", lane: "Automation", follows: "Payment Outcome" },
  { description: "Order Shipped", type: "domainEvent", lane: "Warehouse Worker", follows: "Payment Confirmed" }
])
```

After the bulk call, set the gateway branch labels with two `update_domain_event` calls:

```
update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentConfirmed", conditionLabel: "Success")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentFailed", conditionLabel: "Failed")
```

---

## Phase 3: Domain Model

### Step 3 — Create bounded context

Use just one BC since all events belong to the same part of the system, even the same aggregate.

```
create_bounded_context(workflowId: "wf-1", projectId: "proj-1", name: "Order Management")
```

### Step 4 — Create entities and link aggregate roots to events

Create empty entities and value objects with `create_entities`, batching when it makes sense. Each entity carries the `id` attribute at creation — this classifies it as an entity (vs. a value object, which omits `id`). In the same call, link each aggregate root entity to the events that it is the aggregate root for via `aggregateRootFor`. The call atomically establishes the $ref paths and links the aggregate roots to their events.

In this example we model Order Item as a **value object** (no `id`) — the line items are immutable. Model it as an **entity** (with its own `id`) when items have an independent lifecycle — for example, when each line is individually returnable.

```
create_entities(workflowId: "wf-1", entities: [
  {
    name: "Order",
    description: "Customer order aggregate root",
    boundedContext: "Order Management",
    fields: [{ name: "id", description: "Entity ID", isRequired: true }],
    aggregateRootFor: [
      "#/domainEvents/ItemAddedToOrder",
      "#/domainEvents/OrderPlaced",
      "#/domainEvents/PaymentConfirmed",
      "#/domainEvents/PaymentFailed",
      "#/domainEvents/OrderShipped"
    ]
  },
  { name: "Order Item", boundedContext: "Order Management", fields: [] }
])
-> Order is an entity ($ref under #/schemas/entities/Order)
-> Order Item is a value object ($ref under #/schemas/valueObjects/OrderItem)
-> Order is set as aggregate root for all 5 events
```

### Step 5 — Create commands on events

All commands go in a single `create_commands` bulk call. Each command binds to its own event via
`domainEvent`. Commands reference the empty entities via `relatedEntity` for nested fields. In this simple example we assume a draft order id already exists.

```
create_commands(workflowId: "wf-1", commands: [
  {
    domainEvent: "#/domainEvents/ItemAddedToOrder",
    name: "Add Item To Order",
    fields: [
      { name: "id", isRequired: true, hideInForm: true },
      { name: "orderItem", relatedEntity: "#/schemas/valueObjects/OrderItem", cardinality: "one-to-one",
        fields: [{ name: "productName" }, { name: "quantity" }, { name: "unitPrice" }] }
    ]
  },
  {
    domainEvent: "#/domainEvents/OrderPlaced",
    name: "Place Order",
    fields: [
      { name: "id", isRequired: true },
      { name: "customerId", isRequired: true }
    ]
  },
  {
    domainEvent: "#/domainEvents/PaymentConfirmed",
    name: "Confirm Payment",
    fields: [
      { name: "id", isRequired: true },
      { name: "transactionId", isRequired: true },
      { name: "paidAmount", isRequired: true }
    ]
  },
  {
    domainEvent: "#/domainEvents/PaymentFailed",
    name: "Record Payment Failure",
    fields: [
      { name: "id", isRequired: true },
      { name: "failureReason", isRequired: true }
    ]
  },
  {
    domainEvent: "#/domainEvents/OrderShipped",
    name: "Ship Order",
    fields: [
      { name: "id", isRequired: true },
      { name: "trackingNumber", isRequired: true },
      { name: "carrier", isRequired: true }
    ]
  }
])
```

Note: `customerId` and `id` (aggregate root id, in this case order id) are flat string fields — NOT `relatedEntity` references. The
caller sends a simple ID string.

### Step 6 — Create read models on events

All read models go in a single `create_read_models` bulk call. Each entry binds to its own event.
Read models use `relatedEntity` for composed response data — nested objects make sense in API responses.

```
create_read_models(workflowId: "wf-1", readModels: [
  {
    domainEvent: "#/domainEvents/OrderPlaced",
    name: "Get Order Details",
    cardinality: "one-to-one",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id", isFilter: true },
      { name: "customerId" },
      { name: "status" },
      { name: "totalAmount" },
      { name: "orderItems", relatedEntity: "#/schemas/valueObjects/OrderItem",
        fields: [{ name: "productName" }, { name: "quantity" }, { name: "unitPrice" }] },
      { name: "createdAt" }
    ]
  },
  {
    domainEvent: "#/domainEvents/OrderShipped",
    name: "List Orders To Ship",
    cardinality: "one-to-many",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "status", isFilter: true },
      { name: "id" },
      { name: "customerId" },
      { name: "totalAmount" },
      { name: "createdAt" }
    ]
  }
])
```

### Step 7 — Create domain event schemas on events

All domain event schemas go in a single `create_domain_event_schemas` bulk call. Each entry binds
to its own event. Schemas capture the essential facts about what happened — not the full entity state.

```
create_domain_event_schemas(workflowId: "wf-1", domainEventSchemas: [
  {
    domainEvent: "#/domainEvents/ItemAddedToOrder",
    name: "Item Added to Order",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id" },
      { name: "productName" },
      { name: "quantity" },
      { name: "unitPrice" },
      { name: "addedAt" }
    ]
  },
  {
    domainEvent: "#/domainEvents/OrderPlaced",
    name: "Order Placed",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id" },
      { name: "customerId" },
      { name: "totalAmount" },
      { name: "placedAt" }
    ]
  },
  {
    domainEvent: "#/domainEvents/PaymentConfirmed",
    name: "Payment Confirmed",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id" },
      { name: "transactionId" },
      { name: "paidAmount" },
      { name: "confirmedAt" }
    ]
  },
  {
    domainEvent: "#/domainEvents/PaymentFailed",
    name: "Payment Failed",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id" },
      { name: "failureReason" },
      { name: "failedAt" }
    ]
  },
  {
    domainEvent: "#/domainEvents/OrderShipped",
    name: "Order Shipped",
    entity: "#/schemas/entities/Order",
    fields: [
      { name: "id" },
      { name: "trackingNumber" },
      { name: "carrier" },
      { name: "shippedAt" }
    ]
  }
])
```

### Step 8 — Update entities with full fields

Now that all commands, read models, and domain event schemas exist, update each entity and value
object with real fields derived from the schemas that reference them. `update_entities` accepts an
array of updates and applies them atomically — here we bundle the Order entity and the Order Item
value object into one call. The Order entity created in Step 4 already has an `id` field, so we use
`addFields` to layer on the rest. The Order Item value object has no fields yet.

```
update_entities(workflowId: "wf-1", entities: [
  {
    entity: "#/schemas/entities/Order",
    addFields: [
      { name: "customerId", dataType: "string", description: "Customer who placed the order", exampleData: ["cust-10", "cust-22", "cust-07"]},
      { name: "status", dataType: "string", description: "Current fulfillment status of the order", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
      { name: "totalAmount", dataType: "number", description: "Total amount including items, taxes, and shipping", exampleData: ["59.99", "124.50", "9.99"]},
      { name: "paidAmount", dataType: "number", description: "Total paid amount", exampleData: ["59.99", "124.50", "9.99"]},
      { name: "orderItems", dataType: "object", description: "Line items included in the order", relatedEntity: "#/schemas/valueObjects/OrderItem", cardinality: "one-to-many", exampleData: ["Object", "Object", "Object"] },
      { name: "trackingNumber", dataType: "string", description: "Shipping tracking number assigned after shipping", exampleData: ["TRK-001", "TRK-002", "TRK-003"] },
      { name: "carrier", dataType: "string", description: "Shipping carrier handling the delivery", exampleData: ["FedEx", "UPS", "DHL"] },
      { name: "createdAt", dataType: "string", description: "Timestamp when the order was placed", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
    ]
  },
  {
    entity: "#/schemas/valueObjects/OrderItem",
    addFields: [
      { name: "productName", dataType: "string", description: "Name of the purchased product", exampleData: ["Wireless Mouse", "USB-C Cable", "Laptop Stand"], isRequired: true },
      { name: "quantity", dataType: "number", description: "Number of units ordered", exampleData: ["1", "3", "2"], isRequired: true },
      { name: "unitPrice", dataType: "number", description: "Price per unit at the time of order", exampleData: ["29.99", "8.50", "45.00"], isRequired: true }
    ]
  }
])
```

Entity fields are derived from all commands/read models that reference each entity:
- Order fields include everything from "Add Item To Order", "Place Order", "Confirm Payment", "Ship Order", etc.
- `trackingNumber` and `carrier` come from "Ship Order" command
- `orderItems` relationship comes from "Add Item To Order" and "Get Order Details"

---

## Phase 4: Validation Loop

### Step 9 — Validate and fix the domain model

Run validation and judge each issue — fix real problems, leave legitimate patterns alone.

```
validate_domain_model(workflowId: "wf-1", projectId: "proj-1")
-> { issues: [] }
```

No outstanding issues — validation complete. If issues are returned, judge each one:

- **Real problems** (fix them): command field missing from entity when it should be stored, bad relationships, denormalized data, typos

Re-run validation after fixes and repeat until every remaining issue has been consciously reviewed.

---

## Result

The workflow now has:

- 3 lanes (Customer, Warehouse Worker, Automation) auto-created via `create_domain_event`
- 5 domain events plus 1 decision gateway for payment, including condition labels
- 1 entity (Order) and 1 value object (Order Item) with typed fields, relationships, and example data
- 5 commands attached to events (every event has a command, decisions don't have commands)
- 2 read models (Get Order Details, List Orders To Ship) attached to events
- 5 domain event schemas attached to events (every event has an event payload)
- 5 aggregate root links set via `aggregateRootFor` on `create_entities`
- 1 bounded context (Order Management)
- Validated domain model with no outstanding issues
