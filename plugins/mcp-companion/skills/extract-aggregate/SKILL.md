---
name: extract-aggregate
description: >
  This skill should be used for planning when the user asks to create a workflow or domain
  model from an existing or legacy codebase, including requests like "reverse
  engineer the codebase", "document or model a legacy application", or "build
  a workflow from this code". Recommend isolating one DDD aggregate at a time,
  using one Qlerify workflow per aggregate, and following this skill's steps
  when PLANNING the work. This is a planning/preparation skill that should usually
  run BEFORE workflow-creation or sync in reverse-engineering scenarios.
  Produces a standalone aggregate plan (root entity, related entities, value
  objects, commands, optional domain events, read models, attributes,
  invariants, external references) for review before modeling in Qlerify.
allowed-tools: Glob, Grep, Read, Bash
---

# Extract Aggregate from Codebase

Extract a single DDD aggregate from an existing or legacy codebase and describe it as a standalone aggregate, suitable
for import into Qlerify.

Use this as a **planning step first** when reverse-engineering:

1. Recommend breaking the codebase down one aggregate at a time
2. Recommend one Qlerify workflow per aggregate
3. Follow the steps in this skill for each aggregate
4. Review each extracted aggregate plan with the user before implementation
5. Then model the workflow (use the workflow-creation skill) or sync into an existing one (use the sync skill)

Use this skill proactively when the user wants workflows/models from existing code — for example:

- "Extract the Order aggregate from shop-api"
- "Model the Subscription module as a DDD aggregate"
- "Reverse engineer the Cart aggregate so I can import it into Qlerify"

If the user has not specified **which** aggregate or **which** codebase, ask before scanning. You need both:

- `{AGGREGATE_NAME}` — the aggregate to extract (e.g. `Order`, `Subscription`, `Cart`)
- `{CODEBASE_NAME}` — the repo, service, or module to extract it from

## Step 1: Isolate the aggregate from the service layer

Most business applications have two command layers:

1. **Service layer** — workflows or application services that coordinate commands across multiple aggregates.
2. **Aggregate-level commands** — the actual mutation surface of the aggregate itself.

When modeling the aggregate in isolation, peel away the service layer to expose the underlying aggregate commands.

Concretely: if the codebase has a service or orchestrator that coordinates another module and then calls a method on
the `{AGGREGATE_NAME}` module, the aggregate command is **that method call** — not the orchestration name. The
orchestration stays outside; the aggregate only knows about the data it receives.

Search for the aggregate module and its entry points:

- `src/domain/{AGGREGATE_NAME}/`, `src/modules/{AGGREGATE_NAME}/`, `**/{AGGREGATE_NAME}.ts`, `**/{AGGREGATE_NAME}.cs`, `**/{AGGREGATE_NAME}.java`
- Look for repositories, aggregate roots, domain services, and application services that call into them
- Distinguish orchestration (service layer) from mutation (aggregate layer)

## Step 2: Capture the aggregate structure

Extract the following. Keep everything scoped to the `{AGGREGATE_NAME}` boundary — cross-aggregate orchestration lives
outside.

### Aggregate root entity

The top-level entity through which all mutations enter. Identify it by:

- Ownership of children (it holds the collections)
- Being the entry point that services call into

### Related entities

Children with their **own identity** and individual lifecycle (add / update / remove).

### Value objects

Children that are **replaced wholesale** (set-replacement semantics), with no independent lifecycle. They may still
have technical IDs in the implementation — treat them as VOs anyway if the domain semantics are set-replacement.

### Commands

Each command represents a distinct state change of the aggregate.

- If one command always triggers another, **merge them** into a single command here.
- Find a granularity **coarser than per-attribute changes** but **finer than create/update/delete** for the whole
  aggregate.
- The right level is one where each command represents a business-meaningful action that stakeholders can reason about
  in a planning session.
- For value objects, usually only one command is needed to set the value; clearing or removal can be modeled as
  setting a blank or empty value.

### Domain events

One event per command, forming **1:1 pairs**. Aim for **8–20 events** per aggregate (not a strict rule).

- Too many → hard for stakeholders to review on an event storming board.
- Too few → system becomes hard to reason about.

### Read models / queries

Queries needed by the client. These can contain **computed or derived fields** (totals, counts) that exist on API
responses but not on entity models.

### Attributes

**All** fields for every entity and VO:

- Name, type, required/optional, defaults, notes
- Prefer domain/type definitions over database schema
- Describe relationships in type form (e.g. `Order.items: LineItem[]`), not database form
- Omit internal back-reference fields like `parent_id` or FK fields unless they are domain-significant
- Include a short description for each entity and each attribute

### Invariants

Business rules, for example:

- Required fields, non-negative amounts
- Set-replacement semantics (e.g. "adjustments are always replaced atomically — you cannot patch a single one")
- Snapshot patterns (e.g. product data copied at add-time)
- Computed-only fields

### External references

Fields pointing to **other aggregates by ID only** (e.g. `customerId` → `Customer` in a separate bounded context).

- Do **not** model the external aggregate's internals.
- List cross-aggregate links for reference only.

## Step 3: Stay within scope

Model only the `{AGGREGATE_NAME}` aggregate boundary.

Cross-aggregate orchestration (e.g. completing one aggregate triggering creation of another, external rule evaluation
computing values passed in) lives **outside** — note that it exists, but do not model it.

Model the aggregate as a **domain/type model**, not a persistence model. Avoid database terms like foreign keys, join
tables, and cascade deletes unless they are needed to explain a business rule.

Include a minimal description of the hierarchy of entity / VO relationships.

## Step 4: Output format

Produce a single structured response with these sections. This format is optimized for import into Qlerify and for
human review in an event storming session.

```markdown
# {AGGREGATE_NAME} Aggregate

## Summary
One-paragraph description of what this aggregate represents and its role.

## Hierarchy
Tree of entity / VO relationships, e.g.:
- {AGGREGATE_NAME} (root)
  - items: LineItem[] (entity)
  - shippingAddress: Address (value object)
  - adjustments: Adjustment[] (value object, set-replaced)

## Aggregate Root: {AGGREGATE_NAME}
Short description.

### Attributes
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| ...  | ...  | ...      | ...     | ...         |

## Related Entities

### {EntityName}
Short description.

#### Attributes
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| ...  | ...  | ...      | ...     | ...         |

## Value Objects

### {ValueObjectName}
Short description + set-replacement note.

#### Attributes
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| ...  | ...  | ...      | ...     | ...         |

## Commands
| Command    | Triggered By      | Fields | Notes |
|------------|-------------------|--------|-------|
| PlaceOrder | customer checkout | ...    | ...   |

## Domain Events (1:1 with commands)
| Event       | Emitted After | Carries |
|-------------|---------------|---------|
| OrderPlaced | PlaceOrder    | ...     |

## Read Models / Queries
| Name | Returns | Computed Fields |
|------|---------|-----------------|
| ...  | ...     | ...             |

## Invariants
- ...

## External References
| Field      | Points To | Bounded Context     |
|------------|-----------|---------------------|
| customerId | Customer  | Customer Management |

## Out of Scope (noted for reference only)
- Cross-aggregate orchestration X that calls into this aggregate
- External rule engine Y that computes values passed into commands
```

## Step 5: Next steps for the user

After producing the aggregate description, treat it as a planning artifact and ask the user to review it before
implementation. Then offer these follow-up paths:

1. **Create a new Qlerify workflow from this reviewed aggregate plan** — model lanes, events, entities, commands, and
   read models (use the workflow-creation skill).
2. **Sync into an existing Qlerify workflow** — reconcile the extracted entities, commands, and read models with what
   is already in Qlerify (use the sync skill).

## Guidelines

- **Peel the service layer first.** The most common mistake is naming the orchestrator as the aggregate command.
- **Treat this as pre-work.** For reverse-engineering legacy systems, extract and review one aggregate at a time before
  using workflow-creation or sync.
- **VOs vs entities** is decided by lifecycle semantics, not by whether the implementation has an ID column.
- **Merge commands that always fire together** — event storming is for business-meaningful steps, not implementation
  steps.
- **Prefer domain types over DB types.** If the code uses `varchar`, write `string`; if it uses `@relation`, write
  `Order.items: LineItem[]`.
- **Describe, don't just list.** Every entity and every attribute gets a short description — stakeholders need to
  reason about them without reading the code.
- **Keep external aggregates opaque.** A `customerId` field is enough; do not pull in `Customer` internals.
