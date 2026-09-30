# Models that run in Qlerify Live

Qlerify Live runs a modeler workflow on real data. Every entity becomes a table that a connector fills, and Live
works out the domain events from the rows and groups them into cases. A model that reads well on the board can still
give empty or wrong cases in Live. Apply these rules whenever the model will run in Live.

## Cases

- **The first event's aggregate root is one case.** Live groups by it unless a reader picks another type. Root the
  first event on the thing the user counts: one order, one support ticket, one project.
- **A child holds its parent's id.** Every entity that has events of its own needs a field holding its parent's id:
  `<parent>Id` as a string, or a field with `relatedEntity` pointing at the parent. Live follows links from the
  child to the parent only, so `Order.items` on the parent does not bring the items into the order's case, while
  `OrderItem.orderId` does. When a record belongs to several parents, make the field one-to-many and fill it with
  every parent's id.
- **A record with its own source table and its own events is its own aggregate root**, not a part of another one:
  a time entry, a payment, a shipment. Live works out events only for tables that an event is rooted on.
- **A part-of table with no events** (invoice lines) needs its root's id only when something must join the two,
  such as a figure over the lines.
- **A record that often has no parent is a second case type.** A return with no order never joins an order's case.
  In the default view it counts as an open case that never finishes. With `caseType` set to its own entity, a case
  holds only the steps on that record and its children, so when its later steps sit on a parent, its lead time is
  read in the parent's view, from its own first step (`fromEvent`).

## Status and steps

- **List every status value, in order.** For an entity whose events follow a status field, name the field exactly
  `status` and give every value in its `exampleData`, in lifecycle order, not three samples. Use the same words in
  the event names (Order Shipped for `SHIPPED`). A row whose status is not in the list counts as having reached
  none of the status steps.
- **Alternative outcomes are not steps in that order.** Live reads the status list as a ladder: a row at a later
  value counts as having passed every earlier step. With `APPROVED, REJECTED`, every rejected row also gets
  Application Approved. Put the shared path first and the alternative outcomes last, and model each outcome as a
  branch of a decision whose command adds a field of its own (`approvedAt`, `rejectedAt`) rather than one shared
  command. Live then fires each branch from its own field instead of the status, and that field can date the step.
  An outcome that can happen at any point (withdrawn) still makes every earlier step fire, so in Live it needs
  trigger rules (`build_trigger_rules`).
- **Every end event finishes the case.** Live counts a case done once any event that nothing follows has fired,
  unless the rule is changed: "Done means…" in Live's Overview, or `set_done_rule`. Lead a side branch
  that is not an ending (reminders, notifications) back into the main flow, so the only events with nothing after
  them are real endings. A step that only some cases reach after the real ending (a follow-up sent only when the
  customer must act) must not be the last step, or the cases that skip it never finish: end the flow at the step
  every case reaches and leave the optional one out.
- **Give each timed step a date field** on its entity (`submittedAt`, `decidedAt`), holding an ISO date as a string:
  the modeler has no date type, and the step's own command sets it. In Live, the connector maps each step's event to
  its field (`set_connector_date_roles` with `events`, one field per step): the step is dated from it, and a filled
  date is enough for the step to count. Without a date field, Live dates the step with the row's creation or
  last-change date and marks it estimated, and lead times come out approximate or near zero.
- **Keep a step's command to what that step sets.** A step without a mapped date, and every alternative outcome,
  counts only when every field its command adds is filled, and a yes/no field at false counts as empty. An optional
  field on the command (a flag, a text that can be blank) then stops the step for those rows, so keep such fields
  on the entity only.
- **Write acceptance criteria for every event.** At least one Given/When/Then whose Then names the data change that
  shows the step happened: a field filled, a status value. Live gives them to the code that fills the tables and to
  the rules that tell events apart.

## Names

- **Field names:** English letters, digits and `_` only, in camelCase, not starting with a digit: no å, ä, ö,
  spaces or hyphens. Live makes each field a database column and refuses other names.
- **Entity names:** at most 24 characters once the spaces are gone (`Purchase Order Line` becomes
  `PurchaseOrderLine`, 17), not starting with a digit. Live refuses longer ones, since the database would cut the
  table name short.
- **Entity and event names in English letters.** The modeler drops letters such as å, ä and ö from the keys Live
  uses, so the table gets a different name than the entity and a child's `<entity>Id` field no longer matches it.
- **One name per entity across bounded contexts.** Two systems that both hold customers get `ErpCustomer` and
  `CrmCustomer`, not two `Customer` entities.

## Checking before the load

`validate_domain_model` does not check any of this. In Live, `create_workflow` with `dryRun: true` fetches the model
and reports the case root, the tables that cannot reach it, the done rule (`doneRule`: check that every way a case
can end is in it), and the problems that would stop the load. When the case root is the wrong entity, `rootAggregate`
on `create_workflow` picks another one.
