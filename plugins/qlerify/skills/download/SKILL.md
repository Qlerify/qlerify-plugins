---
name: download
description: >
  This skill should be used when the user asks to "save to file", "download",
  "export", "store in file", or any request that involves getting data from
  Qlerify and saving it locally. Bypasses AI processing and is ~100x faster
  than MCP tools for large data exports.
allowed-tools: Bash, Read, Glob
---

# Fast Download Qlerify Data

**CRITICAL:** When saving ANY Qlerify data to a file, use `curl + jq` instead of MCP tools. MCP responses pass through
AI context which takes minutes for large data. Shell pipes take seconds.

## Step 1: Load the MCP credentials without printing them

Start every command with this block, in the same Bash call: shell variables do not carry over from one call to the
next. It looks for the `qlerify` server where `claude mcp add` saves it (this project first, then the project's
`.mcp.json`, then the user-wide settings) and never prints the API key. Never print `$API_KEY` or the server entry.

```bash
SERVER=$(jq -c --arg dir "$PWD/" '[.projects // {} | to_entries[] | select(.value.mcpServers.qlerify != null) | select(.key as $project | $dir | startswith($project + "/"))] | sort_by(.key | length) | last | .value.mcpServers.qlerify // empty' ~/.claude.json 2>/dev/null)
[ -z "$SERVER" ] && SERVER=$(jq -c '.mcpServers.qlerify // empty' .mcp.json 2>/dev/null)
[ -z "$SERVER" ] && SERVER=$(jq -c '.mcpServers.qlerify // empty' ~/.claude.json 2>/dev/null)
MCP_URL=$(jq -r '.url // empty' <<< "$SERVER")
API_KEY=$(jq -r '.headers["x-api-key"] // empty' <<< "$SERVER")
[ -n "$MCP_URL" ] && [ -n "$API_KEY" ] || echo "No qlerify MCP server with an API key found" >&2
```

If nothing is found, tell the user to add the server with
`claude mcp add --transport http qlerify https://mcp.qlerify.com --header "x-api-key: YOUR_API_KEY"`.

## Step 2: Generic pattern for any MCP tool

```bash
curl -s "$MCP_URL" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "TOOL_NAME",
      "arguments": { ...ARGS... }
    }
  }' | jq -r '.result.content[0].text | fromjson' > output.json
```

## Common examples

### Full workflow → JSON file

```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_workflow","arguments":{"workflowId":"...","projectId":"..."}}}' \
  | jq -r '.result.content[0].text | fromjson | .specification' > workflow.json
```

### Entities from workflow → JSON file

`get_workflow` puts the first bounded context's entities under `schemas` and every other bounded context's under
`externalBoundedContexts`, so collect both:

```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_workflow","arguments":{"workflowId":"...","projectId":"..."}}}' \
  | jq -r '.result.content[0].text | fromjson | .specification | [.schemas.entities // {}] + [.externalBoundedContexts // {} | .[] | .schemas.entities // {}] | add' > entities.json
```

### Domain events from workflow → JSON file

Events are split across bounded contexts the same way:

```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_workflow","arguments":{"workflowId":"...","projectId":"..."}}}' \
  | jq -r '.result.content[0].text | fromjson | .specification | [.domainEvents // {}] + [.externalBoundedContexts // {} | .[] | .domainEvents // {}] | add' > events.json
```

## When to use what

| Data size          | Method    | Example                    |
|--------------------|-----------|----------------------------|
| Small (< 50 lines) | MCP tool  | `list_workflows`           |
| Large (> 50 lines) | curl + jq | `get_workflow`             |
| Any "save to file" | curl + jq | Always, regardless of size |

## Finding IDs first

Use MCP tools for small lookups:

- `list_workflows` → get workflow ID and project ID

Then use curl for the actual data fetch.
