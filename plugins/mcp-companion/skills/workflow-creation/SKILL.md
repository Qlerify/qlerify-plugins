---
name: workflow-creation
description: >-
  Create, extend, validate, and improve Qlerify workflow diagrams and domain
  models, including domain events, roles, read models, commands, policies,
  aggregates, entities, value objects, attributes, given-when-then scenarios, bounded
  contexts, and invariants. Use when the task involves event storming, event
  modeling, domain-driven design (DDD), specification-driven development
  (SDD), model-driven development (MDD), software modeling, domain modeling,
  process modeling, legacy modernization, reverse-engineering code into a
  model, generating code from a model, or any direct use of the Qlerify
  modeling tools.
allowed-tools: Read, Glob, Grep, Bash, mcp__plugin_mcp-companion_qlerify__*
---

# Workflow Creation Guide

Build well-structured Qlerify workflows by following a specific tool sequence. Skipping steps
or calling tools out of order leads to broken references and incomplete diagrams.

## Core Concepts

A Qlerify workflow is a structured (Software Design Level) Event Storming diagram combined with detailed elements and concepts from domain-driven design (DDD):

- **Lanes** — Horizontal swimlanes representing **roles** — actors that invoke commands. Either a human role (e.g., "Customer", "Hotel Staff") or the exact word "Automation" for any system-triggered step. All automated steps share the single Automation lane — do NOT create lanes for internal services (Payment Service, Notification Service, etc.); those belong in bounded contexts. See "Lane Tools" in `references/tools.md` for full naming rules.
- **Domain Events** — Always placed in sequence within lanes. Each event represents something that happened in a business process (e.g., "Order Created", "Payment Received"). Contains a verb in past tense. A domain event is the result of a role invoking a state-changing command on an aggregate. Valid: Order Created, Invoice Approved, Payment Cancelled. Invalid (no state change): Page Viewed, Form Opened. Domain Events are the starting point of the modeling process — they are usually created before entities, commands, or read models, which are all later linked or attached to events.
- **Entities** — An Entity is a domain object that is defined by its identity, not by its attributes. An entity must have an id attribute and a life cycle. Entities can have attributes and other related entities. Some entities play the role of aggregate root from the perspective of a given command. Examples: Order, Customer, Product.
- **Value Objects** — A VO is a domain object that is defined by its attributes. A VO has no identity and is immutable (replaced as a whole when updated). Examples: Money (amount + currency), Address (street + city + postal code), Date Range (start date + end date). A VO must not have an id attribute.
- **Commands** — A Command is a state-changing operation invoked on an aggregate and can trigger a domain event. Commands and domain events have a one-to-one relationship. Commands have arguments (also referred to as fields, attributes or properties) representing data that the actor (human or automation) submits. Examples: "Create Order", "Cancel Subscription", "Add Item to Cart".
- **Read Models** — A Read Model represents the queries, projections or views used to provide the actor with the information needed before invoking a command (used for data retrieval and not state change). A Read Model can have attributes, including filter/search parameters and computed fields, and can have nested related entities or value objects as attributes. Examples: "Get Order Details", "Search Available Rooms", "List Customer Orders".
- **Domain Event Schemas** — Define the structure and meaning of domain event payloads, capturing the essential facts about what happened in the domain.
- **Bounded Contexts** — Logical boundaries that group related domain objects and business rules around a specific capability. They usually map to modules, services, or deployment boundaries. Examples: Order Management, Inventory, Customer Management, Payments.
- **Groups** — Optional visual columns used only to help human viewers organize the diagram into phases or stages, such as Order Placement or Fulfillment. Groups carry no domain, ownership, or architectural meaning. Do not create groups unless the user explicitly asks for them — see Phase 5.

### How elements reference each other

Tools use `$ref` paths to link elements together. The `get_workflow` tool returns the full workflow specification
where each element has a `$ref` path:

- Domain events: `#/domainEvents/OrderPlaced`
- Entities: `#/schemas/entities/Order`
- Value Objects: `#/schemas/valueObjects/Address`
- Commands: `#/schemas/commands/CreateOrder`
- Read models: `#/schemas/queries/GetOrderDetails`
- Domain event schemas: `#/schemas/domainEvents/OrderPlaced`

Use these paths when creating commands, read models, entities, value objects, domain event schemas, or referencing entities in fields.

**Tip:** The workflow specification returned by `get_workflow` includes a `$schema` URL (e.g., `https://app.qlerify.com/schemas/domain-model/v1`). Fetch this JSON Schema to understand the full structure of the spec — field types, allowed values, and relationships between elements.

### Event naming and `$ref` path generation

Event names are converted to PascalCase `$ref` keys:

- Spaces are removed, each word capitalized: `"Order Placed"` → `OrderPlaced`
- Hyphens are removed, next letter capitalized: `"Check-in Completed"` → `CheckInCompleted`
- **Special characters (`?`, `!`, `&`, `#`, `/`) break `$ref` path resolution** — avoid them entirely

Use only alphanumeric characters and spaces in event names. If you use hyphens, call `get_workflow`
afterward to verify the actual `$ref` key before referencing it in subsequent calls.

**Note:** Decisions (`decision`) are visualised as a separate shape on the event storming board taking up the same space as other events, but don't appear in the `get_workflow` domainEvents section. How to find a decision: a domain event is preceded by a decision shape if the domain event has a conditions attribute set. The name of the shape is hidden under the if property ("conditions":[{"after":{"$ref":"#/domainEvents/CustomerCreated"},"if":"Large Customer?","is":"YES"}]). If you need to reference a decision, call `get_workflow`, take the description from the "if" property, PascalCase it (drop spaces and punctuation, capitalize each word), and use it as the $ref segment. For example, a decision described as "Is order paid?" is referenced as #/domainEvents/IsOrderPaid.


## Creation Sequence

Follow these steps in order. Each step depends on the previous one. For an existing or
legacy codebase, start at Phase 0; otherwise skip Phase 0 and start at Phase 1.

If the aggregate is a **state machine** — a state that *guards* transitions (a status enum or another
encoding; see Phase S), with cycles, multiple terminal states, or guard-forks — insert **Phase S**
(between Phase 0 and Phase 1) to map the states first. It changes how Phase 2 builds the flow and
makes Phase 5 groups structural. For an ordinary aggregate, skip Phase S entirely.

### Phase 0: Plan the aggregate model (reverse-engineering only)

**Skip this phase entirely for greenfield workflow creation.** Run it only when the
source is an existing or legacy codebase.

This phase produces a **standalone markdown planning artifact** that captures the
aggregate model derived from the code. **Phase 1+ does not start until the user has
reviewed and approved the artifact.** Reconciliation against the source code still
happens at the end via Phase 4 Step 10 as a belt-and-suspenders safety net.

The artifact is a transient working document: it makes the interpretation of the
code reviewable as prose before any Qlerify state is committed. Boundary calls
(entity vs VO, in-scope vs out, which methods are real commands) are recorded with
their rationale so the user can push back cheaply. Once Phase 1+ has built the
workflow, Qlerify becomes the source of truth; the artifact does not need to be
maintained going forward.

If the user has not specified **which** aggregate or **which** codebase, ask before
scanning. You need both:

- `{AGGREGATE_NAME}` — the aggregate to extract (e.g., `Order`, `Subscription`, `Cart`)
- `{CODEBASE_NAME}` — the repo, service, or module to extract it from

Recommend: one DDD aggregate at a time, one Qlerify workflow per aggregate.

**Step 0.1 — Isolate the aggregate from the service layer**

Most business applications have two command layers:

1. **Service layer** — workflows or application services that coordinate commands across multiple aggregates.
2. **Aggregate-level commands** — the actual mutation surface of the aggregate itself.

When modeling the aggregate in isolation, peel away the service layer to expose the underlying aggregate commands. If the codebase has a service or orchestrator that coordinates another module and then calls a method on the `{AGGREGATE_NAME}` module, the aggregate command is **that method call** — not the orchestration name. The orchestration stays outside; the aggregate only knows about the data it receives.

Search for the aggregate and its entry points:

- `src/domain/{AGGREGATE_NAME}/`, `src/modules/{AGGREGATE_NAME}/`, `**/{AGGREGATE_NAME}.ts`, `**/{AGGREGATE_NAME}.cs`, `**/{AGGREGATE_NAME}.java`
- Look for repositories, aggregate roots, domain services, and application services that call into them
- Distinguish orchestration (service layer) from mutation (aggregate layer)

**Step 0.2 — Capture the aggregate structure**

Extract the following, scoped to the `{AGGREGATE_NAME}` boundary. Cross-aggregate
orchestration lives **outside** — note that it exists, but do not model it.

- **Aggregate hierarchy** — a short tree showing the aggregate root, related entities, and VOs, with each node marked as root / entity / VO.
- **Aggregate root entity** — top-level entity through which all mutations enter. Identified by ownership of children (it holds the collections) and by being the entry point that services call into.
- **Related entities** — children with their **own identity** and individual lifecycle (add / update / remove).
- **Value objects** — children that are **replaced wholesale** (set-replacement semantics), with no independent lifecycle. They may still have technical IDs in the implementation — treat them as VOs anyway if the domain semantics are set-replacement.
- **Commands** — each represents a distinct state change of the aggregate.
  - If one command always triggers another, **merge them** into a single command.
  - Find a granularity **coarser than per-attribute changes** but **finer than create/update/delete** for the whole aggregate. The right level is one where each command represents a business-meaningful action.
  - For value objects, usually only one command is needed to set the value; clearing or removal can be modeled as setting a blank or empty value.
  - Note which fields are create-only — required on create but not available on update.
- **Domain events** — one event per command, forming **1:1 pairs**. Aim for **8–20 events** per aggregate. Too many → hard for stakeholders to review on an event storming board. Too few → system becomes hard to reason about.
- **Read models / queries** — queries needed by the client. Can contain **computed or derived fields** (totals, counts) that exist on API responses but not on entity models. When a field is clearly projection-only, prefer listing it on the read model instead of also on entity/VO attribute tables. Add a short description for any calculated field whose derivation isn't obvious from its name.
- **Attributes** — **all** fields for every entity and VO: name, type, required/optional, defaults, notes. Prefer domain/type definitions over database schema. Describe relationships in type form (e.g. `Order.items: LineItem[]`), not database form. Omit internal back-reference fields like `parent_id` or FK fields unless they are domain-significant. Capture a short description for each entity and each attribute.
- **Invariants** — business rules: required fields, non-negative amounts, set-replacement semantics, snapshot patterns, computed-only fields. Invariants will map to GTWs on commands, command attribute rules and entity attribute rules later in the process.
- **Tests** — for each aggregate command, extract tests that validate the command's behavior **at the aggregate boundary**, in business language. If only service-level tests exist, extract only the part that proves aggregate behavior; ignore external orchestration. Example: "Given no Order exists, When the caller creates an Order with a valid customer id, Then an Order is returned with an assigned id." These map to `acceptanceCriteria` on events in Phase 2 Step 2.
- **External references** — fields pointing to **other aggregates by ID only** (e.g., `customerId` → `Customer` in a separate bounded context). Do **not** model the external aggregate's internals.

**Step 0.3 — Stay within scope**

Model only the `{AGGREGATE_NAME}` aggregate boundary. Cross-aggregate orchestration (e.g., completing one aggregate triggering creation of another, external rule evaluation computing values passed in) lives **outside** — note that it exists, but do not model it.

Model the aggregate as a **domain/type model**, not a persistence model. Avoid database terms like foreign keys, join tables, and cascade deletes unless they are needed to explain a business rule.

**Step 0.4 — Write the planning artifact**

Write the extracted model to `.qlerify/aggregates/{aggregate-name-kebab-case}.md`
(create the `.qlerify/aggregates/` directory if it does not exist). See
`references/aggregate-extraction-artifact.md` for the template and section-by-section
guidance.

Sections scale with content — omit sections that don't apply (e.g., no related
entities). For `Read Models / Queries`, keep the section if the template requires
it; only omit projection-specific fields when there are no read-model projections.
Do not invent content to fill empty sections.

**Step 0.5 — User review and approval gate**

Present the artifact to the user and explicitly ask for approval before proceeding
to Phase 1. Expected types of feedback:

- Classification disputes (this should be an entity, not a VO, because…)
- Boundary disputes (this command belongs to a different aggregate)
- Missing or incorrect invariants
- Tests that should be acceptance criteria but aren't listed
- Commands that should be merged or split

Apply approved corrections to the artifact file, then ask again. Repeat until the
user explicitly approves. **Do NOT call any Qlerify MCP tools (Phase 1+) until the
user has approved the artifact.**

**Extraction guidelines**

- **Peel the service layer first.** The most common mistake is naming the orchestrator as the aggregate command.
- **VOs vs entities** is decided by lifecycle semantics, not by whether the legacy implementation has an ID column.
- **Merge commands that always fire together** — event storming is for business-meaningful steps, not implementation steps.
- **Prefer domain types over DB types.** If the code uses `varchar`, write `string`; if it uses `@relation`, write `Order.items: LineItem[]`.
- **Keep external aggregates opaque.** A `userId` field is enough; do not pull in `User` internals unless modeling a User aggregate.

**After artifact approval, the mapping into Phase 1+ is:**

| Artifact section                                  | Goes into                                                    |
|---------------------------------------------------|--------------------------------------------------------------|
| Domain events                                     | Phase 2 Step 2 (`create_domain_events`)                      |
| Aggregate root, related entities, VOs             | Phase 3 Step 4 (`create_entities`)                           |
| Commands                                          | Phase 3 Step 5 (`create_commands`)                           |
| Read models / queries                             | Phase 3 Step 6 (`create_read_models`)                        |
| Attributes, attribute-level invariants            | Phase 3 Step 8 (`update_entities`)                           |
| Tests (Given/When/Then), command-level invariants | `acceptanceCriteria` on events in Phase 2 Step 2             |
| Invariants                                        | Verified in Phase 4 (`validate_domain_model`)                |
| External references (IDs to other BCs)            | Category 2 ID-only refs in commands (Phase 3 Step 5)         |
| Bounded context (from title metadata)             | Phase 3 Step 3 (`create_bounded_context`)                    |

After completing Phase 0 and securing user approval of the artifact, proceed to Phase 1 — or to
Phase S first if the aggregate is a state machine.

### Phase S: Map the state machine (state-machine aggregates only)

**Skip this phase for ordinary aggregates.** Run it only when the aggregate is a genuine **state
machine**: a notion of state (≈4+ distinct states) whose value *guards* which commands are allowed,
**plus** at least one of — a cycle (reopen / revisit), multiple terminal states, or a guard-fork (one
command landing in different states by input). The state is most often a status enum, but it can be
lifecycle timestamps, a child entity's existence, a flag combination, or a status-history log — and
may live on a child rather than the root (see `references/state-machine-generation.md` → "Where the
state lives"). One or two of these alone is not enough; keep those as a normal linear flow.

Why it matters: for a real state machine the default linear event chain is not just thin, it is
**misleading** — `Drafted → Registered → … → Removed` reads as one sequence when those are
*alternative* lifecycle branches. This phase maps the lifecycle so Phase 2 and Phase 5 can lay it
out faithfully.

**When it runs:** after Phase 0 (reverse-engineering) using the extracted status enum and guarded
transitions, or — for greenfield — after the user has described the lifecycle. Before Phase 1. If
detection is borderline, ask the user whether to map states onto the timeline or keep a linear flow.

**Step S.1 — Write the state-transition map (gate artifact).** Produce
`.qlerify/aggregates/{aggregate-name-kebab-case}-state-machine.md`: **every** status enum value as a
state (entry / intermediate / terminal — never drop one, even if no code touches it), and a
transition table (`fromState | trigger | toState | guard | evidence`) where `evidence` is `code` /
`external` / `business`. Record the `external` / `business` transitions you know about **even when the
code is silent**. Mark cycles, guard-forks, self-loops, and cross-instance transitions, and state
plainly whether the code actually guards its transitions or is a generic setter. See
`references/state-machine-generation.md` for the template and the full layout algorithm.

**Step S.2 — Decide the altitude.** A state machine can be drawn at two altitudes: **service** (only
what this codebase implements; `external` / `business` transitions declared in GWTs, not drawn) or
**business-lifecycle** (the full lifecycle from the status enum + domain knowledge — all states drawn,
speculative edges marked as assumptions). If the code genuinely guards its transitions the two
coincide. If the code is a **passthrough setter** (status set without rules), they diverge — **ask the
user which one they want**, and warn that the code-faithful diagram will be thinner than the lifecycle
they picture.

**Step S.3 — User review and approval gate.** Present the map and the chosen altitude alongside the
Phase 0 aggregate artifact and get explicit approval. Expect pushback on fidelity ("that's not what
the code does"), scope, and which transitions are real versus assumed. Apply corrections; **do not
call any MCP tools until the map is approved.**

**How the approved map drives the build:**

- **Phase 2 (events):** build a single linear `follows` spine where **column = postcondition state**,
  one column per distinct state (don't merge unrelated states); guard-forks become `decision`s with
  `conditionLabel`; alternate entries into mid-lifecycle states are `follows: "start"` events in that
  column.
- **Phase 5 (groups):** the state columns become Qlerify **groups** — structural and required here,
  not the cosmetic default. Create them in left→right state order; a reappearing state (a cut cycle)
  gets a distinct name (`DRAFT` / `DRAFT 2`).
- **Fan-ins** (a state reached from several predecessors; a decision with several parents) are wired
  with `add_connection`.
- **Guardrail — what's drawn depends on the altitude.** At **service** altitude draw only the happy
  path + high-value branches and push delete-from-anywhere (N-to-1), `external`, `business`, and
  `cross-instance` transitions into `acceptanceCriteria` and notes. At **business-lifecycle** altitude
  draw those `external` / `business` states and transitions too, each marked as an assumption. The
  N-to-1 delete rule (one terminal column + a GWT) holds either way.

### Phase 1: Foundation

**Step 1 — Identify or create the workflow**

For an existing workflow, call `list_workflows` to find it, then `get_workflow` to understand
its current state. For a new workflow, call `create_workflow` with a descriptive name —
`create_workflow` needs a `projectId`, so if the user has no project yet (a fresh account, or
`list_workflows` returns none), call `create_project` first and use the returned `projectId`.

### Phase 2: Event Flow

**Step 2 — Create domain events** *(see `references/event-generation.md` for naming and chronology rules)*

Use `create_domain_events` to build the whole event flow in one call. Pass an array of events in
the chronological order they occur in the business process. For inserting one or two events into
an existing workflow afterwards, use the singular `create_domain_event`.

Each entry needs:

- `description` — Aggregate Name + Space + Past Tense Verb
- `lane` — The role/actor the event belongs to (e.g., "Customer", "Automation"). Auto-created on the fly the first time an unfamiliar name is referenced, so you typically don't need `create_lane` when building a workflow from scratch. Pass exactly the same name (case-sensitive) across events that share a lane.
- `follows` — `"start"` for flow entry points, the bare description of an event created earlier in this same array, or a `$ref` path to an event that already exists in the workflow
- `type` — Either `domainEvent` (regular event) or `decision` (decision diamond)
- `parallel` — Optional. Set `true` to add a concurrent branch beside existing follower(s) of the same `follows` instead of inserting between them (see **Branching: parallel vs decision** below)

**Common lane patterns:**

- E-commerce: Customer, Warehouse Worker, Automation
- Hotel Booking: Guest, Hotel Staff, Automation
- HR Onboarding: Candidate, HR Manager, Automation

Optional parameters:

- `acceptanceCriteria` — Array of Given-When-Then acceptance criteria strings. If the input contains test scenarios, behavior specs, or sentences in "Given X, When Y, Then Z" form, attach the relevant ones to each event here — don't leave them as background documentation.

Build the flow left-to-right, creating events in the order they occur in the business process. For how `follows`, `parallel`, and decisions render spatially on the canvas, see `references/layout-and-ui.md`. If the aggregate was flagged as a state machine in **Phase S**, build the flow as the state spine instead (column = postcondition state, guard-forks as decisions, alternate entries as `follows: "start"`) — see `references/state-machine-generation.md`. Do NOT set `aggregateRoot` yet — entities don't exist at this point.
Do NOT set `group` yet — groups are created in Phase 5 (required for Phase S state machines; otherwise only on explicit user request).

### Phase 3: Domain Model

**Step 3 — Create bounded contexts**

Create bounded contexts BEFORE entities, so entities can be assigned during creation. Call
`create_bounded_context` for each logical boundary.

**Bounded context design rules:**

- Ask: "Could a team realistically build and deploy this as a separate service?" If not → same BC
- Over-splitting creates unnecessary inter-service complexity (distributed transactions, API contracts, eventual consistency)
- Common pattern: start with 1 BC, split later when the domain grows and clear boundaries emerge

**Examples:**

- Hotel Booking (6 entities): 1 BC — "Hotel Booking"
- Large E-commerce Platform (20+ entities): 4-5 BCs — "Product Catalog", "Order Management", "Inventory", "Customer Management", "Payments"

**Step 4 — Generate entity and value object names, and link aggregate roots to events**

Create entities and value objects with just their name and bounded context, and in the same call link each entity to the events it is the aggregate root for. This establishes `$ref` paths so commands, read models, and domain event schemas can reference them. Classification is determined by the presence of an `id` field — entities have one, value objects do not. Setting this at creation time avoids a follow-up `update_entities` call to convert misclassified value objects.

**Aggregate roots** are linked via `aggregateRootFor` on each entity. The domain event name often hints at its aggregate root — "Order Created" indicates the Order entity. Each event can have only one aggregate root, and value objects cannot be aggregate roots. **Every event must have an aggregate root before proceeding.**

Use `create_entities` to create the schemas. It is a bulk tool: each call accepts an array of entities and value objects, creates them, and links the aggregate roots in one atomic workflow write — so prefer batching when several can be created together.

```
create_entities(workflowId: "wf-1", entities: [
  {
    name: "Order",
    description: "Aggregate root representing a customer purchase and tracking its fulfilment lifecycle from placement to delivery.",
    boundedContext: "Order Management",
    fields: [{ name: "id", description: "Unique identifier for the order", isRequired: true }],
    aggregateRootFor: ["#/domainEvents/OrderPlaced", "#/domainEvents/OrderCancelled"]
  },
  {
    name: "Order Item",
    description: "Line item within an order capturing the product, quantity, and price at time of purchase.",
    boundedContext: "Order Management",
    fields: [{ name: "id", description: "Unique identifier for the order item", isRequired: true }]
  },
  {
    name: "Address",
    description: "Value object describing a physical postal location; immutable and replaced as a whole.",
    boundedContext: "Order Management"
  }
])
-> Order, Order Item are entities ($ref under #/schemas/entities/...)
-> Address is a value object ($ref under #/schemas/valueObjects/...)
-> Order is set as aggregate root for OrderPlaced and OrderCancelled
```

Identify ALL entities and value objects the domain needs — aggregate root entities, related or associated entities, and value objects. Use DDD judgment to decide which get the `id` field: things with their own lifecycle and identity are entities; things defined purely by their attributes (Money, Address, DateRange) are value objects.

> If you later need to reassign an aggregate root for a single event, use `update_domain_event(domainEvent: "...", aggregateRoot: "...")`. For initial workflow creation, always set it via `aggregateRootFor` on `create_entities`.

**Step 5 — Create commands on events** *(see `references/command-generation.md` for detailed field rules)*

Use `create_commands` to create the commands. It is a bulk tool: each call accepts an array of commands and attaches them to their events in one atomic workflow write — so prefer batching when several commands can be created together. Commands and domain events have a one-to-one relationship — each command in the batch must target a different event. Associate a command with an event via the required `domainEvent` parameter (a `$ref` path like `#/domainEvents/OrderPlaced`).

- Name commands using action verbs and spaces (e.g., "Create Order", "Cancel Subscription")
- Use `relatedEntity` when an attribute holds related entities or value objects

For each domain event, specify the name and arguments (also referred to as fields, attributes or properties) of the command that produces the event. Think of the command attributes as information fields that the actor (human or automation) fills in and submits. Each field name must be in camelCase. Commands are always invoked on aggregates through the aggregate root. The command attributes on the first level correspond to attributes on the aggregate root. Fields can be nested up to two levels down from the aggregate root and should be nested when updating underlying related entities or value objects. The command's argument structure must always mirror the structure of the entities and VOs inside the aggregate and their attribute names. Even if the user input explicitly prescribes a flat command structure, make sure to mirror the nested structure of the aggregate. Remember that VOs always carry all their attributes and have no id field. (While mirroring the internal aggregate structure in the command might not be recommended as a best practice in DDD, we choose this tradeoff to increase transparency when modeling.) Model each command attribute according to one of the following 4 attribute categories and rules:

1. Simple Attributes (Primitive Values): Implicitly typed as string, number, or boolean. Use this for basic input values. No nested sub-fields. Examples: comment, quantity, isApproved. Usually mapped to simple attributes of the aggregate root.
2. References (ID Only): Implicitly typed as string. Use this when referencing an existing entity from another bounded context. Field name must end with Id. No nested sub-fields. Represents a reference only (no mutation of referenced entity). Examples: userId, paymentId, productId. Usually a string attribute on the aggregate root.
3. Referenced Entity or Value Object (same bounded context): Implicitly typed as object but must never contain any nested sub-fields. Can be a single reference or a collection; set `cardinality` to `"one-to-one"` for a single reference and `"one-to-many"` for a collection. Use this when the referenced entity or value object lives within the same bounded context but outside the current aggregate's consistency boundary, and is not mutated. Unlike a Category 2 reference, the structural attributes of the referenced entity or value object are known (although not included here in the command); unlike Category 4, it is not mutated. Field name must be in camelCase without Id suffix. Do not model any nested sub-fields in your reply. The absence of nested sub-fields shows that we are not mutating the related entity or value object (mutations use Category 4). Omit the Id suffix, since the technical decision of how the reference is implemented is a later concern. Examples: currency, country, shippingMethod.
4. Nested Related Entity or Value Object to Be Mutated: Implicitly typed as an object. It must always contain at least one nested sub-field. It may be a single object or a collection; set `cardinality` to `"one-to-one"` for a single object and `"one-to-many"` for a collection. Use this when creating or updating one or more nested entities or VOs inside the aggregate. This field name should be the camelCase form of the targeted nested entity or VO name. Exception: when the same Entity/VO type fills multiple semantic roles on the aggregate, name each field after the role and let multiple fields point to the same `relatedEntity` (e.g., `shippingAddress` and `billingAddress` both referencing a VO named "Address"). Never use generic or implementation-specific command argument names such as options, data, params, or payload when targeting an underlying entity or VO. The nested sub-field names must be identical to the attribute names of the targeted related entity or VO. Nested attributes may themselves be Category 4 fields, with their own second level of nested sub-fields. For example, if the aggregate has an entity/VO structure with three levels — order => orderItems => taxRate — then the command setTaxRate should mirror all levels in its argument structure: setTaxRate({ id: "order-123", orderItems: [{ id: "item-456", taxRate: { percentage: 25, countryCode: "SE" } }] }). Note that the id fields are named exactly id and that the value object has no id. Follow this pattern even for batch operations that update multiple nested items: mirror all nested levels in the command structure.

**Every event should have a command.** After creating commands, call `get_workflow` and verify
every event has one. Events without commands represent gaps in the business process.

- **Search/filter parameters do NOT belong on commands** — they belong on Read Models with `isFilter: true`

**Step 6 — Create read models on events** *(see `references/read-model-generation.md` for detailed field rules)*

Use `create_read_models` to create the read models. It is a bulk tool: each call accepts an array of read models and attaches them to their events in one atomic workflow write — so prefer batching when several can be created together. Each read model is linked to an event via the required `domainEvent` parameter. The read model represents the input data needed by the actor BEFORE triggering the Command that in turn triggers the event.

- Name with Get/List/Search prefixes and spaces (e.g., "Get Order Details", "List Customer Orders")
- Link to the source entity via `entity` ($ref path like `#/schemas/entities/Order`)

**Reuse read models across events when it makes sense.**

Multiple events in the same workflow often need the same query — for example, several events along an Order's lifecycle all need "Get Order
Details". Instead of creating a separate near-duplicate read model per event, include another entry in the `create_read_models` array with the **same `name`** on the new event. The backend recognizes the shared name, links each event to the same underlying read model, and merges any new fields you passed into it. Judge reuse case-by-case: if a new event genuinely needs the same data shape and the same queried entity as an existing one, reuse the same name; if the shape or entity differs, pick a different name and create a distinct read model.

**Step 7 — Create domain event schemas on events (optional)** *(see `references/domain-event-generation.md` for detailed field rules)*

> **Skip this step by default.** Domain event schemas describe the payload published when an event fires, which is only meaningful in **Event Sourcing** architectures where events are the source of truth. Only perform this step if:
>
> - The user explicitly asks for it (e.g., "add domain event schemas", "set up event payloads", "we need the event contracts"), OR
> - You are reverse-engineering a codebase that already uses Event Sourcing (events are persisted and replayed to rebuild state)
>
> For regular CRUD/state-based applications, leave events without schemas — the workflow is complete without them. If unsure, ask the user before creating schemas.

Use `create_domain_event_schemas` to create the schemas. It is a bulk tool: each call accepts an array of schemas and attaches them to their events in one atomic workflow write — so prefer batching when several can be created together. Each schema is linked to an event via the required `domainEvent` parameter.

- Name matching the event (e.g., "Order Created", "Payment Processed")
- Link to the aggregate root entity via `entity` ($ref path like `#/schemas/entities/Order`)
- Include fields that capture the essential facts about the state change
- Use `relatedEntity` on nested fields pointing to the empty entities created in Step 4
- Each schema targets a different event — an event can only have one domain event schema

**Domain event schema field rules (event payload):**

Domain event schemas represent what is published when the event fires. Fields should capture what
happened, not the full entity state:

- Include identifier fields (e.g., `orderId`, `customerId`) so event consumers can correlate
- Include timestamp fields when timing is business-relevant (e.g., `placedAt`, `completedAt`)
- Use nested fields with `relatedEntity` for embedded event data (e.g., order items in the event)
- Keep it focused — usually 3 to 8 fields are sufficient
- Field names should be consistent with command and entity field names

**Step 8 — Update entities with full fields** *(see `references/entity-generation.md` for detailed derivation rules)*

Now that all commands and read models exist (plus any domain event schemas, if Step 7 was performed), update each entity with its real fields. At this point you have full visibility into every schema that references each entity, so you can derive the correct set of fields.

Use `update_entities` to apply the updates. It is a bulk tool: each call accepts an array of entity updates and applies them atomically in one workflow write, so prefer batching when several entities can be updated together. Entities created in Step 4 already have an `id` field, so use `addFields` to layer on the rest.

```
update_entities(workflowId: "wf-1", entities: [
  {
    entity: "#/schemas/entities/Order",
    addFields: [
      { name: "customerId", dataType: "string", description: "Customer who placed the order", exampleData: ["cust-10", "cust-22", "cust-07"], isRequired: true },
      { name: "status", dataType: "string", description: "Current fulfillment status", exampleData: ["pending", "confirmed", "shipped"], isRequired: true },
      { name: "orderItems", dataType: "object", description: "Line items in the order", relatedEntity: "#/schemas/entities/OrderItem", cardinality: "one-to-many", exampleData: ["Object", "Object", "Object"] },
      { name: "createdAt", dataType: "string", description: "Timestamp when the order was placed", exampleData: ["2026-01-15T10:00:00Z", "2026-01-16T14:30:00Z", "2026-01-17T09:15:00Z"], isRequired: true }
    ]
  }
])
```

**Entity field design rules:**

- Include ALL unique fields from all commands that target this entity — merge fields across commands
- Include `relatedEntity` + `cardinality` for relationships to other entities
- **Always include `exampleData`** (3 realistic values per field). For fields with `relatedEntity`, use the placeholder `["Object", "Object", "Object"]`
- **Always include `description`** — a concise one-sentence explanation in business language
- Include `dataType`: `string`, `number`, `boolean`, `object`
- Mark fields essential for creation with `isRequired: true`
- Fields populated only during specific lifecycle transitions (cancel, archive, complete) should NOT be set as required fields

### Phase 4: Validation & Reconciliation Loop

**Step 9 — Validate and fix the domain model**

Run `validate_domain_model` to check for structural issues. This catches field mismatches between
commands/read models and their aggregate root entities.

**This is an iterative loop, not a one-shot check:**

1. Run `validate_domain_model`
2. For each issue (MAJOR or MINOR), **judge whether it genuinely indicates a problem** or is a legitimate domain pattern:
   - **If it's a real problem** → fix it
   - **If it's a valid domain pattern** → leave it as-is
3. Re-run `validate_domain_model`
4. **Repeat until all remaining issues are legitimate patterns you've consciously decided to keep**

**Do NOT use severity (MAJOR vs MINOR) as the sole deciding factor.** Both need judgment. Some "MAJOR" issues can be valid; some "MINOR" issues should be fixed.

**Common real problems to fix:**

- **Command field not on entity when it should be stored** → Add the missing field to the entity via `update_entities`, or remove the field from the command via `update_commands`
- **Missing entity relationship** that the business domain requires → Add `relatedEntity` field to entity via `update_entities`
- **Denormalized fields on commands** (e.g., `guestEmail` when `guestId` already exists) → Replace with flat ID ref and let the service look up related data internally
- **Typos or inconsistent naming** between command/read model and entity fields → Rename for consistency

**Important:** Do not consider the workflow complete until every remaining issue has been consciously reviewed and either fixed or judged as a legitimate pattern.

**Step 10 — Reconcile workflow against source code (reverse-engineering only)**

> **Skip this step entirely for greenfield workflow creation.** It applies only when Phase 0 ran — when the workflow was extracted from an existing codebase and there is source code to compare the final model against.

Step 9 only checks the workflow against itself. For reverse-engineered workflows, the higher-risk question is whether the final model still matches the *source code* it was extracted from. Phase 0's extraction can drop commands, miss fields, misclassify an entity vs a value object, or invent events that have no code trigger. This step closes that loop.

**Step 10.1 — Pull the final model**

Call `get_workflow` to retrieve the workflow as it now exists in Qlerify after all create/update calls have landed.

**Step 10.2 — Re-scan the source for the aggregate boundary**

Re-run the searches from Phase 0 Step 0.1 — the aggregate module, its repository, its entry points, and the tests at the aggregate boundary. Do not trust the earlier extraction; re-derive the truth from the code.

**Step 10.3 — Build a reconciliation diff**

For each category below, list what's in the workflow vs what's in the code and flag discrepancies:

| Category                    | In workflow but not in code                                | In code but not in workflow                                            |
|-----------------------------|------------------------------------------------------------|------------------------------------------------------------------------|
| Commands                    | Phantom command — models a mutation that does not exist    | Missing command — code has a state-changing method that's not modeled  |
| Domain events               | Phantom event — no code path produces it                   | Missing event — code emits/persists something not modeled              |
| Entity fields               | Phantom field on entity                                    | Missing field — code has a domain attribute not on the entity          |
| Entity vs VO classification | Modeled as entity, code says VO (no id, set-replacement)   | Modeled as VO, code says entity (own id, own lifecycle)                |
| Cardinality                 | one-to-one vs one-to-many mismatch                         | —                                                                      |
| External refs               | Workflow models internals of an external aggregate         | —                                                                      |
| Acceptance criteria         | Event has criteria with no matching test                   | Test at the aggregate boundary with no Given/When/Then on its event    |

**Step 10.4 — Report and ask the user**

Present the diff as a punch list grouped by category. For each item, state which side looks correct and recommend one of: *fix the workflow*, *the code is wrong*, or *accept as a deliberate omission*. Do NOT auto-apply fixes — codebase reconciliation has high false-positive risk, since some discrepancies are intentional (a command deliberately merged with another, a field that's implementation-only, an internal helper that should not be modeled). The user is the only one who can adjudicate.

**Step 10.5 — Apply approved fixes**

For each item the user approves:

- Phantom/missing command → `create_commands` / `update_commands` / `delete_command`
- Phantom/missing event → `create_domain_event` / `delete_domain_event`
- Field changes on entities → `update_entities` (`addFields`, `updateFields`, `removeFields`)
- Misclassified entity/VO → recreate via `create_entities` with the corrected `id` field presence (entities have `id`; VOs do not)
- Cardinality fix → `update_entities` with corrected `cardinality`
- Missing acceptance criteria → `update_domain_event` with the Given/When/Then derived from the test

After applying fixes, **re-run Step 9** (`validate_domain_model`) to confirm the structural integrity wasn't broken by the reconciliation edits. The two loops can interact — fixing a missing field can resolve a Step 9 issue, and vice versa.

**Phase 4 stop condition**

Phase 4 is complete when:
1. Step 9 reports no unreviewed structural issues, AND
2. Step 10 reports no unreviewed discrepancies between workflow and source code (or N/A for greenfield).

### Phase 5: Polish (optional)

These steps are cosmetic and can be skipped if not needed.

**Step 11 — Create groups (optional, only on explicit user request)**

> **Do NOT create groups as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "group the events into phases", "add groups for the checkout/fulfillment stages", "organize events into stages". A newly generated workflow should have zero groups by default.
>
> **Exception — state-machine workflows (Phase S):** when the aggregate was mapped as a state machine, the state columns **are** the groups and creating them is **required**, not cosmetic. Create one group per state in left→right state order; reappearing states (cut cycles) get distinct names (`DRAFT` / `DRAFT 2`). See `references/state-machine-generation.md`.

Groups split the workflow into phases seen on the diagram with labels spread out horizontally from left to right at the top of the diagram and vertical dividers splitting the flow into phases. When the user asks for them:

1. Call `create_group` for each phase, in the order they appear on the timeline.
2. Assign events to groups using `update_domain_event` with the `group` parameter. Only set `group` on the **first event** that starts a new group — subsequent events in the same group inherit it automatically based on their position on the timeline.

Common phase patterns (use these as inspiration only when the user asks):

- E-commerce: "Browse & Cart", "Checkout", "Fulfillment"
- Onboarding: "Registration", "Verification", "Setup", "Activation"

**Step 12 — Set colors (optional, only on explicit user request)**

> **Do NOT set colors as part of a default workflow generation.** Skip this step entirely unless the user explicitly asks for them — e.g., "color-code the events by lane", "make the automation events blue", "highlight the checkout events in green". A newly generated workflow should leave every event on its default peach color.

When the user asks, call `update_domain_event` with the `color` parameter. Available colors: peach, yellow, green, teal, blue, lavender, pink, gray.

```
update_domain_event(domainEvent: "#/domainEvents/OrderPlaced", color: "blue")
```

## Constraints and Rules

- **Every domain event should have a command and an aggregate root** — No naked events. Domain event schemas are optional and only expected for Event Sourcing workflows (see Step 7)
- **One command per event** — An event can only have one command
- **One aggregate root per event** — An event can only have one entity linked as aggregate root
- **One domain event schema per event** — An event can only have one domain event schema (when one is created)
- **An event can have multiple read models** — Different read models on the same event are fine (e.g. "Get Order" + "List Orders" both on `OrderPlaced`); only attaching the same read model name twice is rejected
- **Read models require cardinality** — Always specify "one-to-one" or "one-to-many"
- **Lanes cannot be deleted if they contain events** — Move or delete events first
- **Domain events require a lane** — Every event must be assigned to an actor
- **Chain events via follows** — Use `"start"` for root events. In `create_domain_events`, `follows` may also be the bare description of an earlier event in the same batch; when referencing an already-existing event, use its `$ref` path.
- **An event can have multiple children** — To branch two events off the same non-decision parent, point them at the same `follows` and set `parallel: true` on the second (and any later) one. Without `parallel`, the later event is inserted between the parent and its existing follower(s) — reparenting all of them under it (linearized) — instead. For a `decision`, do not use `parallel`; create multiple followers with `conditionLabel` instead. See **Branching: parallel vs decision** below.
- **Bounded contexts must exist before referencing them** — Create BCs before assigning entities to them

## `relatedEntity` Usage Summary

| Context                        | Use `relatedEntity`?                          | Example                                     |
|--------------------------------|-----------------------------------------------|---------------------------------------------|
| Command: primitive type        | **NO** — use flat primitive field             | `bookingId: "bk-001"`                       |
| Command: nested structure      | **YES** — multiple fields needed              | `orderItems: [{ productName, qty, price }]` |
| Read Model: nested structure   | **YES** — nested joined data                  | `guest: { firstName, lastName, email }`     |
| Read Model: filter parameter   | **NO** — use flat field with `isFilter: true` | `checkInDate` (isFilter)                    |
| Domain Event: nested structure | **YES** — nested event payload data           | `orderItems: [{ productId, qty, price }]`   |
| Domain Event: ID reference     | **NO** — use flat field                       | `orderId`, `customerId`                     |
| Entity: related entity         | **YES** — defines data model links            | `order → OrderItem (one-to-many)`           |


**Naming rule:** If using `relatedEntity`, name the field as the entity (`guest`, `hotel`, `orderItems`).
When one Entity/VO type plays multiple semantic roles on the same aggregate, use role-specific field names that all point to the same `relatedEntity` — e.g., `shippingAddress` and `billingAddress` both referencing a VO named "Address".
If it's a flat ID reference, name it with "Id" suffix (`guestId`, `hotelId`). Never combine "Id" suffix
with `relatedEntity`.

**Attribute-name mirroring (strict):** Across a command, its aggregate root entity, and any read model that exposes the same data, a shared attribute MUST use the same name in all three places. The attribute name is independent of the related Entity/VO **type** name — multiple attributes may reference the same Entity/VO type under different role names (see the Address example above).

## Common Workflow Patterns

### CRUD Service

- Lanes: User, Automation
- Flow: User action event → Automation processing event per operation

### Approval Pipeline

- Lanes: Requester, Approver, Automation
- Flow: Request submitted → Review pending → Approved/Rejected gateway → Executed

### Event-Driven Saga

- Lanes: Customer, Admin, Automation
- Flow: Customer/Admin actions trigger events, Automation handles cross-service coordination via decision gateways

## Branching: parallel vs decision

**Default to a single linear chain.** Most timelines are told as one sequence even when steps are technically concurrent or optional — modelers linearize for readability, and so should you. Branching is the exception, not a co-equal option, and `parallel` in particular is rarely the right call. When the flow genuinely does split, pick the construct that matches the business meaning:

- **Decision (`type: "decision"`)** — conditional / exclusive (XOR). Exactly one branch is taken based on a condition. Label the branches with `conditionLabel`. Example: payment succeeds *or* fails.
- **Parallel branches (`parallel: true`)** — concurrent (AND). All branches happen, with no condition between them. Example: after "Order Placed", both "Inventory Reserved" and "Confirmation Email Sent" occur.

To build parallel branches, give each branch the same `follows` and set `parallel: true` on every branch after the first. The first branch creates the parent's initial follower; each subsequent `parallel: true` event is added beside it rather than inserted between. Omitting `parallel` linearizes the events into a single chain.

## Tips for Well-Structured Workflows

- **Start with events, not entities.** Map the business process flow first, then identify what data each step needs.
- **Default to a linear chain; use decisions and parallel branches sparingly.** Decisions are only for genuine conditional (either-or) branching; parallel branches only for events genuinely triggered at the same time by the same predecessor. Unordered or optional steps should still be sequenced, not branched (see **Branching: parallel vs decision**).
- **Name events as past-tense occurrences.** "Order Created", not "Create Order" — the command name carries the imperative.
- **Avoid special characters in event names.** Use only alphanumeric characters and spaces. No `?`, `!`, `&`, `#`.
- **Include realistic example data.** 3 values per field helps stakeholders understand the model.
- **After completing the workflow, use the `/download` skill** to save the specification to a file. This is much faster than fetching via MCP tools for large workflows.

## Additional Resources

### Reference Files

Consult these for detailed rules when creating specific element types:

- **`references/system-prompt.md`** — DDD/event modeling expert context, naming conventions, character limits, quality checks
- **`references/layout-and-ui.md`** — How the diagram is laid out (lane/event/decision/connection/group positioning) and how the rest of the Qlerify UI is organized (the seven tabs, sidebar) — read this to build well-arranged diagrams and answer questions about the UI
- **`references/aggregate-extraction-artifact.md`** — Phase 0 planning artifact: template, section-by-section guidance, file location, quality checks before user approval
- **`references/event-generation.md`** — Event naming rules, role conventions, chronology, gateway and parallel-branch patterns
- **`references/state-machine-generation.md`** — Detecting state-machine aggregates and laying the lifecycle out on the timeline (states as columns/groups, DAG cycle-cutting, guard-forks, fan-ins, completeness-in-GWTs guardrails) — used by Phase S
- **`references/command-generation.md`** — The 4 command field categories, naming rules, 3-level nesting guidelines
- **`references/read-model-generation.md`** — Read model field design, filter vs display fields, cardinality rules
- **`references/domain-event-generation.md`** — Domain event payload rules, identifier/timestamp conventions
- **`references/entity-generation.md`** — Entity field derivation from commands, merge strategy, entity vs value object rules
- **`references/tools.md`** — Complete MCP tool reference with parameters and usage tips

### Example Files

For a complete worked example showing all steps with realistic tool calls and data:

- **`examples/ecommerce-workflow.md`** — End-to-end e-commerce order workflow creation: empty entities first, then commands/read models/domain events referencing them, entity fields derived last, validation loop

