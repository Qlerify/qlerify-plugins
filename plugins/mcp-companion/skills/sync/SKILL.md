---
name: sync
description: >-
  Keep a codebase and its Qlerify domain model in agreement, in both
  directions. Use when the user asks to "sync domain model", "update Qlerify",
  "push changes to Qlerify", "sync schemas/entities/domain events", or after
  implementing features that add or change entities, API endpoints, commands,
  domain events, database schemas, migrations, or Prisma/GraphQL types — sync
  applies that code drift into the Qlerify model. Also use when the Qlerify
  model has been edited (new entities, changed commands, new acceptance
  criteria) and the code needs to catch up — sync detects the model-side drift
  and hands off to the code-generation skill. For brownfield/legacy codebases
  with unclear boundaries, isolate one aggregate at a time first (the
  workflow-creation skill's Phase 0 covers aggregate extraction).
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, mcp__plugin_mcp-companion_qlerify__*
---

# Sync Code and Qlerify Domain Model

Reconcile a local codebase with its Qlerify workflow so the two stay in agreement. Drift happens on both sides: a developer adds a field or endpoint in code, or someone edits the domain model on the Qlerify board. This skill detects drift on either side and routes each direction to the right place.

- **Code is ahead** (code has something the model doesn't) → sync **writes it into Qlerify** using the MCP tools. This is sync's native direction.
- **Model is ahead** (the model has something the code doesn't) → sync **does not rewrite code itself**. It reports the delta and hands off to the **`code-generation`** skill, which is the model → code executor.
- **Both sides changed the same element** → sync reports the conflict and lets the user decide before touching anything.

This skill pairs with two others: `workflow-creation` builds the model (and reverse-engineers code into a model in its Phase 0); `code-generation` writes code from the model. Sync is the ongoing reconciler that keeps them aligned after both have run.

## Core principles

- **During reconciliation, the model and the code are peers — not master/replica.** This is scoped to drift detection: sync does not assume one side wins just because it is the model or the code; its job is to surface the difference, work out which side moved, and apply the safe direction (asking on anything ambiguous or destructive). This does not contradict `code-generation`, where the model *is* authoritative for producing code — that's a different action. Authority depends on what's being done: sync writes code→model drift itself, and hands model→code drift back to `code-generation`, which treats the model as the source of truth.
- **Drift direction is decided by the anchor, not by guessing.** The `modelHash` recorded at the last sync/generation is the pivot. Without it, sync can see *that* code and model differ but not *which* side moved — so it must ask rather than assume.
- **Persistence is not drift.** The model gives types, ownership, and dependencies — never storage. A VO stored as its own table, an embedded subdocument, an optimistic-locking `version` column, or an FK index is an implementation choice, not a missing/extra domain element. Read `persistenceDecisions` from the anchor and do not flag them.
- **Deletions are always user-gated.** Removing an element from one side because the other side lacks it has high false-positive risk (feature flags, work-in-progress, deliberately model-only elements). Never delete without explicit confirmation.
- **Follow the model's own rules when writing to it.** Field design, nesting depth, entity-vs-VO classification, and naming all follow the same rules the other skills use — defer to the reference files rather than improvising.

## The anchor: `.qlerify/codegen.json`

Sync is precise only when it knows the model state as of the last reconciliation. That state lives in `.qlerify/codegen.json` (written by `code-generation`, and maintained by this skill):

- `workflowId` + `workflowName` — which workflow this code corresponds to
- `modelHash` — content hash of the canonical spec at last sync/generation (the drift pivot)
- `stack`, `aggregates` — platform and progress, used by `code-generation`
- `persistenceDecisions` — non-derivable storage choices to ignore as drift
- `generatedAt` — ISO timestamp written by `code-generation`
- `syncedAt` — ISO timestamp written/updated by `sync` on each reconciliation (absent until the first sync)

**Sync maintains this anchor.** If a project has a model but no anchor (hand-written or model-first code that was never generated), sync writes one on its first successful run so every later sync becomes precise. From then on the project is "anchored" regardless of whether `code-generation` ever ran.

## Computing the model hash

`modelHash` is the drift pivot, so **sync and code-generation must compute it identically** — if they diverge, every sync falsely reports "model moved." Until an authoritative `model_hash` MCP tool exists, both skills use this exact recipe:

1. Take the `specification` object returned by `get_workflow`.
2. Remove cosmetic and illustrative fields wherever they occur, at any depth: exactly `color`, `group`, and `follows`. Stripping these means board-only edits (recolor, regroup, drag-reorder events) do not read as model drift. What remains is the domain-semantic model: entities, value objects, commands, read models, domain event schemas, bounded contexts, roles, aggregate-root links, and acceptance criteria.
3. Serialize as compact JSON with every object's keys sorted recursively and no insignificant whitespace.
4. `modelHash` is the SHA-256 hex digest of that string.

Reference implementation (bash + jq, run on the `specification` JSON):

```
jq -cS 'walk(if type == "object" then del(.color, .group, .follows) else . end)' \
  | shasum -a 256 | cut -d" " -f1
```

The scope is **domain semantics only** — two models that differ only in board cosmetics or event ordering must produce the same hash. Follow this definition exactly; a future `model_hash` MCP tool would make it authoritative and remove the hand-rolled step.

## Phase 0: Establish the baseline

1. **Find the anchor.** Look for `.qlerify/codegen.json` (and the cached spec at `.qlerify/workflow.json`).
2. **Acquire the current model.** If a large spec hasn't been downloaded yet, invoke the `download` skill (much faster than MCP for big specs); otherwise call `get_workflow` once and cache it. The field types, values, and relationships you need are already in the spec payload itself — read them from there (and from the cached `.qlerify/workflow.json`) rather than fetching the `$schema` URL.
3. **Resolve the workflow.**
    - **Anchored:** use `workflowId` from the anchor.
    - **Unanchored:** call `list_workflows`, match by project/name, or ask the user. For a brownfield codebase with unclear aggregate boundaries, isolate one aggregate at a time first — defer to `workflow-creation` Phase 0 for the extraction.
4. **Determine the mode.**
    - **Anchored:** recompute the model hash of the current spec (see [Computing the model hash](#computing-the-model-hash)) and compare to `modelHash`. If they differ, the **model moved** since last sync — that side's drift is known up front.
    - **Unanchored:** there is no prior hash. Sync can only diff *current code ↔ current model* structurally and cannot tell which side moved — so it must ask the user to adjudicate direction per conflict, then write an anchor at the end.

## Phase 1: Scan the code domain model

Read the code as a domain model, in the same DDD terms the other skills use — not as raw files.

- **Entities vs value objects** — an entity has its own identity (`id`) and lifecycle; a value object is defined by its attributes and replaced as a whole. Classify by identity/lifecycle semantics, *not* by whether the ORM gave it an id column.
- **Aggregate roots and ownership** — which type owns the collections and is the entry point that services call into. Owned children (entities and VOs) belong with their root.
- **External references** — `*Id` fields pointing at another bounded context's aggregate. These stay opaque strings; do not pull in the external aggregate's internals.
- **Commands** — state-changing operations (handlers, service methods, POST/PUT/DELETE endpoints). A command's argument structure mirrors the aggregate's nested entity/VO structure, up to three levels.
- **Read models** — queries / GET endpoints / projections. Distinguish filter parameters from returned fields.
- **Domain event schemas** — only relevant for Event Sourcing codebases (persisted/replayed events). Skip otherwise.
- **Invariants** — collect from three places: attribute constraints (required, type, min/max, enums → allowed values, regex), command-level rules, and test scenarios (which map to Given-When-Then acceptance criteria on events).

Search hints: `src/domain/`, `src/modules/`, `src/entities|models/`, command/handler/query folders, `src/routes|api|controllers/`, `prisma/schema.prisma`, `**/schema.graphql`, `**/migrations/`. Use `git diff` / `git log` to find recently changed schema files for field-level deltas.

## Phase 2: Classify the drift

Build a reconciliation diff per category (entities, value objects, commands, read models, domain event schemas, fields, bounded contexts). For each element decide which case it is:

| Case                                   | Meaning                    | Action                                                      |
|----------------------------------------|----------------------------|-------------------------------------------------------------|
| In code, not in model                  | Code is ahead              | Apply code → model (Phase 3)                                |
| In model, not in code                  | Model is ahead             | Suggest code-generation (Phase 4)                           |
| In both, differing                     | Conflict or one-sided edit | Anchored: direction from the hash. Unanchored: ask the user |
| Differs but is a `persistenceDecision` | Storage choice, not drift  | Ignore                                                      |

In **anchored** mode, the hash comparison from Phase 0 tells you whether the model moved; combine that with the code scan to attribute each difference to the correct side automatically. In **unanchored** mode, present each difference and let the user say which side is right.

**Renames are ambiguous — never auto-apply them.** Entities, commands, and fields are keyed by name, so a rename (`note` → `customerNote`) looks structurally identical to a delete of `note` plus an add of `customerNote`. Before treating a name as deleted on one side and a new name as added on the other, check whether they are actually the same element renamed: correlate against the **anchored baseline** (the element that existed at last sync) and compare shape — same type/dataType, same `relatedEntity`/`cardinality`, same position among siblings, same surrounding fields. If they plausibly match, it is a rename, not a delete+add — flag it as a conflict (or a one-sided rename) and **ask the user** rather than deleting one name and creating the other, which would drop data and history. When there is no baseline (unanchored) the correlation is weaker, so lean even harder on asking.

## Phase 3: Apply code → model changes

Write code-side drift into Qlerify with the MCP tools, following the model's own authoring rules.

- **Order matters — two passes.** Create new entities/VOs as **stubs first** (name + `id` for entities, no `id` for VOs, `boundedContext`, link `aggregateRootFor`), then create/adjust commands and read models, then **fill entity fields last** with `update_entities` — including the cross-entity `relatedEntity` references. The bulk create tools resolve `relatedEntity` refs against the workflow state at the start of the call, so a forward reference to an entity created in the same call resolves to `null`. The stubs-then-fields order avoids that.
- **Use the bulk tools** (`create_entities`, `create_commands`, `create_read_models`, `update_entities`, …) and batch related changes into one atomic write.
- **Carry the full field model** — nesting up to three levels, `cardinality` on related fields, `computed: true` on calculated read-model fields, `isFilter: true` on query parameters (at any nesting level), `isRequired` for non-nullable fields, and 3 realistic `exampleData` values per entity field.
- **Apply the field-design essentials** (enough to sync correctly on their own):
    - Entities have an `id`; value objects never do and are replaced as a whole.
    - A reference to a same-bounded-context entity/VO uses `relatedEntity` (+ `cardinality`) and is named after the entity (`shippingAddress`, `orderItems`). A reference to an external-BC aggregate is a plain string named `{entity}Id` (`customerId`) — never combine an `Id` suffix with `relatedEntity`.
    - Command argument structure mirrors the aggregate's nested entity/VO structure, up to three levels.
    - Read-model fields carry `isFilter: true` for query parameters and `computed: true` for runtime-calculated fields, at any nesting level.
    - Every entity field gets a one-line description and 3 realistic `exampleData` values (`["Object", "Object", "Object"]` for `relatedEntity` fields).
- **For fuller detail**, if the `workflow-creation` skill is installed alongside this one, consult its `references/` (entity, command, read-model, domain-event generation rules, naming/character limits, and the `relatedEntity` usage table). Don't depend on it being present — the essentials above stand alone.
- **Deletions** (code removed something the model still has) → list them and confirm with the user before calling any delete tool.

## Phase 4: Handle model → code drift

When the model has elements the code lacks (or the hash shows the model moved), sync does **not** edit code. Instead:

1. Report the model-side delta as a punch list grouped by category (new/changed entities, commands, read models, new GWTs, etc.).
2. Recommend running the **`code-generation`** skill to apply the delta as a targeted patch — it already knows how to add an aggregate or apply a model change to existing code from the anchor.
3. Leave the code untouched. Sync's responsibility here ends at detection and handoff.

## Phase 5: Validate, anchor, and report

1. Run `validate_domain_model` on the affected bounded context(s). Treat it as a judgment loop, not a one-shot: fix genuine structural problems, leave legitimate domain patterns (e.g. a read-model filter field that isn't on the entity) as-is. Re-run until every remaining issue is consciously accepted.
2. **Update the anchor.** Recompute the model hash (see [Computing the model hash](#computing-the-model-hash)) and write it back to `.qlerify/codegen.json` as `modelHash`, with a fresh `syncedAt`. Create the anchor here if the project was unanchored, recording `workflowId`, `workflowName`, and the hash so the next sync is precise. Preserve `stack`, `aggregates`, and `persistenceDecisions` if present.
3. **Report** a summary:
    - Code → model: entities/VOs/commands/read models/schemas and fields created, updated, or (with confirmation) deleted
    - Model → code: deltas detected and handed off to code-generation
    - Conflicts left for the user
    - Validation issues fixed vs consciously kept

## Field type mapping (code → model)

| Code type                                      | Qlerify field                                                                |
|------------------------------------------------|------------------------------------------------------------------------------|
| `string`, `varchar`, `text`, `char`, `uuid`    | `string`                                                                     |
| `number`, `int`, `float`, `decimal`, `bigint`  | `number`                                                                     |
| `boolean`, `bool`                              | `boolean`                                                                    |
| Nested object, JSON, `jsonb`, embedded type    | `object` (+ `relatedEntity` + `cardinality` when it's an owned entity/VO)    |
| Foreign key / relation to a **same-BC** entity | `relatedEntity` ($ref) + `cardinality`                                       |
| Foreign key to an **external-BC** aggregate    | plain `string` field named `{entity}Id` — no `relatedEntity`                 |
| Enum / union of literals                       | `string` with the allowed values captured as an invariant in the description |

## What this skill does NOT do

- **Reverse-engineer an entire codebase from scratch into a new model** — that's `workflow-creation` Phase 0.
- **Write code from the model** — that's `code-generation`. Sync detects model→code drift and hands off.
- **Delete without confirmation** — destructive changes are always user-gated.
- **Invent storage opinions** — persistence stays an implementation detail; sync reads `persistenceDecisions` and leaves them alone.
