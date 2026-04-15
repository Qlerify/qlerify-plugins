# Domain Event Schema Generation Guide

This is how Qlerify internally generates domain event payload schemas. Follow these rules when creating or updating domain event schemas via MCP tools.

## Purpose

For each domain event, the schema defines the data payload published when the event occurs. It captures the essential facts about the state change — not the full entity state.

## Field Design

### Identifier Fields
Include aggregate/entity IDs so event consumers can correlate the event: `orderId`, `customerId`, `cartId`

### Timestamp Fields
Include when timing is business-relevant: `placedAt`, `completedAt`, `cancelledAt`

### Simple Data Fields
Primitive values that capture what happened: `totalAmount`, `failureReason`, `status`

### Nested Data (embedded event payload)
When the event carries complex data (e.g., order items), use nested fields with `relatedEntity`:

```json
{
  "name": "cartItem",
  "relatedEntity": "#/schemas/entities/CartItem",
  "cardinality": "one-to-many",
  "fields": [
    { "name": "productId" },
    { "name": "quantity" },
    { "name": "unitPrice" }
  ]
}
```

## Important Rules

- Fields represent the data payload published when the event fires
- Field names: camelCase, max 30 characters
- Include fields that capture the essential facts about what happened — NOT the full entity state
- Include an aggregate identifier field (e.g., `orderId`, `cartId`) so consumers can correlate
- Include a timestamp field when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- When the event carries nested data (e.g., order items, line items), use nested fields with sub-fields
- Link the schema to the aggregate root entity via the `entity` parameter — this is the entity that produced the event
- Reuse field names consistently with command and entity field names
- Usually 3-8 fields are sufficient — keep it focused
- The `entity` parameter should preferably match an entity from the workflow specification

## Examples

**Simple event:**
```json
{
  "name": "Order Placed",
  "entity": "#/schemas/entities/Order",
  "fields": [
    { "name": "orderId" },
    { "name": "customerId" },
    { "name": "totalAmount" },
    { "name": "placedAt" }
  ]
}
```

**Event with nested data:**
```json
{
  "name": "Item Added to Cart",
  "entity": "#/schemas/entities/Cart",
  "fields": [
    { "name": "cartId" },
    {
      "name": "cartItem",
      "relatedEntity": "#/schemas/entities/CartItem",
      "fields": [
        { "name": "productId" },
        { "name": "quantity" },
        { "name": "unitPrice" }
      ]
    },
    { "name": "addedAt" }
  ]
}
```
