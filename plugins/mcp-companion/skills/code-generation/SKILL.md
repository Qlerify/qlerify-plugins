---
name: code-generation
description: >-
  Generate production-ready code from a Qlerify domain model. Use when the user
  asks to "generate code from the model", "implement the workflow", "scaffold
  from Qlerify", "build the aggregate", "code up the domain model", "create the
  app from the model", or after a workflow has been modeled or extracted and the
  next step is producing runnable code on a target tech stack. Pairs with the
  workflow-creation skill (which produces the model) and the sync skill (which
  keeps model and code aligned over time).
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, WebFetch, mcp__plugin_mcp-companion_qlerify__*
---

# Code Generation from Domain Model

Generate production-ready code from a Qlerify workflow. The model is the source of truth — entities, value objects, commands, read models, domain events, bounded contexts, invariants on attributes, and Given-When-Then acceptance criteria on events all map to code on a chosen tech stack.

This skill produces a working, tested application. The model lives in version control alongside the code, and future changes flow in both directions: re-running this skill applies model-side changes to the code; the `sync` skill applies code-side drift back into the model.

## Core principles

- **The model is normative; the diagram is illustrative.** Event ordering on the board shows one valid sequence — actual systems may interleave events in any order. Invariants on attributes, commands, and GWTs gate state changes, not the visual sequence. Generate command handlers that enforce invariants and emit events; do not generate rigid state machines that hard-code the diagram sequence.
- **Invariants are scattered.** Collect them from three sources before generating any handler: command-level GWTs, attribute properties (type, required, min/max, allowed values), and prose descriptions on attributes. Missing any of the three produces silently incorrect code.
- **Persistence is an implementation detail.** The model gives types, ownership, and dependencies. It does not prescribe storage. This skill chooses tables, foreign keys, cascade behavior, and concurrency strategy.

## Operating mode

**Lean toward iterating silently.** Ask the user only when proceeding without an answer would commit you to a hard-to-reverse choice. Specifically:

- Pre-flight scan finds contradictions or structurally missing pieces → **ask** (Phase 1)
- Tech stack ambiguous and no existing project to follow → **ask once with a single recommended default** (Phase 2)
- Test still failing after 3 fix attempts → **ask** (likely model bug, not code bug, Phase 5)
- Aggregate boundary or transaction scope unclear → **ask** (Phase 3)
- Minor underspecification (missing description, optional-field semantics, default value) → **decide and proceed**, note the decision in the final report

## Phase 0: Acquire the model

If the workflow hasn't been downloaded locally yet, invoke the `download` skill — it bypasses MCP processing and is much faster for large specs. Otherwise call `get_workflow` once and cache the result.

Read the `$schema` URL from the spec (e.g. `https://app.qlerify.com/schemas/domain-model/v1.json`) and fetch it if you need to verify field types, allowed values, or relationship structure. Treat it as authoritative for what the model can express.

## Phase 1: Pre-flight model scan

Before generating any code, scan the model for issues that will produce broken or surprising code. Print a short report; ask the user only on hard blockers.

**Hard blockers — ask before proceeding:**

- An event has no aggregate root assigned
- An entity referenced by `$ref` doesn't exist in the spec
- A value object has an `id` field (likely misclassified — should be an entity)
- A command's fields don't correspond to any entity attribute, and no GWT or description explains where they go
- A GWT contradicts an attribute invariant (e.g., GWT says `quantity: 0` is valid, attribute has `min: 1`)
- An aggregate looks suspiciously large (>~7 entities) — boundary may need splitting

**Soft signals — note and proceed:**

- Entity with no commands → likely a read-only reference entity, generate type but no handler
- Event with no read model → fine, not every event needs a query
- Command with no GWTs → generate handler from attribute-level invariants only, flag in the final report
- Missing attribute descriptions → infer purpose from name and type

## Phase 2: Choose the platform

1. **Inspect the working directory first.** If a project already exists (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, etc.), match the existing stack — framework, ORM, test runner, package manager. Do not introduce a parallel stack alongside it.
2. **Greenfield directory.** Ask the user once with a single recommended default — not a menu. Recommended default:
   - **Node.js + TypeScript + Fastify + Prisma + PostgreSQL + Vitest**
   - Why: TypeScript types map cleanly to attribute types, Prisma migrations align with entity changes, Fastify gives a clean command-handler shape, Vitest matches the GWT → test pattern with minimal ceremony.
3. **User-specified stack.** Honor it exactly, including older or unconventional choices.

Record the chosen stack in `.qlerify/codegen.json` (created if absent) so subsequent generation runs stay consistent.

## Phase 3: Map the model to a persistence design

Decide before generating code, not during:

| Domain concept                              | Default persistence choice                                                 |
|---------------------------------------------|----------------------------------------------------------------------------|
| Aggregate root                              | One table; transaction boundary = single-aggregate write                   |
| Owned entity (`one-to-many` from aggregate) | Separate table, FK to aggregate root, `ON DELETE CASCADE`                  |
| Value object, single                        | Inline columns on the owning row, OR JSON column for nested VOs            |
| Value object, collection                    | Separate table with FK + synthetic row `id`, cascade from owner            |
| External reference (other bounded context)  | `*Id` column, no FK constraint across BCs, no cascade                      |
| Concurrency                                 | Optimistic locking via `version` integer on aggregate root                 |

**Synthetic VO IDs are fine in storage.** The domain model forbids `id` on value objects, but row identity in SQL is a practical necessity. Treat the storage ID as an implementation detail — never surface it in command payloads, event payloads, or API responses; set-replacement semantics still hold (the whole VO row gets replaced on update).

**Aggregate boundary = transaction boundary.** Cross-aggregate effects propagate via domain events, not joined transactions.

Record any non-obvious persistence decisions in `.qlerify/codegen.json` so future generations and the `sync` skill can interpret the code correctly.

## Phase 4: Generate code

Generate in this order so each step has the foundation it needs.

### 4.1 — Project scaffold

Initialize the project, install dependencies, configure the test runner, set up the database connection. One migration directory ready to receive Phase 4.2 output.

### 4.2 — Schema and migrations

Translate entities and value objects into the persistence layer using the Phase 3 mapping. One migration per generation run, named after the model version hash so future deltas are traceable.

### 4.3 — Aggregate types and invariant guards

Generate one module per aggregate (typically: one folder per bounded context, one module per aggregate root inside it).

Each module contains:
- Entity and value object types matching model attributes (TypeScript types, Python dataclasses, Java records — stack-appropriate)
- **Consolidated invariant guards** — for each command, merge attribute-level invariants + command-level invariants + GWT preconditions into a single guard function. Order:
  1. Attribute constraints (type, required, min/max, allowed values, regex)
  2. Command-level invariants from the model
  3. GWT preconditions (the Given and pre-When state checks)
- Aggregate root class/factory that exposes one method per command, each running its guard before mutating state

A command's argument structure mirrors the aggregate's nested entity/VO structure (as enforced by `workflow-creation` Phase 3 Step 5). The generated handler should accept that exact shape — don't flatten or rename.

### 4.4 — Command handlers and API surface

One handler per command, mounted at an HTTP endpoint (or RPC method, per stack). Each handler:

1. Authorizes — only the role on the command's lane may invoke it; reject otherwise
2. Loads the aggregate by ID (full load by default — see "advanced" below)
3. Calls the aggregate method, which runs invariant guards and mutates state
4. Persists the aggregate with optimistic-lock version bump (`UPDATE … WHERE id = ? AND version = ?`)
5. Emits the corresponding domain event on the in-process bus
6. Returns success, or a domain-specific error mapped to a 4xx response on invariant violation

**Invariant violations must be distinguishable from infrastructure errors** — use a dedicated error type (e.g. `DomainError`, `InvariantViolation`) so the API layer can map cleanly to 400/422 instead of 500.

### 4.5 — Read models as queries

For a prototype, implement each read model as a direct query against the write tables — not a materialized projection. The `entity` link on the read model tells you which table to query; `isFilter: true` fields become query parameters; remaining fields become the projection.

Promote a read model to a materialized projection only when:
- It crosses bounded context boundaries (cross-BC join is undesirable), or
- It has a measured performance requirement the direct query can't meet, or
- It needs eventual consistency semantics distinct from the write side

Stay simple by default; CQRS-on-day-one is usually wasted complexity for a prototype.

### 4.6 — Domain events

Default to an **in-process event bus**. Each handler emits its event synchronously after a successful command. Subscribers are registered at startup.

Cross-bounded-context events: keep them in-process for the prototype, but flag in the final report as candidates for an external broker (Kafka, RabbitMQ, NATS) when the system splits into separate services.

**Do not generate a state machine that enforces event ordering.** Two different sequences of events may both be legal — the model doesn't always pin them down. Invariants on commands are what gate "illegal" transitions; the diagram is one valid path, not the only one.

### 4.7 — Tests from GWTs

For every event with `acceptanceCriteria`, generate one test per Given-When-Then string. Each test:

- **Given** clause → fixture setup. If the Given references state not literally present in the text (e.g., "Given an Order exists" without specifying how), derive the setup by replaying earlier commands in the same workflow that produce the required preconditions.
- **When** clause → invoke the command handler with the GWT inputs
- **Then** clause → assert the expected event was emitted with the expected payload, and/or the expected error was returned

Place tests structurally: one test file per aggregate, one `describe` per command, one `it` per GWT. Predictable paths let future delta-apply find existing tests to update.

After generating, run the full suite (Phase 5).

## Phase 5: Test-and-iterate loop

1. Run the full test suite.
2. For each failing test, attempt one fix and re-run.
3. **Iterate up to 3 times per failing test.** Stop looping on the same failure.
4. If a test still fails after 3 attempts, **stop and ask the user.** Likely causes:
   - The GWT contradicts an attribute invariant (model is inconsistent)
   - The Given clause assumes context not derivable from prior commands
   - A model-level ambiguity that needs human judgment
5. **Never weaken a test to make it pass.** Don't relax assertions, don't `.skip()`, don't change inputs to match the implementation. A failing test means either the code is wrong or the model is wrong — both warrant escalation, not silencing.

## Phase 6: Anchor for future sync

Write `.qlerify/codegen.json` with:

- `workflowId` and `workflowName`
- `modelHash` — content hash of the canonical JSON form of the spec, so a future run can compute a diff
- `stack` — the platform choices from Phase 2
- `persistenceDecisions` — anything from Phase 3 that isn't directly derivable from the model (e.g., "VO `Address` stored inline on `Order`", "VO `LineItem.discount` stored as separate table with synthetic id")
- `generatedAt` — ISO timestamp

This anchor lets a future invocation of this skill compute a model delta and apply it as a targeted patch instead of regenerating from scratch. Code-side drift (developer-added fields, renamed handlers, etc.) flows back into the model through the `sync` skill — this skill is the model → code direction; `sync` is the code → model direction.

## Anchoring generated code to the model

Anchor **by structure, not by markers**:

- One folder per bounded context (`src/{boundedContext}/`)
- One module per aggregate (`src/{boundedContext}/{aggregate}/`)
- One file per concern within the module: `types.ts`, `invariants.ts`, `commands.ts`, `handlers.ts`, `queries.ts`
- One test file per aggregate (`{aggregate}.test.ts`), structured `describe(command) → it(gwt)`

Predictable paths let a delta-apply locate existing code without comment markers polluting the file. Only fall back to `// qlerify: <ref>` markers if the structure-only approach genuinely can't disambiguate (rare).

## Advanced: per-command aggregate optimization

The default — load the full aggregate, mutate, save — is safe and matches the mental model. For performance-sensitive commands, an advanced user may opt into partial loads. Only do this when **both** are true:

1. The user explicitly asks for the optimization
2. The command's invariants can be proven (from attributes, command rules, and GWTs) to depend only on the loaded slice — no invariant references state outside the slice

If either condition fails, fall back to full-aggregate load. Saving a query isn't worth a silent invariant violation.

## Stop conditions

The skill is done when:

1. Phase 1 hard blockers are resolved (or user has consciously deferred them)
2. The project builds
3. Every GWT has a corresponding passing test
4. `.qlerify/codegen.json` is written
5. A short final report names: what was generated, what platform/persistence defaults were chosen, what was deferred or assumed

## What this skill does NOT do

- **Reverse-engineering** code into a model — that's `workflow-creation` Phase 0
- **Code-drift sync** back into the model — that's the `sync` skill
- **Event Sourcing infrastructure** — only generate domain event *schemas* if they exist in the model (the model includes them only for Event Sourcing workflows; `workflow-creation` Step 7 is opt-in). For default state-based applications, the in-process bus from 4.6 is sufficient.
- **Deployment, CI/CD, infrastructure-as-code** — out of scope. Output is a runnable local prototype suitable for production-promotion later.
