# Connector rules

The rules Qlerify Live's own connector assistant follows, written for an agent working through the MCP server.

## Contents

1. Connector types
2. Before building: what to settle
3. Linking rows into cases
4. Re-runs: incremental, delta, regenerate
5. Demo data
6. Value objects
7. Which events fire, and trigger rules
8. Reading other tables and the event log
9. AI inside a connector
10. Keeping connectors running
11. Knowing the source
12. Limits

## 1. Connector types

`create_connector` takes a `behavior`, which tells the platform where the rows come from:

- **`sync`** (the default): the rows exist in a source of record and the connector mirrors them. The platform can
  then report a row that stops coming back. Re-running is free.
- **`generator`**: the rows exist nowhere until the connector computes them (an AI call or a paid API per row). Nothing
  can go missing, so disappearance is never reported. Re-running costs money.
- **`actuator`**: the connector performs an action in another system (creates a record, sends a message) and then
  lands the result. Re-running changes the outside world. Model rebuilds skip it, and the dry run and field discovery
  refuse it unless `confirmActions` is set.
- **`extractor`**: the connector reads an unstructured source (a document, a sheet) and interprets it with AI.
  Re-running is safe but pays for the extraction again.

Cost does not decide the type. A connector that mirrors a source is `sync` even when it pays for AI to derive one
column; typing it `generator` silently gives up the check for deleted rows.

Work the type out from three plain questions rather than asking the user for it in these words:

1. Does it read an external system, or only data already in this workflow?
2. Does it perform actions anywhere else (update a system, send a message)?
3. Do its rows already exist in the source, or does the connector bring them into existence?

Actions anywhere mean `actuator`. Say so and get a yes before creating it ("this will create records in HubSpot, not
just read them, correct?"). Never decide it silently: a connector that writes while typed `sync` is re-run by model
rebuilds, and each re-run repeats its real-world actions.

Pass `targetSystem` when you can tell which product the connector talks to ("Slack", "HubSpot") and the bounded
context is not already its name. Every warning names that system, and there is no screen where the user can set it.

The type can change later with `set_connector_behavior`, and nothing reclassifies a connector on its own. When your
code starts writing where it only read before, retype it in the same step and say so. Retyping does not rewrite the
connector's description, so follow it with `update_connector_description`; saving or building code already does that.

## 2. Before building: what to settle

Ask only what the conversation has not answered, and for all tables at once rather than table by table.

1. **Source.** An external system (then its credentials), a table already in this workflow (read at run time with
   `ctx.readTable`), or no source at all (demo data, section 5).
2. **Trigger and cadence.** A connector never runs just once, so ask whether it should poll on a schedule and how
   stale the data may get. "Manual only" is a valid answer, and demo data needs no schedule. A calendar condition
   ("on the last day of the month", "when a quarter starts") is not a schedule setting: the schedule is only the
   polling clock. Put the condition in the code, which returns nothing when there is no work; stable ids make the
   extra runs free.
3. **What a run does.** The three questions in section 1, plus, for computed content, the re-run question in
   section 4.
4. **Case linkage.** Section 3.
5. **Which events fire.** This is not a choice: after every run the platform works out the events the landed rows
   justify. Compile trigger rules only when the user states per-event conditions or your checks show wrong events
   (section 7).

## 3. Linking rows into cases

Skip this for a table no event is rooted on: its rows are reference data and join no case.

For every other table that is not the workflow's root, each row must reach its case, and the platform matches by
exact id equality: the child's linking field must hold the parent row's `id`, byte for byte, in the format the parent
table stores. Establish which source field carries it, and keep asking until the answer is concrete: a source field,
a key you can derive, or a lookup your code performs against the parent table with `ctx.readTable`. A reference that
does not match does not fail; it silently starts a one-row case of its own. Write the agreed linkage into the
`instructions` you save.

`get_connector_brief` lists the linking fields the model declares, with real ids from the parent tables. For a
connector triggered by workflow events, copy the linkage off the event, which carries `aggregateId` and `caseId`. For
demo data, section 5 settles linkage.

Period-scoped tables (one row per subject per period, such as a Quarter) have their own rules in the brief: the
platform composes those row ids itself.

## 4. Re-runs: incremental, delta, regenerate

Every manual pull re-runs a connector, and it may be scheduled. The platform lands rows by id: an id already in the
table has its changed fields updated in place, a null never erases a stored value, unchanged rows are skipped, and
nothing is duplicated. So a source record that changes later (an order marked shipped) updates on the next pull and
moves its case forward.

- **Delta** is the right choice whenever the source can answer "what changed since": return `{ rows, cursor }` and
  read only changes after `ctx.cursor` on a delta pull. The brief explains it.
- **Computed content** (an AI call or a paid API per row) needs the user's choice before building:
    - **incremental** (recommend it, and use it when they have no preference): read this connector's own table with
      `ctx.readTable` and process only source items that have no row yet. A run that finds nothing new returns an empty
      list, and that is the gate working, not an error to repair;
    - **regenerate everything** each run: changed values land, but every row's paid work is done again every run.
- **A cycle-linked target** (rows that belong to a Quarter or similar) needs one more answer: is an item per subject
  forever, or per subject per period? Per period means the period goes into the row id, so each new period creates
  fresh rows instead of overwriting the last period's.
- A plain pass-through pull from a system of record needs none of this, and neither does demo data.

Write the chosen behaviour into the `instructions` you save with the code.

## 5. Demo data

When the user asks for demo, example, sample or simulated data, build the same way with two differences: skip
credentials, and the code fabricates realistic rows in memory with no network. The goal is not twenty look-alike rows
but a table that reads as about twenty cases spread evenly along the workflow, because the platform works out each
row's events from its state.

1. **Find the lifecycle.** From the workflow definition, list in order the events whose aggregate root is the target
   table. If there are none, the table is reference data: fabricate plain, varied rows and skip steps 2 and 3. The
   first event creates the row; each later one fills the fields its command introduces and moves `status` along its
   allowed values, in the order the model lists them.
2. **Spread the rows across states.** For the workflow's first table, make about twenty rows split evenly across the
   lifecycle states (with four states, five rows each; any remainder goes to the earliest states), or the count the
   user gives, and ingest with a `limit` that covers it. A row at state N must look like it stopped right after event
   N: the fields introduced by events 1 to N filled with realistic, varied values, the fields of later events empty,
   and `status` set to the value event N leads to. A field the model limits to a set of values (shown in the brief as
   allowed values) takes only those values, never a variant.
3. **Give downstream tables real parents.** A table that is not the root of the first event must reference parent
   rows that exist. Check the parent tables with `list_table_rows` first. If they are empty, do not invent ids: fill
   the parents first. Once they are filled, use their real ids and stay consistent with their state, attaching rows
   only to parents that have already passed the event where this table first appears. The parents also set the count:
   one row per eligible parent, and each child's state follows how far its parent's case has come. Fewer eligible
   parents means fewer rows, which is correct. Never add several same-stage children to one parent unless the model or
   the user says the relationship is one-to-many.

Stable, deterministic ids make re-running demo connectors harmless.

## 6. Value objects

A value object can be filled two ways, and the user chooses: as its own table (`create_connector` targeting the value
object) or embedded as a JSON value on a parent entity's field (the parent's connector returns it as a nested object).
Ask which, unless the conversation makes it obvious.

## 7. Which events fire, and trigger rules

By default the platform works out events from each row's state with general rules (creation, the status sequence,
which fields are filled). Those cannot express conditions such as "fire Upsell Deal Created for upsell deals and Cross
Sell Deal Created for cross-sell deals". When the user states such conditions, when one table drives sibling events
that need telling apart, or when your checks show events a row's state does not justify:

1. Restate each event with its condition.
2. `build_trigger_rules` with the whole family of related events in one call, so sibling conditions stay consistent.
3. `preview_trigger_rule` for each event: read the evidence per row and check how many rows fire against what was
   meant.
4. If a preview is wrong or reports an error, call `build_trigger_rules` again for that event with an `errorReport`
   saying exactly what fired that should not have, or the other way round.
5. Only then ingest, or run `rebuild_events` when rows are already in.

The conditions belong to the events, and the model's Given/When/Then is their permanent home; the rule is compiled
from them. Do not filter events inside the fetch code instead, and do not create rules nobody asked for. A rule can
also name the row's own date column that best dates its event (a "referred date" for a "Motion Referred" step),
which gives a more precise time than the general created or last-modified dates.

## 8. Reading other tables and the event log

- When a connector works from data already in the workflow (a list of companies from another table), read it at run
  time with `ctx.readTable("<Table>")`. Never paste a copy of another table's rows into the code: it goes stale the
  moment that table changes.
- When a connector should react to something that happened in the workflow ("when a case reaches step X", "for each
  approved order"), read the event log with `ctx.readEvents` and name the triggering events in the `instructions`.
  Decide "already handled" only from this connector's own rows: the event log is rebuilt with fresh ids whenever the
  model changes, so an event id or timestamp used as a watermark would re-fire everything.

## 9. AI inside a connector

A connector can ask the organisation's AI provider at run time with `ctx.ai(prompt, { json: true })`. The platform
supplies the provider and key; the code never holds one. Use it for judgements code cannot make (who claimed a card
in its comments, classifying free text), never to parse structured fields. Every call is billed to the organisation,
so batch independent items into one prompt, ask for JSON with exact keys, send only the fields needed, and combine it
with the incremental gate (section 4) so landed rows are not asked about again. A run allows 50 calls of up to 8,192
output tokens each; a prompt containing one of the connector's credential values is refused.

## 10. Keeping connectors running

**Schedules.** If the user mentioned a cadence at any point ("every 6 hours", "nightly", "keep it in sync"), act on
it before finishing: convert it to minutes (6 hours is 360, daily is 1440, the minimum is 5) and call
`set_connector_schedule`. Pass `startAt` whenever the time of day matters or connectors must run in order; without
it, the clock counts from each connector's last run. If you cannot honour the cadence, say so. Never let it pass
silently.

**Wake-ups.** `get_adapter_config` returns `wakeSources`: the connectors the model says this one waits for (declared
in the modeller by giving an event a domain-event schema in a read-model slot). When the list is not empty, offer
`set_connector_wake` after scheduling, so the connector runs as soon as its upstream one lands data; the schedule
stays as the fallback. This replaces staggering start times by hand, which breaks the day an upstream run takes
longer. Never invent a dependency the model does not declare.

**Notifications.** `set_connector_notify` posts a message when a run lands new rows, changes rows, fails, finishes
badly, or when the connector stops running. Offer it when an acceptance criterion on the connector's table describes
telling someone outside the system, and quote that criterion; it replaces modelling the notification as its own
event, entity and connector. It cannot set the webhook URL, which is a credential entered in the connector's
Notifications panel in Live; with no destination, say so and stop. Every pull that lands rows posts, manual ones
included. Express conditions ("only deals over 50k") as `where` clauses and the message line as a `line` template
(`{firstName} {lastName}`, `{_raw.plan}` for a source field, `{createdAt:date}` for a readable date), then check the
exact message with `preview_connector_notification`. Leave `problems` and `stopped` on unless the user says otherwise.

## 11. Knowing the source

When you are unsure of a source API's endpoints, fields, authentication or pagination, read the vendor's public
documentation before writing code, and use real field names rather than guesses. After the first successful test,
`discover_source_fields` records the source's actual field shape on the connector, and later briefs include it.

## 12. Limits

- A single pull returns at most 10,000 rows, even with no limit. A run that returns exactly 10,000 means more are
  waiting: run it again.
- `ctx.readTable` gives at most 10,000 rows per table. A connector that checks its own table for work already done
  cannot see past that, so filter at the source as well once a table grows that large.
- A run is killed at its time budget and loses every row it produced; run independent AI batches at the same time
  rather than one after another.
