# Read Model Generation Guide

This is how Qlerify internally generates read model (query) schemas. Follow these rules when creating or updating read models via MCP tools.

## Purpose

For each domain event, the read model defines a query that displays the current system state relevant to the actor BEFORE invoking the command. It provides the background information the actor needs to make decisions, fill in the input form, and submit it.

## Field Design

### Simple Fields
Primitive values returned by the query. Tag with `isFilter: true` if the field is a search/filter parameter: `{ "name": "status", "isFilter": true }`, `{ "name": "totalAmount", "computed": true }`

### Computed Fields
When a read model needs a field that is not stored on the queried entity because it is calculated at runtime — aggregates (totals, counts, sums), derived values, or composed/formatted values — mark it with `computed: true`. This tells the system the field is legitimate even though no matching entity field exists. Do not use `computed` for plain entity attributes, filter parameters, or related entity references.
```json
[
  { "name": "totalAmount", "computed": true },
  { "name": "itemCount", "computed": true }
]
```


### ID Reference Fields (cross-context)
When referencing an entity in another bounded context, use `{entityName}Id` — no nested fields: `{ "name": "customerId", "isFilter": true }`

### Composed Response Data (nested)
When the response includes data from a related entity, use `relatedEntity` with nested sub-fields. Name the field as the entity in camelCase (not with "Id" suffix):

```json
{
  "name": "cartItem",
  "relatedEntity": "#/schemas/entities/CartItem",
  "cardinality": "one-to-many",
  "fields": [
    { "name": "productName" },
    { "name": "quantity" },
    { "name": "unitPrice" }
  ]
}
```

## Descriptions (optional)

- Provide a short read-model-level `description` (one sentence) explaining what the query returns and when it is used. Omit it when the queried entity and fields already make the intent obvious.
- Field-level `description` is optional. Add a short description ONLY on fields where the meaning, purpose, or filter/computed intent is non-obvious from the name. Omit `description` on self-explanatory fields (e.g. `id`, `name`, `email`).
- Descriptions apply at both the top-level fields and nested sub-fields.

## Nesting Depth

Nesting is limited to ONE level deep. Inside an already-nested field (e.g., `items`, `cartItems`, `shippingAddress`), sub-fields must NOT themselves contain a `fields` array. If a sub-field represents a related entity or collection (e.g., `adjustments`, `taxLines`), include it by name and `cardinality` only — do NOT specify its `fields`.

## Important Rules

- Include the `id` of the queried entity — name it just `id`, not `entityNameId`
- Set `isFilter: true` for query/search parameters. Filter fields CAN be cross-entity parameters (e.g., `checkInDate` on a Hotel search) — this is valid
- Omit `isFilter` for returned data fields
- Use `relatedEntity` for composed response data — nested objects make sense in API responses
- Set `cardinality` on the read model itself: `"one-to-one"` for single-record queries, `"one-to-many"` for list queries. Default to `"one-to-many"` if unclear
- Set `cardinality` on nested fields: `"one-to-many"` for collections (e.g., `cartItems`), `"one-to-one"` for single objects (e.g., `shippingAddress`)
- Link to the source entity via the `entity` parameter — this should be the entity that is the starting point for building the query
- The queried entity should preferably match an entity from the workflow. If the read model clearly suggests a different entity, use that name
- Read model fields should preferably match the fields of the queried entity from the specification
- When a field legitimately can't match the entity because it is calculated at runtime (totals, counts, derived values), mark it with `computed: true` instead of forcing it onto the entity
- Field names: camelCase, max 30 characters
- Usually 2-8 fields are sufficient

## The Read Model's Relationship to the Command

The read model shows the actor the data they need BEFORE executing the command:
- For "Create Product" → the Product id does not exist yet, so the read model queries something else (e.g., existing products list)
- For "Ship Order" → the read model shows the order details the warehouse worker needs to see before shipping

## Example

```json
{
  "name": "Get Cart Details",
  "description": "Full cart contents shown to the shopper before checkout.",
  "entity": "#/schemas/entities/Cart",
  "cardinality": "one-to-one",
  "fields": [
    { "name": "id" },
    {
      "name": "status",
      "isFilter": true,
      "description": "Current cart status (e.g. active, checked-out), used for filtering."
    },
    {
      "name": "cartItem",
      "relatedEntity": "#/schemas/entities/CartItem",
      "cardinality": "one-to-many",
      "fields": [
        { "name": "productName" },
        { "name": "quantity" },
        {
          "name": "unitPrice",
          "description": "Price per unit at the time the item was added."
        }
      ]
    },
    {
      "name": "totalAmount",
      "computed": true,
      "description": "Sum of line-item prices, computed at read time."
    }
  ]
}
```
