# Entity Generation Guide

This is how Qlerify internally generates entities and value objects. Follow these rules when updating entities with full fields (Step 10) via MCP tools.

## Purpose

Generate all entities (aggregate roots) and value objects involved in the execution of all commands. Each command operates on an aggregate root entity, which may have associated entities and value objects.

## Entities vs Value Objects

**Entities** have their own identity and lifecycle. They are referenced by ID from other entities.
- MUST have `id` as the first field
- MUST include `id` in the `required` array
- Examples: Customer, Order, Product, Cart Item

**Value Objects** are defined only by their attributes — no `id` field.
- Must NOT have an `id` field
- Derive from nested structures in commands without a lifecycle and persistent identity
- Examples: Money, Address, Adjustment

## Critical Rules

### Explicit Field Inclusion
For every command that targets an aggregate, include ALL command attributes in an entity or value object. Every command attribute needs a corresponding attribute on an entity/value object as its "destination". No command attribute should be left out or ignored.

### Merge Command Fields
When an entity is targeted by multiple commands, assemble ALL unique command attributes from all associated commands. The resulting entity attributes should be the **union** of all unique command attributes.

### No Field Filtering
Include ALL command attributes without exception. Do NOT filter or omit:
- Timestamp fields (e.g., `createdAt`, `pickedAt`, `deliveredAt`)
- URL fields (e.g., `shippingLabelUrl`)
- Signature fields (e.g., `deliverySignature`)
- Note fields (e.g., `deliveryNotes`)

### Step-by-Step Derivation
1. Identify the input data/arguments for each command
2. Determine which entity acts as the aggregate root and its associated entities/value objects
3. Extract all unique fields from these commands and include them on the appropriate entity/value object

### Handling Nested Command Fields
When a command has a field with nested sub-fields (e.g., `shippingAddress` containing street, city, postalCode, country), create a corresponding entity or value object with a matching name:
- Command field `shippingAddress` → Entity/Value Object "Shipping Address"
- Include ALL the nested sub-fields as attributes

### Aggregate Root References to Nested Structures
The aggregate root entity must reference nested structures with the **EXACT SAME ATTRIBUTE NAME** as the command field:
- Command field `shippingAddress` → aggregate root field `shippingAddress` (NOT `shippingAddressId`)
- Set `dataType: "object"` and `relatedEntity` to the referenced entity

## Field Naming Rules

- **Entity names**: human-readable format with spaces (e.g., "Payment Method", not "PaymentMethod"). Max 30 characters.
- **Field names**: camelCase (e.g., `paymentMethod`, not "Payment Method"). Max 30 characters.
- **Field dataType**: one of `string`, `number`, `boolean`, `object`

## Deriving Entities from Database Tables

When the source material includes database table definitions (rather than commands), model the entities as a **domain/type model**, not a persistence model:

- Describe relationships through **ownership and references**, not storage mechanics
- Express ownership from the parent type (e.g., `Order.items: LineItem[]`)
- Omit internal back-reference fields like `parent_id` / foreign key columns unless they're domain-significant
- Avoid database terms like foreign keys, join tables, cascade deletes — use domain type relationships instead

## Relationship Rules

### Same bounded context (relatedEntity)
Name the field as the entity in camelCase (singular or plural):
```json
[
  { "name": "orderItems", "dataType": "object", "relatedEntity": "#/schemas/entities/OrderItem", "cardinality": "one-to-many" },
  { "name": "shippingAddress", "dataType": "object", "relatedEntity": "#/schemas/entities/Address", "cardinality": "one-to-one" }
]
```

### Cross-context reference (flat ID)
Name the field as `{entityName}Id` with `dataType: "string"`:
```json
[
  { "name": "customerId", "dataType": "string" },
  { "name": "productId", "dataType": "string" }
]
```

## Required Fields

Mark fields essential for the entity to exist in a **valid initial state** in the `required` array. Required means the field must have a value from the moment the entity is first created.

Fields populated only during specific lifecycle transitions (expire, cancel, archive, complete) should NOT be required, even if required in those commands. Example: `expiryReason` is required on an "Expire Cart" command but optional on the Cart entity because it doesn't exist at creation time.

## Example Data

**Every field MUST include `exampleData`** with 3 realistic example values. This applies to ALL fields, including fields with `relatedEntity`.

For fields with `dataType: "object"` and `relatedEntity` (nested entity/value object references), use the placeholder `["Object", "Object", "Object"]`. This is the Qlerify convention — it signals that the field is a reference to another entity whose actual data is defined on that entity itself.

```json
{ "name": "orderItems", "dataType": "object", "relatedEntity": "#/schemas/entities/OrderItem", "cardinality": "one-to-many", "exampleData": ["Object", "Object", "Object"] }
```

## Descriptions (Required)

Both entities/value objects AND their fields must carry `description` properties. Qlerify relies on these for stakeholder communication and downstream code generation.

### Top-level entity/VO description

Every entity or value object must include a **top-level `description`**: one or two concise sentences explaining its role in the domain.

- For **aggregate roots**: explain what the aggregate represents and its lifecycle responsibility (e.g., "Aggregate root that represents a customer purchase and tracks its fulfilment lifecycle from placement to delivery.")
- For **value objects**: explain its semantics and note that it's replaced as a whole (e.g., "Value object describing a physical postal location; immutable and replaced as a whole.")
- For **related entities** (non-root entities within an aggregate): explain what they represent and how they relate to the aggregate root

### Field description

Every field must include a `description`: a single concise sentence (ideally under 120 characters) describing what the attribute represents in the domain.

- Do NOT restate the field name — explain meaning, purpose, or role
- For `id` fields: describe what the entity identifier references (e.g., "Unique identifier of the order")
- For relationship fields: describe the nature of the relationship (e.g., "Line items included in this order")
- Use business language, not technical jargon

```json
{ "name": "customerId", "dataType": "string", "description": "Customer who placed the order", "exampleData": ["cust-10", "cust-22", "cust-07"], "isRequired": true }
```

## Entity Example

```json
{
  "name": "Order",
  "description": "Aggregate root that represents a customer purchase and tracks its fulfilment lifecycle from placement to delivery.",
  "boundedContext": "Order Management",
  "fields": [
    { "name": "id", "dataType": "string", "description": "Unique identifier of the order", "exampleData": ["ord-001", "ord-002", "ord-003"], "isRequired": true },
    { "name": "customerId", "dataType": "string", "description": "Customer who placed the order", "exampleData": ["cust-10", "cust-22", "cust-07"], "isRequired": true },
    { "name": "status", "dataType": "string", "description": "Current fulfillment status of the order", "exampleData": ["pending", "confirmed", "shipped"], "isRequired": true },
    { "name": "totalAmount", "dataType": "number", "description": "Total amount to be paid including all items, taxes, and shipping", "exampleData": ["59.99", "124.50", "9.99"], "isRequired": true },
    { "name": "orderItems", "dataType": "object", "description": "Line items included in the order", "relatedEntity": "#/schemas/entities/OrderItem", "cardinality": "one-to-many", "exampleData": ["Object", "Object", "Object"] },
    { "name": "trackingNumber", "dataType": "string", "description": "Shipping carrier tracking number assigned after the order is shipped", "exampleData": ["TRK-001", "TRK-002", "TRK-003"] },
    { "name": "createdAt", "dataType": "string", "description": "Timestamp when the order was placed", "exampleData": ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], "isRequired": true }
  ]
}
```

## Value Object Example

```json
{
  "name": "Address",
  "description": "Value object describing a physical postal location; immutable and replaced as a whole.",
  "boundedContext": "Order Management",
  "fields": [
    { "name": "street", "dataType": "string", "description": "Street name and number", "exampleData": ["123 Maple Ave", "456 Oak St", "789 Pine Rd"], "isRequired": true },
    { "name": "city", "dataType": "string", "description": "City name", "exampleData": ["New York", "Chicago", "Austin"], "isRequired": true },
    { "name": "postalCode", "dataType": "string", "description": "Postal or ZIP code", "exampleData": ["10001", "60601", "73301"], "isRequired": true },
    { "name": "country", "dataType": "string", "description": "ISO country code", "exampleData": ["USA", "USA", "USA"], "isRequired": true }
  ]
}
```

Note: the Address value object has NO `id` field — it is defined entirely by its attributes.
