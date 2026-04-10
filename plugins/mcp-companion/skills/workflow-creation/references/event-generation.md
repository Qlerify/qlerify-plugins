# Event Generation Guide

This is how Qlerify internally generates domain events for the BPMN diagram. Follow these principles when creating events via MCP tools.

## Purpose

This is the starting point of both Event Modeling and Event Storming: brainstorming domain events and arranging them on a timeline. Identify all state-changing domain events in chronological order based on the business process description.

## What is a Domain Event?

A state-changing domain event is the result of an actor (human role or Automation) invoking a state-changing command (create, update, or delete) on an aggregate.

**Valid events** (state change occurred):
- Order Placed, Cart Item Added, Payment Cancelled, Invoice Approved, Shipment Dispatched

**Invalid events** (no state change — do NOT include):
- Page Viewed, Form Opened, Button Clicked, Report Generated

## Event Naming

**Format**: Aggregate Name + Space + Past Tense Verb, in Title Case.

Examples:
- `Order Placed` (aggregate: Order, verb: Placed)
- `Cart Item Added` (aggregate: Cart Item, verb: Added)
- `Payment Cancelled` (aggregate: Payment, verb: Cancelled)
- `Customer Registered` (aggregate: Customer, verb: Registered)

Rules:
- Use spaces between words
- Title Case (capitalize each word)
- Max 40 characters
- Use only alphanumeric characters and spaces — no `?`, `!`, `&`, `#`, `/`
- The aggregate is the entity being created, updated, or deleted

## Roles

For each event, specify who or what triggers it:

**Human actors**: short, business-oriented role names
- Sales Rep, Customer, Courier, Hotel Staff, Admin, Warehouse Worker
- Max 18 characters

**Automated actions**: always use exactly the word `Automation`
- Sending emails, processing payments, generating invoices, checking availability → all `Automation`
- Do NOT use service names (Payment Service, Notification Service) as roles

## Event Flow

Events are arranged chronologically — left-to-right on the timeline:
- Use `follows: "start"` for the first event(s) in the flow
- Use `follows: "#/domainEvents/PreviousEvent"` to chain subsequent events
- Use `type: "bpmn:ExclusiveGateway"` for decision points (branching logic)
- Use `conditionLabel` on branches from a gateway to label the condition

## Chronology

Events must follow a natural business-chronological sequence:
- If events are parallel or unordered, list them in the most natural business sequence
- Make reasonable assumptions when the order is ambiguous

## Common Patterns

**Linear flow:**
```
start → Item Added to Cart → Order Placed → Payment Processed → Order Shipped
```

**Decision gateway:**
```
Order Placed → Payment Outcome (gateway)
  → Payment Confirmed (conditionLabel: "Success")
  → Payment Failed (conditionLabel: "Failed")
```

**Multiple starting points:**
```
start → Customer Registered (Customer lane)
start → Product Listed (Admin lane)
```

## Example Output

```json
[
  { "role": "Customer", "domainEvent": "Order Placed" },
  { "role": "Automation", "domainEvent": "Payment Processed" },
  { "role": "Warehouse Staff", "domainEvent": "Order Shipped" }
]
```

## Tips

- Aim for 5-15 events for a typical business process
- Each event should represent a genuine state change, not a UI interaction
- Use Automation for anything triggered by the system without human intervention
- Group related events by the aggregate they affect — this helps identify bounded contexts later
- Think about the full lifecycle: creation → updates → terminal states (completion, cancellation, archival)
