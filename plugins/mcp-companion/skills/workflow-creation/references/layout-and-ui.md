# Qlerify Diagram Layout & UI Guide

## Purpose

How Qlerify lays out the workflow diagram and organizes its views, so you can build
well-arranged diagrams and answer questions about where things live. You never set pixel
positions — Qlerify derives placement automatically from the flow you create (`follows`,
`parallel`, `type: "decision"`, `lane`, `group`). This guide explains how those choices render.

## The workflow page at a glance

- **Diagram (top):** the main canvas. The process reads **left → right** in (roughly) chronological order, partitioned top-to-bottom into role lanes.
- **Seven tabs (below the diagram):** 1 Domain Model · 2 Entities · 3 Read Models · 4 Domain Events · 5 Backlog · 6 User Story Map · 7 History.
- **Right sidebar:** per-event detail (the selected event's elements, a form mockup of its command, file uploads) and the AI Assistant chat.

## Diagram layout

The diagram has a small, fixed vocabulary: **lanes, events, decisions, connections, and groups.**

### Lanes (swimlanes)

Horizontal rows, one per role, with the role label on the left edge. A new lane is added at the
bottom. Every event lives in exactly one lane — this is how the diagram shows *who does what*.
See "Lane Tools" in `references/tools.md` for naming rules and limits.

### Events and how they're positioned

Events are sticky-note boxes (default orange). Position is **derived, not manual**:

- An event is either a **start event** (no parent — pinned to the far left, or to the left edge
  of its group) or a **child** placed immediately to the right of its parent.
- You set the flow with `follows` (plus `parallel` / `type: "decision"`); Qlerify does placement.
- An event can have **several parents but only one child** (a single outgoing path). The one
  exception is a **decision**, which may fan out to several children.
- With multiple parents, the **first parent in the list is the "main parent"** and it decides the
  event's horizontal position (one slot to its right). Other parents just draw an extra arrow in.
- Events snap to fixed slots: users can drag them, but they always land in a slot and never on top
  of an arrow. Creating an event inserts it after the current one and pushes later events right.

> This is why a second event that `follows` the *same* parent is inserted *between* that parent and
> its existing child unless you mark it `parallel: true` — see `references/event-generation.md`.

### Decisions (gateways)

A decision is a diamond instead of a box; phrase it as a question ("Payment approved?"). Unlike a
normal event it can have **multiple outgoing branches**. Each branch carries a short condition label
(e.g. "Yes" / "No") shown next to the following event — set it with `conditionLabel`. Use a decision
for exclusive either-or paths; use `parallel` branches for steps that all happen.

### Connections (arrows)

Arrows show the order events typically unfold — a *likely* sequence, not a strict "A causes B". An
event with multiple parents gets one arrow from each. A loop-back arrow (returning to an earlier
event) routes down below both ends, runs back, then up into the target, to stay clear of the boxes.
Keeping the happy path as one forward chain — and pushing rare or returning transitions onto labels
or separate branches — is what keeps the arrows readable.

### Groups (phases / stages)

Optional vertical bands that split the canvas into labeled phases (e.g. "Order Placement",
"Fulfillment") via draggable dividers, with a header label across the top. A new group appears at the
far right; its divider is dragged left to size it and cannot cross into another group. Groups are
purely visual — they carry no domain meaning. Don't add them unless the user asks (see Phase 5),
except in state-machine workflows (Phase S) where groups are required to represent state columns
(see `references/state-machine-generation.md`).

## The tabs — where the model lives

The diagram shows only the flow; the domain model attached to each event lives in the tabs:

- **Domain Model (tab 1):** the aggregate-centric projection, grouped by bounded context. Selecting
  an event shows one horizontal slice, left → right: **read model(s) → role → command → aggregate
  root → resulting domain event** (→ the read-model projections that event updates). Multiple
  commands on the same aggregate stack against a single aggregate box. This is the data/state dual
  of the timeline.
- **Entities (tab 2):** each entity as a spreadsheet with example-data rows.
- **Read Models (tab 3)** and **Domain Events (tab 4):** dedicated pages for those schemas, grouped
  by entity, where read models and domain events are linked to each other.
- **Backlog (tab 5)** and **User Story Map (tab 6):** prioritization views of the requirement items
  (user stories, given-when-thens, …) attached to events.
- **History (tab 7):** version log; any of the last 50 versions can be restored.

## What an event carries

Beyond its name and lane, an event can hold one command, one or more read models, one aggregate
root, one domain event schema, and any number of requirement items (given-when-thens, user
stories, …). None of these are drawn on the box — they live in the tabs and sidebar. Create and link
them with the matching tools (`create_commands`, `create_read_models`, `create_entities`,
`create_domain_event_schemas`, `create_card`) — see `references/tools.md`.
