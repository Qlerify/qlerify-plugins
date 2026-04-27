# Command Generation Guide

This is how Qlerify internally generates command schemas. Follow these rules when creating or updating commands via MCP tools.

## Purpose

For each domain event, the command defines the set of input fields required to invoke the action on the aggregate. Think of each command as a UX input form (wireframe-style) that the actor fills in and submits.

## Core Principle: Mirror the Aggregate

The command's argument structure must always mirror the structure of the entities and VOs inside the aggregate and their attribute names. Even if the user input explicitly prescribes a flat command structure, make sure to mirror the nested structure of the aggregate.

This applies even for batch operations that update many nested items — mirror all nested levels rather than collapsing into a flat array.

## The 4 Field Categories

Every command field belongs to exactly one category:

### Category 1 — Simple Attributes (Primitive Values)
Basic input values: string, number, or boolean. No nested sub-fields.

Examples: `comment`, `quantity`, `isApproved`

Usually mapped to simple attributes of the aggregate root.

### Category 2 — References (ID Only)
Referencing an existing entity from **another** bounded context.

Rules:
- Field name MUST end with `Id`
- Type is string
- No nested sub-fields
- Represents a reference only — no mutation of the referenced entity

Examples: `userId`, `paymentId`, `productId`

### Category 3 — Referenced Entity or Value Object (same bounded context, NOT mutated)
The referenced entity lives within the same bounded context but is NOT mutated by this command.

Rules:
- Field name in camelCase, **without** Id suffix
- No nested sub-fields (absence of sub-fields signals no mutation)
- Omit Id suffix since the technical reference implementation is a later concern

Examples: `customer`, `shippingMethod`, `user`

### Category 4 — Mutated Related Entity or Value Object (Nested Structure)
Input data that mutates a related entity or value object of the aggregate root.

Rules:
- Field name MUST be the camelCase form of the related entity name (e.g., `cartItem` for "Cart Item", `shippingAddress` for "Shipping Address")
- Never use generic names like `options`, `data`, `params`, or `payload`
- MUST contain at least one nested sub-field (sub-fields signal mutation)
- Sub-fields can themselves be Category 4 with their own nested sub-fields — **up to 3 levels deep**
- A related entity with matching attributes must either already exist or will be created

Examples: `cartItem`, `shippingAddress`, `appliedPromotion`

### Nested Example (3 levels)
```json
{
  "name": "cartItem",
  "fields": [
    { "name": "productId" },
    { "name": "quantity" },
    { "name": "unitPrice" },
    {
      "name": "adjustment",
      "fields": [
        { "name": "amount" },
        { "name": "code" },
        { "name": "description" }
      ]
    }
  ]
}
```

## Important Rules

### Identifier rules
- **Always include the `id` of the aggregate root** as a command argument when that entity already exists (e.g., `UpdateProduct`, `DeleteCustomer`). The aggregate root id field must be named exactly `id`, not `entityNameId`. When first creating the aggregate root, do NOT include an `id`.
- **When updating an underlying related entity**, include the `id` of that related entity inside its own nested structure to identify which specific instance is being updated. For new (not-yet-existing) related entities, do NOT include an `id`.
- **Deep nesting id walk**: when updating an underlying nested related entity (or replacing a value object) that is two levels below the aggregate root, include the id of the aggregate root AND the id of the first-level intermediate entity. This lets the backend walk down the hierarchy and find the exact instance.
- **Value objects never have an `id`** — they are identified by their attributes, not by a unique identifier. VOs are immutable and replaced as a whole.

### Field placement
- Only place an attribute inside a nested structure if it genuinely belongs to that entity or VO (don't smuggle unrelated fields down)
- Reuse identical field names consistently across commands
- Field names should match entity attributes — avoid generic names like `options`, `data`, `params`, `payload`, or `config`

### General
- Include only the fields essential to execute the command
- The number of input fields per step varies, typically between 1 and 10
- Field names: camelCase, max 30 characters
- Do NOT include UI-specific, technical, or derived fields
- Prefer business-meaningful names aligned with domain language
- List mandatory fields in the `required` array — only fields that must have a value when the command is invoked
- When a field represents a mutated related entity/VO, the field name MUST be the camelCase form of that entity or value object name (e.g., `shippingMethod` for Shipping Method, `cartItem(s)` for Cart Item). Never use generic names like `options` or `data`. This applies even when the source uses a shorter or more generic name — e.g., if the source describes a Cart with an `items` collection of Line Items, the command field must still be `cartItems` (or `lineItems`), not `items`.

## MCP Mapping

When using `create_commands` or `update_command`:
- Category 1 fields → `{ name: "quantity" }`
- Category 2 fields → `{ name: "productId" }`
- Category 3 fields → `{ name: "customer" }` (no relatedEntity, no nested fields)
- Category 4 fields → `{ name: "cartItem", relatedEntity: "#/schemas/entities/CartItem", cardinality: "one-to-many", fields: [...] }`

The `relatedEntity` $ref is needed in MCP because there's no post-hoc name matching like the internal flow. The entity must already exist (created empty in Step 5).
