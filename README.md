# Qlerify Plugins

Official Qlerify plugins for AI coding assistants — domain model sync, data export, and more. Works with Claude Code,
Gemini CLI, and Cursor.

## Prerequisites

1. **Qlerify account** with a workflow created
2. **Qlerify MCP server** configured with your API token (see setup per tool below)
3. For `qlerify-live-companion`: a **Qlerify Live** organization you administer, and its MCP server configured with a
   Live token (see below)

## Installation

### Claude Code

Configure MCP server in `~/.claude.json`:

```json
{
  "mcpServers": {
    "qlerify": {
      "type": "url",
      "url": "https://mcp.qlerify.com",
      "headers": {
        "x-api-key": "YOUR_API_TOKEN"
      }
    }
  }
}
```

Install the plugin:

```bash
/plugin marketplace add qlerify/qlerify-plugins
/plugin install mcp-companion@qlerify-plugins
```

After installation, skills are available as `/mcp-companion:workflow-creation`, `/mcp-companion:code-generation`, `/mcp-companion:sync`, and `/mcp-companion:download`.

#### Qlerify Live

In Qlerify Live, open **Organization admin**, then the **MCP** tab, create a token and run the command it shows. It
looks like this, with your Live address and token:

```bash
claude mcp add --transport http qlerify-live https://YOUR_LIVE_DOMAIN/mcp --header "x-api-key: YOUR_LIVE_TOKEN"
```

Then install the plugin:

```bash
/plugin install qlerify-live-companion@qlerify-plugins
```

The skill is available as `/qlerify-live-companion:connector-building`.

### Gemini CLI

Install each skill using the `--path` flag:

```bash
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/workflow-creation
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/code-generation
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/sync
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/download
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/qlerify-live-companion/skills/connector-building
```

Configure the Qlerify MCP server (and the Qlerify Live one, for `connector-building`) in `~/.gemini/settings.json` per
[Gemini CLI docs](https://geminicli.com/docs/cli/mcp/).

### Cursor

1. Open Cursor Settings (`Cmd+Shift+J`)
2. Go to **Rules** > **Add Rule** > **Remote Rule (GitHub)**
3. Enter: `https://github.com/qlerify/qlerify-plugins`

Configure the Qlerify MCP server in Cursor's MCP settings.

Get your API token from the Qlerify UI.

## Plugins

### `mcp-companion`

Teaches AI agents how to effectively use Qlerify's MCP server. Contains the following skills:

#### `workflow-creation`

Guides AI agents through building complete Qlerify workflows — lanes, domain events, entities, value objects,
commands, read models, and bounded contexts. Also supports reverse-engineering existing or legacy codebases into
DDD aggregates with visualized life cycles.

**Triggers:**

- "create a workflow"
- "build a domain model"
- "set up domain events"
- "add commands and read models to workflow"
- "extract the Order aggregate from shop-api"
- "reverse engineer a domain model from this code"
- "model the Subscription module as a DDD aggregate"
- Any request involving building a Qlerify workflow, adding structural elements, or modeling from existing code

**What it does:**

1. Builds the workflow in dependency order: events → bounded contexts → entities → commands → read models → schemas → validation
2. Reverse-engineers existing codebases into DDD aggregates (root entity, value objects, commands, events, read models, invariants)
3. Applies naming and modeling best practices, including nested fields for related entity references
4. Validates the result and reconciles it against the source code at the end

#### `code-generation`

Generates production-ready code from a Qlerify domain model. Pairs with `workflow-creation` (which produces the model)
and `sync` (which keeps model and code aligned over time).

**Triggers:**

- "generate code from the model"
- "implement the workflow"
- "scaffold from Qlerify"
- "build the aggregate"
- "code up the domain model"
- Any request to produce runnable code from a Qlerify workflow

**What it does:**

1. Pre-flight model scan — flags structural blockers before generating anything (missing aggregate roots, contradictory GWTs, misclassified value objects)
2. Platform selection — matches an existing project's stack, or picks an opinionated default for greenfield
3. Maps the model to a persistence design — aggregate boundaries, value-object storage, optimistic locking, cross-bounded-context references
4. Generates entities, invariant guards, command handlers, read-model queries, in-process event bus, and GWT-derived tests
5. Iterates the test suite until green, escalating to the user only after repeated failures
6. Writes a `.qlerify/codegen.json` anchor so future runs can apply model deltas as targeted patches

#### `sync`

Keeps your codebase and its Qlerify workflow in agreement, in both directions. Detects drift on either side and routes
each direction: code-ahead changes are written into Qlerify directly, while model-ahead changes are detected and handed
off to `code-generation` to apply to the code.

**Triggers:**

- "sync domain model"
- "update Qlerify"
- "sync entities"
- After implementing features that change domain objects
- After editing the model on the Qlerify board and needing the code to catch up

**What it does:**

1. Establishes a baseline using the `.qlerify/codegen.json` anchor (`modelHash`) to tell which side drifted; writes an anchor on first run if there isn't one
2. Scans the codebase for domain objects (entities, value objects, commands, read models, invariants)
3. Classifies each difference as code-ahead, model-ahead, or a conflict
4. Applies code-ahead changes to Qlerify (deletions confirmed first); hands model-ahead changes to `code-generation`
5. Validates the model and updates the anchor

#### `download`

Fast download any Qlerify data directly to files. Uses `curl + jq` to bypass AI processing, making it ~100x faster than
standard MCP tools for large data.

**Triggers:**

- "save to file"
- "download workflow"
- "export"
- Any request to save Qlerify data locally

**What it does:**

1. Fetches data directly via shell commands
2. Pipes to file without AI processing
3. ~1 second instead of 3-5 minutes for large workflows

### `qlerify-live-companion`

Teaches AI agents how to build the connectors that fill a Qlerify Live workflow through the Qlerify Live MCP server.
Contains one skill:

#### `connector-building`

Builds, tests and fixes connectors for one table or a whole workflow in one session: the agent writes each
connector's code itself, tests it in Live's sandbox, ingests the data and checks that the resulting cases and events
are right. When the data shows the model is wrong, it changes the model in the modeller (with `mcp-companion`) and
reloads it in Live.

**Triggers:**

- "build the connectors for this workflow"
- "fill the Order table from our Postgres database"
- "simulate demo data for all tables"
- "keep it in sync every hour"
- "why are the cases wrong / fix this connector"
- Any request to get data into Qlerify Live

**What it does:**

1. Reads the workflow's model and plans the build order, parents before the tables that link to them
2. Settles sources, credentials, cadence and actions for all tables at once
3. Per table: creates the connector, writes and tests its code, saves it, dry-runs and ingests
4. Checks the cases and events the rows produce, and fixes wrong events with trigger rules
5. Updates the model in the modeller and reloads it in Live when the data proves it wrong
6. Sets up schedules, wake-ups and notifications

## Usage Examples

```bash
# Invoke skills directly
/mcp-companion:workflow-creation
/mcp-companion:code-generation
/mcp-companion:sync
/mcp-companion:download
/qlerify-live-companion:connector-building

# Or just ask naturally - skills trigger automatically
> create a workflow for an e-commerce order process
> sync my domain model with Qlerify
> download the Cart Microservice workflow to workflow.json
> extract the Order aggregate from shop-api and build a workflow
> generate code from the Cart workflow
> build connectors for all tables in the Order Fulfilment workflow in Qlerify Live
> fill the Customer table in Live with demo data and keep it in sync nightly
```
