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

### Step 3 — Create groups

```
create_group(workflowId: "wf-1", projectId: "proj-1", name: "Browse & Cart")
create_group(workflowId: "wf-1", projectId: "proj-1", name: "Checkout")
create_group(workflowId: "wf-1", projectId: "proj-1", name: "Fulfillment")
```

---

## Phase 2: Event Flow

### Step 4 — Create domain events

Build the chain left-to-right. Each event references the previous one via `follows`.
Set `group` only on the first event that starts a new group.

```
create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Item Added to Cart",
  type: "bpmn:Task",
  lane: "Customer",
  follows: "start",
  group: "Browse & Cart",
  color: "blue"
)
-> { $ref: "#/domainEvents/ItemAddedToCart" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Order Placed",
  type: "bpmn:Task",
  lane: "Customer",
  follows: "#/domainEvents/ItemAddedToCart",
  group: "Checkout",
  color: "peach"
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
  follows: "#/domainEvents/PaymentOutcome",
  color: "green"
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
  follows: "#/domainEvents/PaymentOutcome",
  color: "pink"
)
-> { $ref: "#/domainEvents/PaymentFailed" }

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/PaymentFailed", conditionLabel: "Failed")

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Order Shipped",
  type: "bpmn:Task",
  lane: "Warehouse Staff",
  follows: "#/domainEvents/PaymentConfirmed",
  group: "Fulfillment",
  color: "peach"
)
-> { $ref: "#/domainEvents/OrderShipped" }
```

---

## Phase 3: Domain Model

### Step 5a — Create bounded context

Create the bounded context BEFORE entities so they can be assigned during creation.
One BC is fine for a small workflow like this.

```
create_bounded_context(workflowId: "wf-1", projectId: "proj-1", name: "Order Management")
```

### Step 5b — Create entities and link aggregate roots

Create entities first so commands and read models can reference them.

```
create_entity(
  workflowId: "wf-1",
  name: "Order",
  boundedContext: "Order Management",
  fields: [
    { name: "id", dataType: "string", exampleData: ["ord-001", "ord-002", "ord-003"], isRequired: true },
    { name: "customerId", dataType: "string", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
    { name: "status", dataType: "string", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
    { name: "totalAmount", dataType: "number", exampleData: ["59.99", "124.50", "9.99"], isRequired: true },
    { name: "orderItems", dataType: "object", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many" },
    { name: "createdAt", dataType: "string", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
  ]
)
-> { $ref: "#/schemas/entities/Order" }

create_entity(
  workflowId: "wf-1",
  name: "OrderItem",
  boundedContext: "Order Management",
  fields: [
    { name: "id", dataType: "string", exampleData: ["ci-001", "ci-002", "ci-003"], isRequired: true },
    { name: "productName", dataType: "string", exampleData: ["Wireless Mouse", "USB-C Cable", "Laptop Stand"], isRequired: true },
    { name: "quantity", dataType: "number", exampleData: ["1", "3", "2"], isRequired: true },
    { name: "unitPrice", dataType: "number", exampleData: ["29.99", "8.50", "45.00"], isRequired: true }
  ]
)
-> { $ref: "#/schemas/entities/OrderItem" }
```

Now link aggregate roots to all events:

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

### Step 6 — Create commands on events

Each command is attached to an event via `domainEvent`. This auto-creates the Command card.
Note: commands use **flat ID fields** for references, NOT `relatedEntity` — except for embedded
collections like `orderItems` where multiple fields from a related entity are needed.

```
create_command(
  workflowId: "wf-1",
  domainEvent: "#/domainEvents/ItemAddedToCart",
  name: "Add Item To Cart",
  fields: [
    { name: "orderId", isRequired: true, hideInForm: true },
    { name: "orderItems", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-one",
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

### Step 7 — Create read models on events

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

---

## Phase 4: Validation

### Step 8 — Validate the domain model

Run validation to catch field mismatches between commands/read models and their entities.

```
validate_domain_model(workflowId: "wf-1", projectId: "proj-1")
-> { issues: [] }
```

If issues are returned, fix them:

- **Command field not on entity** → Remove from command or add field to entity
- **Missing relationship** → Add `relatedEntity` field to entity
- **MINOR filter field issues** on read models are usually fine (cross-entity query params)

---

## Result

The workflow now has:

- 3 lanes (Customer, Warehouse Staff, Automation)
- 3 groups (Browse & Cart, Checkout, Fulfillment)
- 7 domain events with a decision gateway for payment, including condition labels
- 2 entities (Order, OrderItem) with typed fields and relationships
- 5 commands attached to events (every event has a command)
- 2 read models (Get Order Details, List Customer Orders) attached to events
- 5 aggregate root links (every event linked to an entity)
- 1 bounded context (Order Management)
- Validated domain model with no issues
