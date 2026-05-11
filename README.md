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

After installation, skills are available as `/mcp-companion:workflow-creation`, `/mcp-companion:sync`, and `/mcp-companion:download`.

### Gemini CLI

Install each skill using the `--path` flag:

```bash
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/workflow-creation
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/sync
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/download
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

#### `workflow-creation`

Guides AI agents through building complete Qlerify workflows — lanes, domain events, entities, value objects,
commands, read models, and bounded contexts. Also supports reverse-engineering existing or legacy codebases into
DDD aggregates that become workflows.

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

## Usage Examples

```bash
# Invoke skills directly
/mcp-companion:workflow-creation
/mcp-companion:sync
/mcp-companion:download

# Or just ask naturally - skills trigger automatically
> create a workflow for an e-commerce order process
> sync my domain model with Qlerify
> download the Cart Microservice workflow to workflow.json
> save the swagger spec for my workflow to api.yaml
> extract the Order aggregate from shop-api and build a workflow
```
