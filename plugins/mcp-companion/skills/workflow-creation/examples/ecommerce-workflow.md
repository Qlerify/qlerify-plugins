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

```
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Customer")
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Order Service")
create_lane(workflowId: "wf-1", projectId: "proj-1", name: "Payment Gateway")
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
  description: "Payment Successful?",
  type: "bpmn:ExclusiveGateway",
  lane: "Payment Gateway",
  follows: "#/domainEvents/OrderPlaced"
)
-> { $ref: "#/domainEvents/PaymentSuccessful" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Payment Confirmed",
  type: "bpmn:Task",
  lane: "Payment Gateway",
  follows: "#/domainEvents/PaymentSuccessful",
  color: "green"
)
-> { $ref: "#/domainEvents/PaymentConfirmed" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Payment Failed",
  type: "bpmn:Task",
  lane: "Payment Gateway",
  follows: "#/domainEvents/PaymentSuccessful",
  color: "pink"
)
-> { $ref: "#/domainEvents/PaymentFailed" }

create_domain_event(
  workflowId: "wf-1", projectId: "proj-1",
  description: "Order Shipped",
  type: "bpmn:Task",
  lane: "Order Service",
  follows: "#/domainEvents/PaymentConfirmed",
  group: "Fulfillment",
  color: "peach"
)
-> { $ref: "#/domainEvents/OrderShipped" }
```

---

## Phase 3: Domain Model

### Step 5 — Create entities

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

### Step 6 — Create commands on events

Each command is attached to an event via `domainEvent`. This auto-creates the Command card.

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

### Step 7 — Create read models on events

Each read model is attached to an event via `domainEvent`. This auto-creates the Read Model card.

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

## Phase 4: Organization

### Step 8 — Link aggregate roots and create bounded context

Link entities as aggregate roots to events, and ensure bounded context exists.

```
# Link aggregate roots to events
update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/ItemAddedToCart", aggregateRoot: "#/schemas/entities/OrderItem")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/OrderPlaced", aggregateRoot: "#/schemas/entities/Order")

update_domain_event(workflowId: "wf-1", projectId: "proj-1",
  domainEvent: "#/domainEvents/OrderShipped", aggregateRoot: "#/schemas/entities/Order")

# Bounded context was already created via boundedContext parameter on create_entity.
# If not, create it explicitly:
create_bounded_context(workflowId: "wf-1", projectId: "proj-1", name: "Order Management")
```

---

## Result

The workflow now has:

- 3 lanes (Customer, Order Service, Payment Gateway)
- 3 groups (Browse & Cart, Checkout, Fulfillment)
- 6 domain events with a decision gateway for payment
- 2 entities (Order, OrderItem) with typed fields and relationships
- 3 commands (Add Item To Cart, Place Order, Ship Order) attached to events
- 2 read models (Get Order Details, List Customer Orders) attached to events
- 3 aggregate root links on events
- 1 bounded context (Order Management)
