---
name: connector-building
description: >-
  Build, test, fix and schedule the connectors that fill a Qlerify Live workflow's tables, for one table or the
  whole workflow in one go, through the Qlerify Live MCP server (tools such as list_workflows, create_connector,
  test_connector_code and ingest_connector). Use it whenever the user wants data in Qlerify Live: "build the
  connectors", "fill the tables", "connect Live to HubSpot / Postgres / this API", "ingest", "simulate demo data",
  "populate the workflow", "keep it in sync", "fix this connector", "why are the cases wrong", or when ingested data
  shows the model needs a change. Use it even when the user only names a table or a source system, as long as the
  Qlerify Live MCP tools are available.
allowed-tools: Read, WebFetch, WebSearch, mcp__qlerify-live__*, mcp__qlerify__*
---

# Building Qlerify Live connectors

Qlerify Live runs a Qlerify workflow model as a live app. Each entity in the model becomes a table. A connector is a
small JavaScript module that fills one table from a source. After every ingest the platform works out which domain
events the landed rows imply and groups them into cases, one case per end-to-end run of the workflow. So a connector
is only right when its rows produce the right cases and events, not merely when rows land.

You work through the Qlerify Live MCP server. Every tool except `list_workflows` takes a `workflowId`.

## If the tools are missing

If no `qlerify-live` tools are available, the MCP server is not connected. Tell the user: in Qlerify Live, open
Organization admin, then the MCP tab, create a token, and run the command it shows. Building connectors needs
organization admin rights, so a refusal that names a permission means the token's owner lacks them.

## The whole workflow, start to finish

1. **Read the workflow.** `list_workflows`, then `get_workflow_definition` for the one the user means. It lists every
   event in order with its acceptance criteria (Given/When/Then) and every entity with its fields and allowed values.
   Those acceptance criteria are the spec your connectors must satisfy. Then `list_model_kinds` shows the tables per
   system and the connectors already on them, and `list_table_rows` shows what a table already holds.
2. **Plan the order.** A child row joins its parent's case only through a field holding the parent row's exact id
   (`orderId` holding an `Order.id`). So fill parents before children: the root table first, then the tables that
   point at it, and so on down. A table no event is rooted on is reference data; fill it before any connector that
   reads it. A table has at most one connector, so repair an existing one instead of creating a second.
3. **Ask once, for everything.** Settle what you cannot work out yourself for all tables at once, grouped by source
   system, instead of one table at a time: where the data lives and how to reach it, how often it should refresh,
   whether any connector writes to another system, and for each child table which source field carries the parent's
   id. Skip anything the workflow definition or the user already answered. See "When to ask".
4. **Build each table** in the planned order: `create_connector`, credentials if the source needs them,
   `get_connector_brief`, write the code, `test_connector_code` until it is right, `save_connector_code`,
   `set_connector_date_roles`, `adapter_dry_run`, then `ingest_connector` with a small first batch. Check it, then
   ingest the rest. That first batch lands for real: if checking it makes you change the row ids or the linking, empty
   the table with `clear_table` before ingesting again, since an ingest never removes rows. Details in "Writing a
   connector".
5. **Check the result** after each table, not only at the end. See "Checking the result".
6. **Fix the model** when the data proves it wrong. See "When the model is wrong".
7. **Keep it running** once everything checks out: schedules, wake-ups and notifications. See
   `references/connector-rules.md`, "Keeping connectors running".

Tell the user briefly which table you are on and what you found. Do not stop to ask permission for steps the user
already asked for; "When to ask" lists the exceptions.

## Writing a connector

Write the code yourself. `get_connector_brief` returns exactly what the platform's own code writer is told: the
target fields and their allowed values, the events this table drives with their acceptance criteria, real ids from
parent tables, fields already seen at the source, and the code contract (`fetchRows(ctx)`, what `ctx` offers, the
re-run rules). Follow it: it is the contract the platform runs your code against.

- `test_connector_code` runs your code in the platform's sandbox, against the live source and real snapshots of the
  workflow's tables, and saves nothing. Iterate with it. Read `missingRequired`, the error and the trace;
  `extraFields` are source fields the model does not declare, which are kept, not an error.
- `save_connector_code` stores the code. Pass `instructions`: a plain description of the source and of what the code
  does, including filters and re-run behaviour. It replaces the stored description, so send the whole of it every
  time. A later rebuild works from it.
- `set_connector_date_roles` next: which fields hold the source's creation date and last-change date. Saving code does
  not work these out, and without them event dates fall back to a guess.
- `build_connector` makes the platform's AI write the code from `instructions` instead. Use it only when you cannot
  write and test code yourself: it is slower and it bills the organisation.

The rules that matter most in the code: authenticate only from `ctx.credentials`, and keep secrets out of the code,
log lines and returned rows (a stored credential value in a tool result shows as `[credential <field> hidden]`);
return every field the source has, not just the model's; give each row a stable `id` taken from the source's natural
key; honour `ctx.limit`, where null means everything. The brief has the rest.

## When to ask

Keep going without asking for anything the user asked you to do. Stop and ask first when:

- **a connector would write to another system** (create records, send messages). That makes it an `actuator`: say
  so, name the system, and get a yes before `create_connector`. Running it performs those actions for real, whether
  through `ingest_connector` or a test with `confirmActions`, so say what the first run will do and get a yes first.
- **scheduling or waking an actuator**, since its actions then happen with nobody watching.
- **deleting or rebuilding**: `remove_connector`; `clear_table` on a table holding data you did not load yourself, or
  on an actuator's table, whose rows are the record of the actions it took, so its next run may take them again (the
  tool refuses that without `confirmActions`); `reload_model` with `rebuild: "full"`; or `reload_model` answering
  `needsConfirmation` because stored values would be lost.
- **credentials are needed.** Ask for exactly the fields the source needs and never invent them. Store them with
  `set_connector_credentials` and never repeat them back. When another connector already holds them,
  `list_connector_credentials` and `copy_connector_credentials` reuse them without anyone typing them again.
- **how a child row links to its parent is unclear.** "Match them up roughly" is not an answer: a reference that does
  not equal a parent id exactly keeps the row out of its parent's case, and nothing reports it.

## Checking the result

A connector is done when the cases are right.

- `list_table_rows`: the rows are there, the fields are filled, the values look like the source.
- `list_cases`: one case per row of the workflow's root table, with how far each has come. `count` is the total, and
  the list comes in pages; `nextOffset` fetches the next one. Pass `summary: true` to get only each case's id and
  progress; `get_case_details` takes it too and leaves out source records and event payloads.
- `find_case` also matches rows of other tables and returns the case they belong to, naming the row in `matchedVia`.
- Child rows should sit inside their parent's case: open a few cases with `get_case_details` and check that the child
  events are there and belong to that parent. A child whose linking field does not match a parent id exactly joins
  no parent's case, and nothing reports it.
- `get_case_details` on a few cases: the events that fired should match each row's state. An order that is only
  placed must not have a "shipped" event.
- When events fire that a row's state does not justify, or two sibling events need telling apart, compile trigger
  rules with `build_trigger_rules`, check each with `preview_trigger_rule` and read its code with
  `view_trigger_rules`, then run `rebuild_events`. Ingesting again only adds events; it never removes wrong ones.
- When rows landed wrong (wrong ids, a child linked to the wrong parent), fix the code, `clear_table`, and ingest
  again. Ingesting again alone leaves the wrong rows in place.
- When an ingest reports rows as updated on every run although nothing changed at the source, the stored value
  differs from what the code returns (rounding, formatting, types). Compare a row from `list_table_rows` with your
  test output and tell the user if the platform changed the value.
- The case tools (`list_cases`, `find_case`, `get_case_details`, `get_event_log`) group events by the workflow's own
  case unless you pass `caseType`, and that default view is the one to check connectors in. Another entity as
  `caseType` regroups the same events around that object, one case per order for instance: an event can then sit in
  several cases, and events no case of that type holds are left out. Use the same `caseType` in every call of one
  check, or the answers will not line up.

## When the model is wrong

Data sometimes shows the model is missing something: a field the source has and the events need, a status value the
model does not allow, an event that never fires because nothing in the rows can show it. Do not bend a connector to
hide that. Tell the user what you found, and once they agree:

1. Change the model in the Qlerify modeller through the Qlerify MCP tools (the `qlerify` server, with the
   `mcp-companion` plugin's skills). `list_workflows` in Live gives each workflow's `modelLink`, and
   `https://app.qlerify.com/workflow/<projectId>/<workflowId>` names the modeller workflow to edit.
2. `reload_model` in Live. It reconciles tables in place. If it answers `needsConfirmation`, tell the user which
   stored values would be lost before calling it again with `confirm: true`.
3. Update the connectors the change touches and ingest again. `reload_model` already works the events out again when
   the change affects them. If acceptance criteria behind trigger rules changed, compile those rules again with
   `build_trigger_rules`, then run `rebuild_events`.

## Tool errors

- "did not finish within 240s and may still complete": the work may still be running. Check `get_connector_history`
  or `list_table_rows` before running it again, or it may happen twice.
- `[credential <field> hidden]` in a result: a stored credential's value was there. Never write the placeholder into
  code: read the value from `ctx.credentials.<field>`, and stop logging or returning it.
- "no connector": use the `adapterId` that `create_connector` returned. The optional `id` you pass it is only a short
  name.

## The detailed rules

Read `references/connector-rules.md` before the first connector of a session. It covers in depth the connector types,
the questions to settle before building, linking rows into cases, re-run behaviour, demo data, value objects, trigger
rules, reading other tables and the event log, AI inside a connector, and keeping connectors running.
