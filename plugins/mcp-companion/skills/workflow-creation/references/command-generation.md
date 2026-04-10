# Command Generation Guide

This is how Qlerify internally generates command schemas. Follow these rules when creating or updating commands via MCP tools.

## Purpose

For each domain event, the command defines the set of input fields required to invoke the action on the aggregate. Think of each command as a UX input form (wireframe-style) that the actor fills in and submits.

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

- Use only fields essential to execute the command (usually 4-8 fields)
- Reuse identical field names consistently across commands
- Field names: camelCase, max 30 characters
- Do NOT include UI-specific, technical, or derived fields
- Prefer business-meaningful names aligned with domain language
- Field names should match entity attributes — avoid generic names like `options`, `data`, `params`
- Include an `id` field when the command targets an existing entity (e.g., Update, Delete). Name it exactly `id`, not `entityNameId`. For create commands, do NOT include `id`. For value objects, do NOT include `id`
- List mandatory fields in the `required` array — only fields that must have a value when submitted
- When a field represents a mutated entity/value object, the field name MUST be the camelCase of that entity name. Its sub-fields should match the entity's attributes, up to 3 levels deep

## MCP Mapping

When using `create_command` or `update_command`:
- Category 1 fields → `{ name: "quantity" }`
- Category 2 fields → `{ name: "productId" }`
- Category 3 fields → `{ name: "customer" }` (no relatedEntity, no nested fields)
- Category 4 fields → `{ name: "cartItem", relatedEntity: "#/schemas/entities/CartItem", cardinality: "one-to-many", fields: [...] }`

The `relatedEntity` $ref is needed in MCP because there's no post-hoc name matching like the internal flow. The entity must already exist (created empty in Step 5).
