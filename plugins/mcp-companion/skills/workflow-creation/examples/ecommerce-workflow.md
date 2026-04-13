# Example: E-Commerce Order Workflow

A complete workflow creation sequence for a simple e-commerce order process.
All IDs and `$ref` paths below are illustrative — actual calls return real UUIDs and paths.

---

## Phase 1: Foundation

### Step 1 — Create workflow

```
create_workflow(projectId: "proj-1", name: "E-Commerce Order Process")
-> { workflowId: "wf-1" }
```

### Step 2 — Create lanes

Lanes are real-world actor roles or Automation — NOT internal services.

```
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Customer")
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Warehouse Staff")
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Automation")
```

---

## Phase 2: Event Flow

### Step 3 — Create domain events

Build the chain left-to-right. Each event references the previous one via `follows`.
Do NOT set `aggregateRoot` yet — entities don't exist. Do NOT set `group` — groups are optional and come later.

```
create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Item Added to Cart",
  type: "bpmn:Task",
  lane: "Customer",
  follows: "start"
)
-> { $ref: "#/domainEvents/ItemAddedToCart" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Order Placed",
  type: "bpmn:Task",
  lane: "Customer",
  follows: "#/domainEvents/ItemAddedToCart"
)
-> { $ref: "#/domainEvents/OrderPlaced" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Payment Outcome",
  type: "bpmn:ExclusiveGateway",
  lane: "Automation",
  follows: "#/domainEvents/OrderPlaced"
)
-> { $ref: "#/domainEvents/PaymentOutcome" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Payment Confirmed",
  type: "bpmn:Task",
  lane: "Automation",
  follows: "#/domainEvents/PaymentOutcome"
)
-> { $ref: "#/domainEvents/PaymentConfirmed" }

# Set condition label on gateway branch
update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentConfirmed", conditionLabel: "Success")

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Payment Failed",
  type: "bpmn:Task",
  lane: "Automation",
  follows: "#/domainEvents/PaymentOutcome"
)
-> { $ref: "#/domainEvents/PaymentFailed" }

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentFailed", conditionLabel: "Failed")

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Order Shipped",
  type: "bpmn:Task",
  lane: "Warehouse Staff",
  follows: "#/domainEvents/PaymentConfirmed"
)
-> { $ref: "#/domainEvents/OrderShipped" }
```

---

## Phase 3: Domain Model

### Step 4 — Create bounded context

One BC is fine for a small workflow like this.

```
create_bounded_context(workflowId: "wf-1", projectId: "proj-1", name: "Order Management")
```

### Step 5 — Create empty entities

Create all entities with just names — no fields yet. This establishes $ref paths for use
in commands, read models, and domain event schemas.

```
create_entity(workflowId: "wf-1", name: "Order", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/Order" }

create_entity(workflowId: "wf-1", name: "Order Item", boundedContext: "Order Management")
-> { $ref: "#/schemas/entities/OrderItem" }
```

### Step 6 — Link aggregate roots to events

Now that entities exist, link every event to its aggregate root:

```
update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/ItemAddedToCart", aggregateRoot: "#/schemas/entities/Order")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/OrderPlaced", aggregateRoot: "#/schemas/entities/Order")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentConfirmed", aggregateRoot: "#/schemas/entities/Order")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentFailed", aggregateRoot: "#/schemas/entities/Order")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/OrderShipped", aggregateRoot: "#/schemas/entities/Order")
```

### Step 7 — Create commands on events

Each command is attached to an event via `domainEvent`. This auto-creates the Command card.
Commands reference the empty entities via `relatedEntity` for nested fields.

```
create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/ItemAddedToCart",
  name: "Add Item To Cart",
  fields: [
    { name: "orderId", isRequired: true, hideInForm: true },
    { name: "orderItem", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-one",
      fields: [{ name: "productName" }, { name: "quantity" }, { name: "unitPrice" }] }
  ]
)
-> { $ref: "#/schemas/commands/AddItemToCart" }

create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderPlaced",
  name: "Place Order",
  fields: [
    { name: "customerId", isRequired: true },
    { name: "orderItems", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many",
      fields: [{ name: "productName" }, { name: "quantity" }, { name: "unitPrice" }] }
  ]
)
-> { $ref: "#/schemas/commands/PlaceOrder" }

create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/PaymentConfirmed",
  name: "Confirm Payment",
  fields: [
    { name: "orderId", isRequired: true },
    { name: "transactionId", isRequired: true },
    { name: "amount", isRequired: true }
  ]
)
-> { $ref: "#/schemas/commands/ConfirmPayment" }

create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/PaymentFailed",
  name: "Record Payment Failure",
  fields: [
    { name: "orderId", isRequired: true },
    { name: "failureReason", isRequired: true }
  ]
)
-> { $ref: "#/schemas/commands/RecordPaymentFailure" }

create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderShipped",
  name: "Ship Order",
  fields: [
    { name: "orderId", isRequired: true },
    { name: "trackingNumber", isRequired: true },
    { name: "carrier", isRequired: true }
  ]
)
-> { $ref: "#/schemas/commands/ShipOrder" }
```

Note: `customerId` and `orderId` are flat string fields — NOT `relatedEntity` references. The
caller sends a simple ID string, and the service looks up the related data internally.

### Step 8 — Create read models on events

Each read model is attached to an event via `domainEvent`. This auto-creates the Read Model card.
Read models use `relatedEntity` for composed response data — nested objects make sense in API responses.

```
create_read_model(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderPlaced",
  name: "Get Order Details",
  cardinality: "one-to-one",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId", isFilter: true },
    { name: "customerId" },
    { name: "status" },
    { name: "totalAmount" },
    { name: "orderItems", relatedEntity: "#/schemas/entities/OrderItem",
      fields: [{ name: "productName" }, { name: "quantity" }, { name: "unitPrice" }] },
    { name: "createdAt" }
  ]
)
-> { $ref: "#/schemas/queries/GetOrderDetails" }

create_read_model(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderShipped",
  name: "List Customer Orders",
  cardinality: "one-to-many",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "customerId", isFilter: true },
    { name: "status", isFilter: true },
    { name: "totalAmount" },
    { name: "createdAt" }
  ]
)
-> { $ref: "#/schemas/queries/ListCustomerOrders" }
```

### Step 9 — Create domain event schemas on events

Each domain event schema defines the payload published when the event fires. It captures the
essential facts about what happened — not the full entity state.

```
create_domain_event_schema(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/ItemAddedToCart",
  name: "Item Added to Cart",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId" },
    { name: "productName" },
    { name: "quantity" },
    { name: "unitPrice" },
    { name: "addedAt" }
  ]
)
-> { $ref: "#/schemas/domainEvents/ItemAddedToCart" }

create_domain_event_schema(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderPlaced",
  name: "Order Placed",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId" },
    { name: "customerId" },
    { name: "totalAmount" },
    { name: "placedAt" }
  ]
)
-> { $ref: "#/schemas/domainEvents/OrderPlaced" }

create_domain_event_schema(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/PaymentConfirmed",
  name: "Payment Confirmed",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId" },
    { name: "transactionId" },
    { name: "amount" },
    { name: "confirmedAt" }
  ]
)
-> { $ref: "#/schemas/domainEvents/PaymentConfirmed" }

create_domain_event_schema(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/PaymentFailed",
  name: "Payment Failed",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId" },
    { name: "failureReason" },
    { name: "failedAt" }
  ]
)
-> { $ref: "#/schemas/domainEvents/PaymentFailed" }

create_domain_event_schema(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/OrderShipped",
  name: "Order Shipped",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "orderId" },
    { name: "trackingNumber" },
    { name: "carrier" },
    { name: "shippedAt" }
  ]
)
-> { $ref: "#/schemas/domainEvents/OrderShipped" }
```

### Step 10 — Update entities with full fields

Now that all commands, read models, and domain event schemas exist, update each entity with
real fields derived from the schemas that reference them.

```
update_entity(
  workflowId: "wf-1",
  entity: "#/schemas/entities/Order",
  fields: [
    { name: "id", dataType: "string", description: "Unique identifier of the order", exampleData: ["ord-001", "ord-002", "ord-003"], isRequired: true },
    { name: "customerId", dataType: "string", description: "Customer who placed the order", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
    { name: "status", dataType: "string", description: "Current fulfillment status of the order", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
    { name: "totalAmount", dataType: "number", description: "Total amount including items, taxes, and shipping", exampleData: ["59.99", "124.50", "9.99"], isRequired: true },
    { name: "orderItems", dataType: "object", description: "Line items included in the order", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many", exampleData: ["Object", "Object", "Object"] },
    { name: "trackingNumber", dataType: "string", description: "Shipping tracking number assigned after shipping", exampleData: ["TRK-001", "TRK-002", "TRK-003"] },
    { name: "carrier", dataType: "string", description: "Shipping carrier handling the delivery", exampleData: ["FedEx", "UPS", "DHL"] },
    { name: "createdAt", dataType: "string", description: "Timestamp when the order was placed", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
  ]
)

update_entity(
  workflowId: "wf-1",
  entity: "#/schemas/entities/OrderItem",
  fields: [
    { name: "id", dataType: "string", description: "Unique identifier of the order item", exampleData: ["ci-001", "ci-002", "ci-003"], isRequired: true },
    { name: "productName", dataType: "string", description: "Name of the purchased product", exampleData: ["Wireless Mouse", "USB-C Cable", "Laptop Stand"], isRequired: true },
    { name: "quantity", dataType: "number", description: "Number of units ordered", exampleData: ["1", "3", "2"], isRequired: true },
    { name: "unitPrice", dataType: "number", description: "Price per unit at the time of order", exampleData: ["29.99", "8.50", "45.00"], isRequired: true }
  ]
)
```

Entity fields are derived from all commands/read models that reference each entity:
- Order fields include everything from "Place Order", "Confirm Payment", "Ship Order", etc.
- `trackingNumber` and `carrier` come from "Ship Order" command
- `orderItems` relationship comes from "Place Order" and "Get Order Details"

---

## Phase 4: Validation Loop

### Step 11 — Validate and fix the domain model

Run validation and judge each issue — fix real problems, leave legitimate patterns alone.

```
validate_domain_model(workflowId: "wf-1", projectId: "proj-1")
-> { issues: [] }
```

No outstanding issues — validation complete. If issues are returned, judge each one:

- **Real problems** (fix them): command field missing from entity when it should be stored, bad relationships, denormalized data, typos
- **Legitimate patterns** (leave as-is): calculated fields on read models (`total`, `subtotal`), cross-entity filter parameters, formatted/presentation fields

Re-run validation after fixes and repeat until every remaining issue has been consciously reviewed.

---

## Result

The workflow now has:

- 3 lanes (Customer, Warehouse Staff, Automation)
- 7 domain events with a decision gateway for payment, including condition labels
- 2 entities (Order, Order Item) with typed fields, relationships, and example data
- 5 commands attached to events (every event has a command)
- 2 read models (Get Order Details, List Customer Orders) attached to events
- 5 domain event schemas attached to events (every event has an event payload)
- 5 aggregate root links (every event linked to an entity)
- 1 bounded context (Order Management)
- Validated domain model with no outstanding issues
