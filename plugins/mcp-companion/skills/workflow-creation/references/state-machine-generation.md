# State-Machine Aggregate Layout Guide

How to detect that an aggregate is really a **state machine** and lay its lifecycle out on the
Qlerify timeline, instead of flattening it into a misleading linear chain. Builds on the spatial
model in `references/layout-and-ui.md`.

## When this applies (detection)

Most aggregates are a simple lifecycle and a single linear event chain is the right model. Reach
for a state machine **only** when all of these hold:

- an explicit **status/state field** (an enum) with several values (≈4 or more), **and**
- transitions are **guarded by that status** — a command is only valid from certain states, **and**
- at least one of: a **cycle** (reopen / revisit returns to an earlier state), **multiple terminal
  states**, or a **guard-fork** (one command lands in different states depending on input).

One or two of these alone is not enough — keep it a normal linear flow. If detection is borderline,
**ask the user** whether to map states onto the timeline or keep a linear flow.

The tell-tale sign: a plain chain of the lifecycle events (`Drafted → Registered → … → Removed`)
reads as one *sequence* when those are really *alternative branches*. That mismatch is the signal
you need a state machine, not a chain.

## The state-transition map (gate artifact)

Before any MCP writes, produce `.qlerify/aggregates/{name}-state-machine.md` and get user approval
(alongside / extending the Phase 0 aggregate artifact). It contains:

1. **States** — every enum value, each marked **entry** / **intermediate** / **terminal**.
2. **Transition table** — one row per transition:

   | fromState | trigger (command / event) | toState | guard |
   |-----------|---------------------------|---------|-------|

3. **Marks** — note which transitions are **cycles** (back to an earlier state), **guard-forks**
   (same trigger → different `toState` by input), or **self-loops** (no status change).
4. **Flags** (these are NOT drawn as edges — see Guardrails):
   - `cross-instance` — the transition spawns or consumes a **different** aggregate row
     (e.g. Unlink creates a new Proposal receipt).
   - `external` — the trigger lives outside this codebase / aggregate (mark it upstream).
   - `unverified` — the state exists but **no code in this repo performs the transition**
     (a modeling assumption).

A mermaid `stateDiagram-v2` is a good companion, but the **table is the source of truth**.

## Layout method (state → timeline)

Lay the machine out left-to-right as a **DAG**:

1. **Column = postcondition state.** Place each event in the column of the state it lands in. The
   column becomes a Qlerify **group** named after the state.
2. **Order columns by longest-path-from-entry.** Entry states leftmost, terminal states rightmost.
3. **Cut cycles into a DAG.** Reverse the minimal set of back-edges (reopen, unlink, re-supplement)
   and render them as **forward** flow into a **reappearing** column — never a loop-back arrow. A
   reappearing state needs a distinct group name (`DRAFT` / `DRAFT 2`) because group names are unique.
4. **Self-loops stay in-column.** No-status-change events (edit, upload image) chain rightward
   inside the same column; never draw them as arrows.
5. **One linear `follows` spine drives x-position** — each event is parent + 1 slot. Keep it a
   single chain; do **not** use `parallel` (it scrambles the slot math).
6. **Guard-fork = one event per landing column**, behind a `decision` (Register → Registered /
   Pending).
7. **Alternate entry into a mid-lifecycle state = a `start` event in that column**
   (`follows: "start"`) — an extra "way in" that doesn't disturb the spine.
8. **Decisions carry the guards** — model guard-forks as a `decision` with `conditionLabel` on each
   branch.

## Guardrails — draw the spine, declare the rest

The on-canvas machine is a happy-path **validation lens** ("show me my state machine on your
timeline"), not a complete transition graph. Keep it clean:

- **N-to-1 (delete-from-anywhere)** → one terminal column reached from the spine, plus a GWT that
  enumerates the eligible source states — **not** an edge from every state. Heuristic: if a
  transition's source is a *set* of states rather than one, it's a rule, not a line.
- **`cross-instance` / `external` / `unverified`** transitions → GWTs, notes, or an explicit
  "assumption" marker — never drawn edges.
- Everything you can't draw cleanly lives in `acceptanceCriteria` (Phase 2 / 3), which still feed
  code-gen and tests — so the unhappy paths aren't lost, just not cluttering the picture.

## Building it with MCP

- Create the spine with `create_domain_events` (a single `follows` chain) — see
  `references/event-generation.md`.
- Create the state **groups** with `create_group` in left→right state order, then assign each
  column's first event with `update_domain_event(group:)`. Group order is creation order and is
  immutable, so create them in timeline order.
- Create guard-forks as `decision` events with `conditionLabel` branches.
- Wire multi-parent **fan-ins** (a state reached from several predecessors; a single decision with
  several parents) with `add_connection(from, to, conditionLabel?)` — it appends a parent without
  moving the target. See `references/tools.md`.
- **Read caveat:** `get_workflow` serializes a multi-parent *decision* as one entry per parent (the
  same decision name repeated). Don't infer a decision's parent-count — or a branch "explosion" —
  from the export; it's one diamond. (Multi-parent *events* serialize correctly as a `follows` array.)

## Worked fragment (Receipt)

States: `Draft` (entry) · `Proposal` · `Matched` · `Registered` · `Pending` · `Supplement` ·
`Supplemented` · `Approved` (terminal) · `Denied` · `Deleted` (terminal).

- **Guard-fork:** `Register` → `Registered` (Private / CompanyCreditCard) or `Pending`
  (CompanyAccount) — one event per column behind an `Account type?` decision.
- **Cycle:** `Denied → Reopen → Draft` — render as a reappearing `DRAFT 2` column flowing forward,
  not a loop-back to the first `DRAFT`.
- **N-to-1:** `Delete` from almost every state → a single `DELETED` terminal column + a GWT
  "Given a receipt in any non-Approved state, When it is removed, Then it is Deleted".
- **cross-instance:** `Unlink` spawns a *new* `Proposal` row → flag and note, not an edge.
- **unverified:** `Approve` / `Deny` have no writer in this repo → mark as an assumption.
