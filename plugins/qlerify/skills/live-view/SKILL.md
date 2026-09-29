---
name: live-view
description: >-
  Turn a business process and the data behind it into a live view in Qlerify Live, from one request: model the
  workflow in the Qlerify modeler, load it into Qlerify Live, build and run the connectors that fill it, then answer
  with figures such as the median lead time and give a link to the dashboard. Use it when the user asks for a "live
  view", a "dashboard" or "KPIs" of a business process and its data, "lead time" or "cycle time" of a process, "track
  this process with our data", "use the open data", or "show the workflow with real data", also when they do not
  name Qlerify or Live. It needs the Qlerify Live MCP tools, and the Qlerify modeler MCP tools to build a new view.
allowed-tools: Read, WebFetch, WebSearch, mcp__qlerify-live__*, mcp__qlerify__*
---

# Building a live view

A live view is a Qlerify Live workflow whose tables connectors keep filled from the real sources. Live works out the
domain events from the rows and groups them into cases, so the dashboard and the figures come from the data, not
from a model alone. The user gets three things at the end: the model in the modeler, the workflow running in Live,
and an answer with a link.

This skill runs the whole path and hands the detailed work to two others: `workflow-creation` for the model and
`connector-building` for the connectors. Follow their rules inside each step.

## Before you start

- **A process already in Live needs only step 7.** Call `list_workflows` first. When the user's process already
  runs there with data and they want a figure, go straight to step 7 with the Live tools alone.
- **Both tool sets are needed to build a view.** The modeler tools (`create_workflow`, `create_domain_events` and
  the rest, from the `qlerify` server) and the Live tools (`list_workflows`, `create_workflow`, `get_lead_time`,
  from the `qlerify-live` server). Both servers have a `create_workflow`: the modeler's makes an empty model, Live's
  loads a finished model into Live. If a server you need is missing, stop and give the user the setup: the
  modeler's is `claude mcp add --transport http qlerify https://mcp.qlerify.com --header "x-api-key: <key>"` with
  the key from the modeler's account settings; Live's is the command on Live's Organization admin → MCP tab.
- **Keep a run log.** Write `live-view.md` in the working folder and update it after every step: the modeler
  workflow's link, the Live workflow's id and url, each table with its connector and whether it is filled, the
  questions still open. When a run is picked up again, read it first and carry on from where it stopped.

## 1. Settle the scope in one question

Ask once, and only what you cannot find out yourself:

- where the records live: which systems, or which open data;
- what one case is (one order, one proposition, one project) and when a case is done;
- the time window, if the request names one. Live measures the last 24 hours, 7 days, 30 days, 3 months or 12
  months, counted back from now;
- how often the view should refresh (daily suits a public register, hourly a business system);
- for each closed system, whether a read-only login exists. Credentials are entered later in Live's form, never here.

Skip anything the request already answers. Then work without stopping, except for the points in "When the user
must act".

## 2. Look at the data before modeling

The model has to match the records, so read the source first.

- **Open data or a public API:** read its documentation and fetch a few records with `WebFetch`.
- **A closed system:** ask for its schema, an export or a few sample rows. Do not ask for credentials here.

Write down, per record type: its key, which field holds its parent's id, its status values in their real order,
and which field dates each step. Look for traps: a date filter that looks right but filters on something else (the
day a document was submitted, when the user asked for activity), dates in the future, placeholder records, records
that link to several parents, and records that often have no parent at all.

## 3. Model it in the modeler

Follow `workflow-creation`, and its `references/live-readiness.md` above all: that is what makes the cases come out
right in Live. When the case follows a state machine and `workflow-creation` asks for approval of the state map
(Phase S), still write the map, but do not wait for approval: take the states and their order from the source's
real status values, note the map in `live-view.md`, and carry on. The user reviews the model in the modeler at the
end.

Live fetches the model with the Live organization's own modeler key, so the project must have that key's account
as a member. Prefer a project that a `modelLink` in Live's `list_workflows` already names. Do not ask about the
project up front: step 4's check tells you whether Live can fetch the model. Finish with `validate_domain_model`.

## 4. Load it into Live

1. `create_workflow` on the Live server with the modeler workflow's link and `dryRun: true`. It fetches the model
   and reports the case root, the tables that cannot reach it and any problem that would stop the load. Fix those in
   the modeler and check again.
2. `create_workflow` without `dryRun`. It returns the Live `workflowId` and `url`.

If Live says a workflow already follows this modeler workflow, use that one and call `reload_model` after changing
the model. If Live cannot fetch the model, that is the modeler-access point in "When the user must act".

## 5. Build and fill the connectors

Follow `connector-building` for every table, parents before children. Ask for all credentials in one go: call
`request_connector_credentials` for each connector that needs them, give the user every form link in one message,
and wait once. Before the full ingest, set each connector's date roles, including `events`, which maps every step
the source dates on its own to that field: events are dated when they are first worked out, so dates set later need
`rebuild_events`. Ingest each table in full: pass `ingest_connector` a `limit` above the table's size, since without
one it lands only a first batch. A connector that checks its own table and would not finish in one run's time is
ingested in batches instead (`connector-building`, connector-rules section 12). Check the cases as that skill
describes before moving on.

## 6. Keep it live

Schedule every connector at the agreed refresh with `set_connector_schedule`, parents first, and offer wake-ups
where the model declares them. A live view without schedules goes stale.

## 7. Answer, with a link

Call `get_lead_time`. For a window, pass `within` (24h, 7d, 30d, 3mo or 12mo, counted back from now) and
`windowBy` when the window is about when cases started; for any other period, use the closest window and say in the
answer which one the figure covers. Use `fromEvent` and `toEvent` when the user means a specific span, such as
submitted to decided.

Check `doneRule` against what the user said a case being done means. If they differ, pass the user's closing event
as `toEvent`, and tell the user they can set the rule in Live's Overview under "Done means…", which no tool can
change.

Pass `caseType` when the figure is about another kind of record than the workflow's root. When the user counts two
kinds of record (propositions and motions), call `get_lead_time` once per kind and give each median with its own
basis. In the root's view, a record that links to no root row (a motion with no proposition) counts as a case that
never finishes, so do not report those as unfinished work.

Answer with:

- the median, and the P85 when it helps;
- its basis: how many cases were measured, the window, what counts as done (`doneRule`), and what was left out
  (no date, not finished yet, estimated dates);
- `reportsUrl` on a line of its own. It opens the Reports page with the same window and case type; when
  `reportsShowsThisFigure` is false, say the page shows the default figure rather than this one.

If many measured cases rely on estimated dates, say the figure is approximate and name the table that lacks a real
date. Do not work a median out by hand from single cases.

## When the user must act

Some steps need a person. Gather them so the user is interrupted as rarely as possible, and say exactly what to do:

- **credentials**, entered in Live's form from the links `request_connector_credentials` returns;
- **access to the modeler project**: when Live cannot fetch the model, the account behind the key in Live's
  Organization admin → General must be added to the modeler project, or a key from an account that is a member saved
  there;
- **organization admin rights** in Live, which creating workflows and connectors needs.
