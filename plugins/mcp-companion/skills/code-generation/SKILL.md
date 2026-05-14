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
- **Persistence is an implementation detail.** The model gives types, ownership, and dependencies. It does not prescribe storage. This skill chooses how aggregates, owned entities, and value objects map onto the chosen engine — relational, document, key-value, or otherwise.

## Operating mode

**Lean toward iterating silently.** Ask the user only when proceeding without an answer would commit you to a hard-to-reverse choice. Specifically:

- Pre-flight scan finds contradictions or structurally missing pieces → **ask** (Phase 1)
- Tech stack ambiguous and no existing project to follow → **ask once with a single recommended default** (Phase 2)
- Test still failing after 3 fix attempts → **ask** (likely model bug, not code bug, Phase 5)
- Aggregate boundary or transaction scope unclear → **ask** (Phase 3)
- Minor underspecification (missing description, optional-field semantics, default value) → **decide and proceed**, note the decision in the final report

## Phase 0: Acquire the model

If the workflow hasn't been downloaded locally yet, invoke the `download` skill — it bypasses MCP processing and is much faster for large specs. Otherwise call `get_workflow` once and cache the result.

Save the spec at `<output-dir>/.qlerify/workflow.json` (output dir is chosen in Phase 2) so the model travels with the code.

Read the `$schema` URL from the spec (e.g. `https://app.qlerify.com/schemas/domain-model/v1.json`) and fetch it if you need to verify field types, allowed values, or relationship structure. Treat it as authoritative for what the model can express.

## Phase 1: Pre-flight model scan

Before generating any code, scan the model for issues that will produce broken or surprising code. Print a short report; ask the user only on hard blockers.

**Hard blockers — ask before proceeding:**

- An event has no aggregate root assigned
- An event has no command assigned to emit it
- An entity referenced by `$ref` doesn't exist in the spec
- A value object has an `id` field or an entity has no `id` field in the model
- A command's fields don't correspond to any entity attribute, and no GWT or description explains where they go, and it's not self explanatory
- A GWT contradicts an attribute invariant (e.g., GWT says `quantity: 0` is valid, attribute has `min: 1`)
- An aggregate looks suspiciously large (>~10 entities) — boundary may need splitting

**Soft signals — note and proceed:**

- Entity with no commands → likely a read-only reference entity, generate type but no handler
- Event with no read model → fine, not every event needs a query
- Command with no GWTs → generate handler from attribute-level invariants only, flag in the final report
- Missing attribute descriptions → infer purpose from name and type

## Phase 2: Choose the platform

1. **Inspect the working directory.** Three cases:
   - **Prior codegen target** (`.qlerify/codegen.json` present) — generate in place; match the recorded stack.
   - **Existing project, no anchor** (`package.json`, `go.mod`, etc.) — almost always legacy. Ask once: extend in place (rare, same-stack only) or generate into a sibling directory (default — `../{workflowName}-new`). With the sibling option, the model file and `.qlerify/codegen.json` go there too.
   - **Empty directory** — generate in place.
2. **Greenfield stack.** Default the runtime to **Node.js + TypeScript** (types map cleanly to attribute types). Pick framework, ORM, database, and test runner yourself based on what fits the model's shape. State the full stack in one line; don't offer a menu.
3. **User-specified stack.** Honor it exactly, including older or unconventional choices.

Record the chosen stack in `.qlerify/codegen.json` (created if absent) so subsequent generation runs stay consistent.

## Phase 3: Map the model to a persistence design

The model gives types, ownership, and dependencies — never storage. The agent chooses how to persist, guided by these principles:

- **Aggregate = consistency boundary.** Whatever the storage primitive is (table row, document, partition), one command commits one aggregate atomically. Cross-aggregate effects propagate via domain events, not joined transactions.
- **Owned entities and value objects belong with their root.** Embed where the storage model supports it (document subdocs, JSON columns); otherwise link with whatever the engine uses for relationships, and propagate deletion of the root to its children.
- **Value objects have no domain identity, but may carry a stable ID at system boundaries** when there's a real need — picking from a UI list, deduplication, audit logging. Set-replacement semantics still hold (the whole VO is overwritten on update, not patched field-by-field). Don't add IDs to VOs just because the storage engine wants them — only when something outside the aggregate needs to reference the VO.
- **External references stay opaque.** A `*Id` field pointing at another bounded context's aggregate carries no integrity constraint and no cascade across that boundary.
- **Aggregates need concurrency control.** Optimistic versioning on the root is a common default; conditional writes, pessimistic locking, or transaction isolation are all valid translations.

Translation examples — pick the idiomatic one for the chosen stack:

- **Relational (Postgres/MySQL):** one table per aggregate root, separate tables for owned entity collections with FK + cascade, value objects inlined as columns or JSON, optimistic locking via a `version` integer.
- **Document store (Mongo/DynamoDB/Firestore):** one document per aggregate root, owned entities and value objects embedded as subdocuments, optimistic locking via document revision or conditional writes.

Record any non-derivable persistence decisions in `.qlerify/codegen.json` — e.g. "Address VO embedded inline in Order" vs "Address VO stored as its own collection because the UI picker needs to list saved addresses" — so the `sync` skill and future generations can interpret the code.

## Phase 4: Generate code

Generate in this order so each step has the foundation it needs.

### 4.1 — Project scaffold

Initialize the project, install dependencies, configure the test runner, set up the storage connection.

### 4.2 — Persistence schema

Translate entities and value objects into the persistence layer using the Phase 3 mapping. If the stack uses schema migrations, generate one per generation run, named after the model version hash so future deltas are traceable. For schema-less stores, the hash recorded in `.qlerify/codegen.json` is enough.

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
4. Persists the aggregate with the stack's concurrency control (optimistic version bump, conditional write, or equivalent)
5. Emits the corresponding domain event on the in-process bus
6. Returns success, or surfaces invariant violations as 4xx (not 5xx) using whatever the stack idiomatically uses to distinguish domain errors from infrastructure errors

**Authorization is auth-ready, not auth-stubbed.** The role check runs through a pluggable auth middleware (Fastify hook, Express middleware, NestJS Guard — stack-specific); its default implementation reads the role from an env var or request header so the server can be run and tested immediately. Swap in real auth (JWT, OAuth, session store) by replacing the middleware — handler code doesn't change. Flag the current wiring in the final report so the user knows what to replace before deploying.

### 4.5 — Read models

A read model can be a direct read against the write side, a database view, or a precomputed projection — pick what's idiomatic for the chosen stack. Default to direct reads; reach for precomputed projections only with a concrete reason (cross-BC reads, measured perf needs, event sourcing).

The `entity` link names what to read from, `isFilter: true` fields are query parameters, and the rest are response fields.

### 4.6 — Domain events

Default to an **in-process event bus**. Each handler emits its event synchronously after a successful command. Subscribers are registered at startup.

Cross-bounded-context events: keep them in-process while the system is a single deployable. When the bounded contexts split into separate services, promote those events to an external broker (Kafka, RabbitMQ, NATS) — flag candidates in the final report.

**Do not generate a state machine that enforces event ordering.** Two different sequences of events may both be legal — the model doesn't always pin them down. Invariants on commands are what gate "illegal" transitions; the diagram is one valid path, not the only one.

### 4.7 — Tests from GWTs

For every event with `acceptanceCriteria`, generate one test per Given-When-Then string. Each test:

- **Given** clause → fixture setup. If the Given references state not literally present in the text (e.g., "Given an Order exists" without specifying how), derive the setup by replaying earlier commands in the same workflow that produce the required preconditions.
- **When** clause → invoke the command handler with the GWT inputs
- **Then** clause → assert the expected event was emitted with the expected payload, and/or the expected error was returned

Place tests structurally: one test file per aggregate, one `describe` per command, one `it` per GWT. Predictable paths let future delta-apply find existing tests to update.

After generating, run the full suite (Phase 5).

### 4.8 — Frontend

Aim for demo-quality polish — a UI you'd be willing to show a stakeholder, not a debug console. Drive layout from the model:

- **Navigation by lane (role), not aggregate.** Landing page is the role's task list / inbox, built from the read models that role can see.
- **Read models** become real list and detail views; `isFilter: true` fields are filters/search, not raw query-string boxes.
- **Commands** become forms with labels, inline validation from attribute invariants and GWT preconditions, and meaningful error/success states. Events surface as toasts or status updates.
- **Pick UI affordances that match the domain.** A `Cart` aggregate gets a product grid with images and add-to-cart buttons; an `Order` aggregate gets a status timeline; a `Subscription` aggregate gets plan-picker cards. Avoid debug-console tells: no JSON dumps, no raw IDs in URLs, no "click to test command" buttons.
- **Seed the backing store with beautifully crafted, domain-appropriate data — not the model's `exampleData` verbatim.** For a cart, invent a small catalog of plausible products (real-sounding names, sensible prices, images from a placeholder service like `picsum.photos` or `placehold.co`). For an order history, generate believable past orders across statuses. For subscriptions, named tiers with realistic pricing. The model's `exampleData` is the floor, not the ceiling, and it never belongs as pre-filled form values — that reads as a demo.

Apply a polished styling baseline — Tailwind + small components, or Pico.css when avoiding a build step. Typography, spacing, and visual hierarchy do most of the work; bare HTML is most of what makes a generated UI feel like a test tool. For non-default stacks, ask once whether to include a frontend; the user may already have one or prefer a separate API client.

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
- `aggregates` — names generated so far (merge with any prior list), so a later run knows what's done and what's pending
- `persistenceDecisions` — anything from Phase 3 that isn't directly derivable from the model (e.g., "VO `Address` stored inline on `Order`", "VO `LineItem.discount` stored as separate table with synthetic id")
- `generatedAt` — ISO timestamp

This anchor lets a future invocation add a new aggregate or apply a model delta to existing ones as a targeted patch, instead of regenerating from scratch. Code-side drift (developer-added fields, renamed handlers, etc.) flows back through the `sync` skill — this skill is model → code; `sync` is code → model.

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
- **Deployment, CI/CD, infrastructure-as-code** — out of scope. The code itself is production-grade; deploying it, wiring CI, and provisioning infrastructure are separate concerns.
