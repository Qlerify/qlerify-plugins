# Read Model Generation Guide

## Purpose

For each domain event, the read model defines a query that displays the current system state relevant to the actor BEFORE invoking the command that triggers the event. It provides the background information the actor needs to make decisions and take action (trigger the command with required input data). This way of working is normal practice in Event Storming: We find the Read Model by asking what input data is needed upfront. This is unlike Event Modeling where we ask what Read Models should be updated AFTER a domain event happened. Just different perspectives on the same system.

## Field Design

### Simple Fields
Primitive values returned by the query. Tag with `isFilter: true` if the field should be a search/filter parameter on the query:
```json
[
  { "name": "status", "isFilter": true }
]
```

### Computed Fields
When a read model needs a field that is not stored on the queried entity because it is calculated at runtime — totals, counts, sums, derived values, or composed/formatted values — mark it with `computed: true`. This tells the system the field is legitimate even though no matching entity field exists. Do not use `computed` for plain entity attributes or related entity references.
```json
[
  { "name": "totalAmount", "computed": true },
  { "name": "itemCount", "computed": true }
]
```


### ID Reference Fields (cross-context)
When a query attribute is an id string pointing to an entity in another bounded context, always use `Id` as suffix and never include nested fields. Use a separate query if you need to query data from multiple bounded contexts. In the example below the user entity lives in another bounded context:
```json
[
  { "name": "userId" }
]
```

### Nested Structures In Response Data
When a query has a field that contains a nested structure of data from a different related entity or value object, other than the entity that is the starting point of the query, but in the same bounded context, set `relatedEntity` and include nested sub-fields. Name the top level field as the related entity / VO in camelCase WITHOUT an "Id" suffix. Exception: when the same Entity/VO type fills multiple semantic roles in the response, use role-specific names instead (e.g., `shippingAddress` and `billingAddress` both referencing a VO named "Address"):

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

Nesting can go up to three levels deep total: a top-level field referencing a related entity or value object (e.g., `cartItem`, `shippingAddress`), its sub-fields, and — when a sub-field itself references a related entity or collection worth displaying (e.g., `cartItem` → `adjustment`, `orderItem` → `product`) — that sub-field may in turn set `relatedEntity` and carry its own `fields` array. Do not nest deeper than three levels: third-level sub-fields may still carry the usual per-field properties (e.g. `description`, `computed`) but must not include another `fields` array (no fourth level). Set `cardinality` on every nested related-entity field: `"one-to-many"` for a collection (e.g., `cartItem` for a list of cart items), `"one-to-one"` for a single object (e.g., `shippingAddress`). When a field is only an id reference to another entity (usually in another bounded context), name it `{entityName}Id` in camelCase (e.g., `customerId`, `productId`) with no nested fields.

## Important Rules

- Include the `id` of the queried entity — name it just `id`, not `entityNameId`
- Set `isFilter: true` for query/search parameters. But keep the number of filters down to what is needed to avoid slow queries or heavy indexing.
- Set `relatedEntity` for fields with nested entities / VOs
- Set `cardinality` on the read model itself: `"one-to-one"` for get queries (single-record), `"one-to-many"` for list queries. Default to `"one-to-many"` if unclear
- Set `cardinality` on fields with nested structures: `"one-to-many"` (e.g., `cartItem` for a list of cart items), `"one-to-one"` for single objects (e.g., `shippingAddress`)
- Every query has an entity or VO as entry point. Set this via the `entity` parameter
- The queried entity is often an entity created or updated earlier in the workflow, but can also belong to an external bounded context. Prefer matching an entity already present in the workflow; if the read model clearly suggests a different entity, use that name
- Read model fields should preferably match the attributes of the queried entity / VO
- When a field legitimately can't match the entity because it is calculated at runtime (totals, counts, derived values), mark it with `computed: true` instead of forcing it onto the entity
- When a read model attribute matches a command attribute, use the same name in both places. If the command uses `lineItems`, the read model must also use `lineItems`, not the generic `items`. The read model may add fields beyond those of the command, but must not rename shared ones.
- Field names: camelCase, max 30 characters
- Usually 2-8 fields are sufficient

## The Read Model's Relationship to the Command

The read model shows the actor the data they need BEFORE executing the command:
- For a command like "Create Product" → the Product id does not exist yet, so the read model provided BEFORE the command cannot ask for that particular id, unless querying another source (e.g., an existing products list)
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
        },
        {
          "name": "adjustment",
          "relatedEntity": "#/schemas/entities/Adjustment",
          "cardinality": "one-to-one",
          "fields": [
            { "name": "amount" },
            {
              "name": "code",
              "description": "Promotion or discount code applied to this line item."
            }
          ]
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
