# Domain Event Generation Guide

## Purpose

Generating domain events is the starting point of both Event Modeling and Event Storming: brainstorming domain events and arranging them on a timeline. Identify all state-changing domain events in chronological order based on the business process description.

## What is a Domain Event?

A state-changing domain event is the result of an actor (human role or Automation) invoking a state-changing command (create, update, or delete) on an aggregate.

**Valid events** (when state change occurred):

- Order Placed, Cart Item Added, Payment Cancelled, Invoice Approved, Shipment Dispatched

**Invalid events** (no state change — do NOT include):

- Page Viewed, Form Opened, Button Clicked, Report Generated

## Event Naming

**Format**: Aggregate Name + Space + Past Tense Verb, in Title Case.

Examples:

- `Order Placed` (aggregate: Order, verb: Placed)
- `Cart Item Added` (aggregate: Cart Item, verb: Added)
- `Payment Cancelled` (aggregate: Payment, verb: Cancelled)

Rules:

- Use spaces between words
- Title Case (capitalize each word)
- Max 40 characters
- Use only alphanumeric characters and spaces — no `?`, `!`, `&`, `#`, `/`
- The aggregate is the entity being created, updated, or deleted

## Roles

For each event, specify who or what triggers it via the `lane` parameter — either a human role (Sales Rep, Customer, Courier, Hotel Staff, Admin, Warehouse Worker) or the exact word `Automation` for system-triggered steps. See "Lane Tools" in `references/tools.md` for naming rules, character limits, and matching behavior.

## Event Flow

Events are arranged chronologically — left-to-right on the timeline:

- Use `follows: "start"` for the first event(s) in the flow
- Use `follows: "#/domainEvents/PreviousEvent"` to chain subsequent events
- Use `type: "decision"` for conditional/exclusive (either-or) branching, and `conditionLabel` on branches from a gateway to label the condition
- Use `parallel: true` for concurrent (AND) branching — two or more events that follow the **same** event with no condition between them

## Chronology

Events must follow a natural business-chronological sequence:

- If events are parallel or unordered, list them in the most natural business sequence
- Make reasonable assumptions when the order is ambiguous

## Common Patterns

**Linear flow:**

```
start → Item Added to Cart → Order Placed → Payment Processed → Order Shipped
```

**Decision gateway (conditional / either-or):**

```
Order Placed → Payment Outcome (gateway)
  → Payment Confirmed (conditionLabel: "Success")
  → Payment Failed (conditionLabel: "Failed")
```

**Parallel branches (concurrent / no condition):**

Two or more events that happen after the same event with no decision between them. Give each branch the same `follows`, and set `parallel: true` on every branch after the first — otherwise a second event pointing at the same parent is inserted *between* the parent and its existing follower instead of beside it.

```
Order Placed → Inventory Reserved       (follows: "Order Placed")
Order Placed → Confirmation Email Sent   (follows: "Order Placed", parallel: true)
```

Use a decision when only one path is taken based on a condition; use parallel branches when all of them happen.

**Multiple starting points:**

```
start → Customer Registered (Customer lane)
start → Product Listed (Admin lane)
```

## Tips

- Aim for 5-15 events for a typical business process
- Each event should represent a genuine state change, not a UI interaction
- Think about the full lifecycle: creation → updates → terminal states (completion, cancellation, archival)

