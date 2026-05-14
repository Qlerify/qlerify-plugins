# Aggregate Extraction Planning Artifact

The planning artifact is the deliverable of Phase 0. It is a single markdown file
that captures the aggregate model extracted from a codebase, in a form the user can
review and approve **before** any Qlerify state is committed.

Treat it as a transient working document. Once Phase 1+ has built the workflow in
Qlerify, the Qlerify model is the source of truth; the artifact does not need to
be maintained after that.

## File location

Write the artifact to:

```
.qlerify/aggregates/{aggregate-name-kebab-case}.md
```

Examples:

- `Cart` aggregate → `.qlerify/aggregates/cart.md`
- `Subscription Plan` → `.qlerify/aggregates/subscription-plan.md`
- `Order Fulfillment` → `.qlerify/aggregates/order-fulfillment.md`

Create `.qlerify/aggregates/` if it does not already exist. If a file with the same
name is already present, read it first — it may be a prior extraction the user wants
to refine rather than replace.

## Why this artifact exists

Reverse engineering from code involves judgment calls that are easy to get wrong
and expensive to undo once committed to Qlerify:

- Is this child a value object (set-replaced) or an entity (own lifecycle)?
- Is this method a real aggregate command, or orchestration that lives outside?
- Is this field a stored attribute, or a computed projection on a read model?
- Is this collection adjustment a separate `add…` command, or merged into `set…`?

Recording these decisions as prose, with their **rationale**, makes them reviewable.
The user can challenge a single classification ("no, `Address` should be an entity
because we have separate update commands for it") without re-doing the whole model.

## Template overview

The artifact has the following sections. **Sections scale with content** — omit
sections that don't apply for the aggregate at hand, and do not invent content to
fill empty sections.

| # | Section | Required? |
|---|---|---|
| — | Title + source pointers | yes |
| 1 | Aggregate Hierarchy | yes |
| 2 | Aggregate Root | yes |
| 3 | Value Objects on the root | only if any |
| 4 | Related Entities | only if any |
| 4.x | Value Objects nested under related entities | only if any |
| 5 | Additional related entities (e.g. shipping, credit lines) | only if any |
| 6 | (continuation of related entities as needed) | only if any |
| 7 | Commands | yes |
| 8 | Domain Events | yes |
| 9 | Read Models / Queries | yes |
| 10 | Invariants | yes |
| 11 | External References | only if any |
| 12 | Tests (Given/When/Then) | yes if tests exist in the codebase |

Use the numbering only as a guide — the actual section numbers should match the
structure of the specific aggregate.

## Section-by-section guidance

### Title and source pointers

```markdown
# {Aggregate Name} Aggregate — Standalone DDD Model

**Source:** `{path to aggregate module in the codebase}`
**Service entry point:** `{class or function name}` (`{path to file}`).

{One paragraph: what's IN this aggregate model and what's explicitly OUT (e.g.,
"The service layer in X orchestrates across A + B + C. Those workflows are outside
this aggregate. What follows is only what the Cart aggregate itself owns and
decides.")}
```

This framing paragraph is important. It tells the reviewer what was peeled away
during extraction.

### Section 1: Aggregate Hierarchy

A tree diagram showing the aggregate root, its related entities, and its value
objects. Mark each node as **(aggregate root)**, **(Related Entity)**, or
**(Value Object)**. Indicate which collections are set-replaced vs add/update/remove.

```markdown
## 1. Aggregate Hierarchy

\`\`\`
Cart (aggregate root)
├── billing_address           : Address                         (Value Object)
├── items[]                   : LineItem                        (Related Entity)
│   ├── adjustments[]         : LineItemAdjustment              (Value Object — set-replaced)
│   └── tax_lines[]           : LineItemTaxLine                 (Value Object — set-replaced)
└── credit_lines[]            : CreditLine                      (Related Entity)
\`\`\`

**Why these classifications:**
- `LineItem` has independent add/update/remove commands and its own identity in the
  business vocabulary — it's a **Related Entity**.
- Adjustments are exposed through `set…` commands that replace the full collection
  atomically. Their IDs are technical. They are **Value Objects**.
- `Address` exists at most once on the cart and is replaced wholesale by
  `SetBillingAddress` — **Value Object**, despite having a DB id.
```

The "Why these classifications" block is the most important content in the entire
artifact. State the **reason** for each non-obvious classification — lifecycle
semantics, set-replacement, identity in the business vocabulary. This is where the
user will push back if you got it wrong.

### Section 2: Aggregate Root

One paragraph describing what the aggregate represents in business terms, followed
by an attribute table.

```markdown
## 2. Aggregate Root: Cart

A shopping cart owned by a customer, priced in a single currency, scoped to a
region and a sales channel. It holds purchasable items, selected shipping methods,
and pricing artefacts until it is either abandoned or converted to an order.

### Attributes (Cart)

| Attribute | Type | Req | Default | Notes |
|---|---|---|---|---|
| id | string | yes (system-generated) | — | Prefix `cart_`. Create-only. |
| currency_code | string (ISO-4217) | yes | — | Normalised to lower-case. Create-only. |
| customer_id | string \| null | no | null | External reference → Customer aggregate. |
| email | string \| null | no | null | Email of the buyer. |
| items | LineItem[] | no | [] | Owned collection, see §4. |
| ... | | | | |
```

Note rollup/computed fields as **projections** in the read-model section, NOT here.
The attribute table is for stored state only.

### Sections 3+: Value Objects and Related Entities

One subsection per VO and per related entity. Each gets:

- A one-paragraph business description
- An attribute table in the same format as the aggregate root

For VOs, explicitly note the set-replacement semantics when relevant.

### Section 7: Commands

A numbered table of the aggregate's commands. Commands are at the **aggregate
boundary** — orchestrator methods stay outside.

```markdown
## 7. Commands (Aggregate Boundary)

The Cart aggregate exposes **N commands**. Each has a 1:1 domain event.
Workflow-level orchestration (promotion evaluation, tax calculation, order
creation) is explicitly **not** an aggregate command.

| # | Command | Payload (aggregate-facing) | Notes |
|---|---|---|---|
| 1 | **CreateCart** | `currency_code`, `region_id?`, `customer_id?`, `email?`, `items?` | Atomic: may seed items. |
| 2 | **AddLineItem** | `cartId`, `items: LineItem[]` | Creates entities. |
| ... | | | |

**Commands explicitly merged or omitted:**
- `addLineItemAdjustments` — merged into `SetLineItemAdjustments`; per CLAUDE.md,
  set-replacement is the canonical pattern.
- Cart completion (create order) — NOT an aggregate command; lives in a
  cross-aggregate workflow.
- Promotion evaluation — NOT an aggregate command; lives in the promotion workflow
  which then calls `SetLineItemAdjustments`.
```

The "Commands explicitly merged or omitted" block is critical. It documents the
extraction decisions the reviewer most needs to challenge. Always include this
section when any merges or omissions were made.

### Section 8: Domain Events

A 1:1 table mapping each command to its event.

```markdown
## 8. Domain Events

| # | Event | Emitted by |
|---|---|---|
| 1 | CartCreated | CreateCart |
| 2 | LineItemAdded | AddLineItem |
| ... | | |
```

### Section 9: Read Models / Queries

The queries the aggregate exposes for clients, including computed/projected fields
that are derived on read.

```markdown
## 9. Read Models / Queries

### Query: GetCart

Inputs: `cartId`, optional relation selectors.

Returns the full Cart plus computed fields:

| Field | Description |
|---|---|
| total | Final payable amount (items + shipping + tax − discounts). |
| subtotal | item_subtotal + shipping_subtotal (pre-tax, after discounts). |
| discount_total | Sum of all adjustment amounts. |
| ... | |

### Secondary Queries

| Query | Purpose |
|---|---|
| ListCarts(filter) | Admin listing / customer cart history. |
```

When a field is clearly projection-only (totals, counts, derived flags), list it
here and **not** in the entity attribute tables. This is one of the most common
mistakes in extractions.

### Section 10: Invariants

A numbered list of business rules. Each rule should be enforceable and testable.

```markdown
## 10. Invariants

1. **Currency is fixed at creation** — `currency_code` is required on `CreateCart`
   and treated as create-only thereafter.
2. **LineItem requires `title`, `quantity`, `unit_price`** — missing any raises
   `INVALID_DATA`.
3. **Set-replacement for adjustments** — `SetLineItemAdjustments` always replaces
   the entire collection. Passing `[]` clears all adjustments. You cannot patch a
   single adjustment.
4. **Adjustments target owned children only** — adding an adjustment for a line
   item that doesn't belong to the target cart is rejected.
...
```

Invariants map to GWT acceptance criteria, command attribute rules, and entity
attribute rules in later phases. Spell them out here so they are reviewable as
prose before Qlerify gets them.

### Section 11: External References

A table of fields that point to other aggregates by ID only.

```markdown
## 11. External References

| Field | Points to |
|---|---|
| Cart.customer_id | Customer aggregate |
| Cart.region_id | Region aggregate |
| LineItem.variant_id | Product aggregate (snapshotted) |
| LineItemAdjustment.promotion_id | Promotion aggregate |

Cross-aggregate orchestration that is out of scope but exists in the codebase:
- **Cart completion** → Order aggregate (workflow in `core-flows/.../complete-cart.ts`).
- **Promotion evaluation** → Promotion aggregate computes adjustments, then calls
  `SetLineItemAdjustments`.
```

The second block — out-of-scope orchestration — is valuable even though those
flows won't be modeled. It tells the reviewer "we saw this, we decided it's not
part of this aggregate."

### Section 12: Tests

Extracted from the codebase's aggregate-level tests, rephrased in business
language as Given/When/Then. Group by command.

```markdown
## 12. Tests (at the aggregate boundary)

Extracted from `{path to integration tests}`. Rephrased in business language.

### CreateCart
- Given no cart exists, When the caller creates a cart with currency code EUR,
  Then a cart is returned with an assigned id and currency code EUR.
- Given no cart exists, When the caller creates a cart without a currency code,
  Then the command is rejected with a required-field error on currency_code.

### AddLineItem
- Given a cart exists, When the caller adds a line item with quantity 1 and unit
  price 100, Then the cart contains exactly one line item with those values.
- Given a cart id that does not exist, When the caller adds a line item, Then
  the command is rejected with a not-found error.
```

These map directly to `acceptanceCriteria` on events in Phase 2 Step 2. If the
codebase has no aggregate-level tests, write the section as a single line saying
so — do not invent tests.

## Quality checks before requesting user approval

Before presenting the artifact to the user, run through this checklist:

- [ ] Every classification (entity / VO) has a stated reason
- [ ] "Commands explicitly merged or omitted" section exists if any merges happened
- [ ] No computed/projection-only fields appear in entity attribute tables
- [ ] External references list is exhaustive — every ID-typed field points somewhere
- [ ] Out-of-scope orchestration is named, even if not modeled
- [ ] Invariants are stated as enforceable rules, not vague principles
- [ ] Tests are in Given/When/Then form, grouped by command
- [ ] The framing paragraph at the top names what was peeled away from the service layer

## After approval

Once the user approves the artifact, follow the mapping table in SKILL.md Phase 0
to translate each section into Phase 1+ tool calls. The artifact does not need to
be updated as Qlerify is built — Qlerify becomes the source of truth at that
point. Phase 4 Step 10 (post-hoc reconciliation against source code) still runs
as a safety net.
