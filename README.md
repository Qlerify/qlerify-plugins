# Qlerify Plugins

Official Qlerify plugins for AI coding assistants — domain model sync, data export, and more. Works with Claude Code,
Gemini CLI, and Cursor.

## Prerequisites

1. **Qlerify account** with a workflow created
2. **Qlerify MCP server** configured with your API token (see setup per tool below)

## Installation

### Claude Code

Configure MCP server in `~/.claude/settings.local.json`:

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

After installation, skills are available as `/mcp-companion:sync` and `/mcp-companion:download`.

### Gemini CLI

Install each skill using the `--path` flag:

```bash
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/sync
gemini skills install https://github.com/qlerify/qlerify-plugins.git --path plugins/mcp-companion/skills/download
```

Configure the Qlerify MCP server in `~/.gemini/settings.json` per
[Gemini CLI docs](https://geminicli.com/docs/cli/mcp/).

### Cursor

1. Open Cursor Settings (`Cmd+Shift+J`)
2. Go to **Rules** > **Add Rule** > **Remote Rule (Github)**
3. Enter: `https://github.com/qlerify/qlerify-plugins`

Configure the Qlerify MCP server in Cursor's MCP settings.

Get your API token from the Qlerify UI.

## Plugins

### `mcp-companion`

Teaches AI agents how to effectively use Qlerify's MCP server. Contains the following skills:

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
/mcp-companion:sync
/mcp-companion:download

# Or just ask naturally - skills trigger automatically
> sync my domain model with Qlerify
> download the Cart Microservice workflow to workflow.json
> save the swagger spec for my workflow to api.yaml
```
