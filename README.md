# Qlerify Plugins

Official Qlerify plugins for AI coding assistants — domain model sync, data export, and more. Works with Claude Code,
Gemini CLI, and Cursor.

## Prerequisites

1. **Qlerify account** with a workflow created
2. **Qlerify MCP server** configured with your API token (see setup per tool below)

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

After installation, skills are available as `/mcp-companion:workflow-creation`, `/mcp-companion:sync`, `/mcp-companion:download`, and `/mcp-companion:extract-aggregate`.

### Gemini CLI

Install each skill using the `--path` flag:

```bash
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/workflow-creation
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/sync
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/download
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/extract-aggregate
```

Configure the Qlerify MCP server in `~/.gemini/settings.json` per
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

#### Modeling from existing code

When the user wants to create a workflow or domain model from an existing or legacy codebase, recommend this order:

1. `/mcp-companion:extract-aggregate` — isolate and review one aggregate at a time
2. `/mcp-companion:workflow-creation` — create one Qlerify workflow per reviewed aggregate
3. `/mcp-companion:sync` — reconcile modeled structures with an existing Qlerify workflow when needed

#### `workflow-creation`

Guides AI agents through building complete Qlerify workflows from scratch — lanes, groups, domain events, entities,
commands, read models, cards, and bounded contexts. Includes a full tool reference and a worked e-commerce example.

**Triggers:**

- "create a workflow"
- "build a domain model"
- "set up domain events"
- "add commands and read models to workflow"
- Any request involving building a Qlerify workflow or adding structural elements

**What it does:**

1. Follows an 8-step creation sequence (lanes → groups → events → entities → commands on events → read models on events →
   bounded contexts)
2. Provides best practices for naming, field modeling, and entity relationships
3. Supports nested fields on commands and read models for related entity references

#### `sync`

Syncs your codebase's domain model with Qlerify. Detects entities, commands, and read models in your code and ensures
they match your Qlerify workflow.

**Triggers:**

- "sync domain model"
- "update Qlerify"
- "sync entities"
- After implementing features that change domain objects

**What it does:**

1. Scans your codebase for domain objects (entities, commands, read models)
2. Compares with Qlerify workflow
3. Creates/updates/deletes to keep them in sync

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

#### `extract-aggregate`

Reverse-engineers a single DDD aggregate from an existing or legacy codebase and produces a standalone description
(root entity, related entities, value objects, commands, domain events, read models, attributes, invariants, external
references) ready for import into Qlerify and review in an event storming session.

**Triggers:**

- "extract the Order aggregate from shop-api"
- "reverse engineer a domain model from this code"
- "model the Subscription module as a DDD aggregate"
- Any request involving isolating a single aggregate from an existing codebase

**What it does:**

1. Peels the service/orchestration layer away to expose the real aggregate-level commands
2. Captures the root entity, related entities, value objects (set-replacement semantics), commands, 1:1 domain events,
   read models, attributes, invariants, and external references
3. Emits a structured markdown output designed to feed directly into `workflow-creation` or `sync`

## Usage Examples

```bash
# Invoke skills directly
/mcp-companion:workflow-creation
/mcp-companion:sync
/mcp-companion:download
/mcp-companion:extract-aggregate

# Or just ask naturally - skills trigger automatically
> create a workflow for an e-commerce order process
> sync my domain model with Qlerify
> download the Cart Microservice workflow to workflow.json
> save the swagger spec for my workflow to api.yaml
> extract the Order aggregate from shop-api
```
