# State-Machine Aggregate Layout Guide

How to detect that an aggregate is really a **state machine** and lay its lifecycle out on the
Qlerify timeline, instead of flattening it into a misleading linear chain. Builds on the spatial
model in `references/layout-and-ui.md`.

## When this applies (detection)

Most aggregates are a simple lifecycle and a single linear event chain is the right model. Reach
for a state machine **only** when all of these hold:

- a notion of **state** — a mode that decides which operations are legal next — that the aggregate
  moves through, with several distinct states (≈4 or more). Most often a **status enum**, but state
  has many encodings (see **Where the state lives** below), **and**
- operations are **guarded by that state** — a command is only valid from certain states, **and**
- at least one of: a **cycle** (reopen / revisit returns to an earlier state), **multiple terminal
  states**, or a **guard-fork** (one command lands in different states depending on input).

One or two of these alone is not enough — keep it a normal linear flow. If detection is borderline,
**ask the user** whether to map states onto the timeline or keep a linear flow.

The tell-tale sign: a plain chain of the lifecycle events (`Drafted → Registered → … → Removed`)
reads as one *sequence* when those are really *alternative branches*. That mismatch is the signal
you need a state machine, not a chain.

## Where the state lives

"State" is whatever determines which operations are legal next — **not** necessarily a status field.
A status enum is the commonest and easiest encoding, so look for it first, but if there isn't one (or
it doesn't tell the whole story) check for these before concluding "not a state machine":

1. **A status / state enum field** — explicit and easiest (`Receipt.status`).
2. **Lifecycle timestamps / milestone fields** — state = which nullable fields are set
   (`paidAt`, `shippedAt`, `cancelledAt` ⇒ Pending / Paid / Shipped / Canceled). No enum at all.
3. **Existence of a related entity or child** — state = presence/absence of a relationship (a Cart
   with an Order is "checked out"; a Subscription with an open billing period is "active").
4. **A combination of booleans** — the state is the *tuple* of flags (`isProposal`, `booked`,
   `expensePaid` layer substates on top of a status).
5. **The latest row of a status-history / event-log entity** — state = the `type` of the most recent
   entry (Receipt's `ReceiptEvent.type` carries `Approved` / `Denied` even though no code ever sets
   `status = Approved`).
6. **(Event-sourced) the fold of events** — there is no stored state field; state is derived by
   replaying events.

Two consequences:

- **The state-bearer need not be the aggregate root.** The meaningful lifecycle may live on a child
  entity (e.g. per-line fulfilment status) or a different aggregate — model the state machine of
  whatever actually carries it.
- **A single status enum often under-counts the real state space.** When flags or relationships also
  gate behavior, the true state is the *combination* — fold those substates in rather than trusting
  the enum alone.

Whatever the encoding, derive the **state set** from it; the transition table, altitude call, and
layout below are the same once you have the states.

## Altitude — what the diagram is the truth of

A state machine can be drawn at two altitudes, and they only diverge when the code is thin:

- **Service altitude (code-faithful) — the default; recommend this.** Model exactly what *this*
  codebase's mutation surface does. Transitions with no in-repo writer (other services, UI-only,
  business convention) are recorded and flagged but **not drawn**. Lean and exact; can look thin when
  the code is a passthrough.
- **Business-lifecycle altitude** — model the aggregate's full life as a domain expert recognizes it,
  using the state set as the skeleton. Draw **all** states and the business-known transitions,
  each speculative edge visibly **marked as an assumption**. Richer; deliberately crosses
  the aggregate/code boundary into other systems and tribal knowledge.

**First check whether the code actually guards the transitions.** Many aggregates have a status enum
but a *generic setter* (`entity.status = whatever-the-caller-passed`) with no rules — the real state
machine lives in the UI, another service, or convention, not this code. That is a **partial** state
machine: a few transitions may be guarded (e.g. link requires `Proposal`) while the rest are
unenforced. When the code *is* genuinely guarded, the two altitudes coincide — no choice to make.
When it's a passthrough, they diverge sharply, so **ask the user which altitude they want — and
recommend the code-faithful service altitude.** If it looks thin, prefer widening the analysis (e.g.
pull in the frontend or the owning service) over switching to speculation. The altitude decides what
is *drawn* vs *declared* and how many columns appear.

## The state-transition map (gate artifact)

Before any MCP writes, produce `.qlerify/aggregates/{name}-state-machine.md` and get user approval
(alongside / extending the Phase 0 aggregate artifact). It contains:

1. **States — the full set.** List **every** state (even ones no code reads
   or writes), each marked **entry** / **intermediate** / **terminal**. Never silently drop one — a
   state the code is silent about is exactly the one a reviewer will miss.
2. **Transition table** — one row per transition, each carrying an **evidence** level so nothing is
   lost and the altitude call is informed:

   | fromState | trigger (command / event) | toState | guard | evidence |
   |-----------|---------------------------|---------|-------|----------|

   `evidence` is one of: **code** (a method in this repo performs it — note whether it is actually
   guarded or a blind setter), **external** (another service/system performs it, e.g. a booking
   agent), or **business** (the domain expert knows it happens but no in-scope code implements it — a
   recorded assumption). **Include the `external` / `business` transitions you know about even when
   the code is silent** — record-then-decide beats dropping them.
3. **Marks** — tag each transition as a **cycle** (back to an earlier state), **guard-fork** (same
   trigger → different `toState` by input), **self-loop** (no status change), or **cross-instance**
   (spawns/consumes a *different* aggregate row, e.g. Unlink creates a new Proposal receipt).
4. **State of guarding** — state plainly whether the code enforces the transitions or is a generic
   setter, and which specific transitions are actually guarded. This drives the altitude call above.

A mermaid `stateDiagram-v2` is a good companion, but the **table is the source of truth**.

## Layout method (state → timeline)

Lay the machine out left-to-right as a **DAG**:

1. **Column = postcondition state, one column per distinct state.** Place each event in the column of
   the state it lands in; the column becomes a Qlerify **group** named after that state. Give **each
   state its own column** — do **not** merge unrelated states that merely sit at the same depth into a
   shared column (e.g. `Supplement / Matched`); that produces confusing labels. Merge two states only
   when they are genuinely the same behavioral phase.
2. **Order columns by longest-path-from-entry.** Entry states leftmost, terminal states rightmost.
3. **Cut cycles into a DAG.** Reverse the minimal set of back-edges (reopen, unlink, re-supplement)
   and render them as **forward** flow into a **reappearing** column — never a loop-back arrow. A
   reappearing state needs a distinct group name (`DRAFT` / `DRAFT 2`) because group names are unique.
4. **Self-loops stay in-column.** No-status-change events (edit, upload image) chain rightward
   inside the same column; don't model them as loop-back transitions.
5. **One linear `follows` spine drives x-position** — each event is parent + 1 slot. Keep it a
   single chain; do **not** use `parallel` (it scrambles the slot math).
6. **Guard-fork = one event per landing column**, behind a `decision` (Register → Registered /
   Pending).
7. **Alternate entry into a mid-lifecycle state = a `start` event in that column**
   (`follows: "start"`) — an extra "way in" that doesn't disturb the spine.
8. **Decisions carry the guards** — model guard-forks as a `decision` with `conditionLabel` on each
   branch.

## Guardrails — draw the spine, declare the rest

At **service altitude** the on-canvas machine is a happy-path **validation lens** ("show me my state
machine on your timeline"), not a complete transition graph. Keep it clean:

- **N-to-1 (delete-from-anywhere)** → one terminal column reached from the spine, plus a GWT that
  enumerates the eligible source states — **not** an edge from every state. Heuristic: if a
  transition's source is a *set* of states rather than one, it's a rule, not a line.
- **`external` / `business` / `cross-instance`** transitions → GWTs, notes, or an explicit
  "assumption" marker — not drawn edges.
- Everything you can't draw cleanly lives in `acceptanceCriteria` (Phase 2 / 3), which still feed
  code-gen and tests — so the unhappy paths aren't lost, just not cluttering the picture.

At **business-lifecycle altitude** you instead *draw* the `external` and `business` transitions too —
as their own columns, reappearing columns, and fan-ins (`add_connection`) — with every speculative
edge visibly marked as an assumption. The N-to-1 delete rule still holds (one terminal column + a
GWT) at either altitude.

## Building it with MCP

- **Create the state groups first, before the event spine:** call `create_group` once per state in
  left→right state order (group order is creation order and is immutable, so create them in state
  order). Creating the columns up front means each event is born into its column — there is no later
  `update_domain_event(group:)` re-assignment pass that shifts the diagram and exposes collisions.
- Build the spine with `create_domain_events` (a single `follows` chain), setting `group: "<state>"`
  on the **first event of each column**; later events inherit their group along the chain, and each
  column's position is derived from where its events land, so the boundaries line up automatically.
  See `references/event-generation.md`.
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
- **business / external:** `Approve` / `Deny` (and the `Denied → Reopen → Draft` cycle) have no writer
  in this repo — they're set by a separate booking service. At **service altitude** they live in GWTs
  marked `external` / `business`; at **business-lifecycle altitude** they become their own drawn
  `Approved` / `Denied` / reappearing-`Draft` columns, each marked as an assumption.
