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

## Step 1: Find MCP credentials

```bash
cat ~/.claude.json 2>/dev/null | jq -r '.mcpServers.qlerify // empty'
```

Extract the `url` and `headers.x-api-key`.

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

### OpenAPI spec → YAML file
```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"generate_openapi_spec","arguments":{"workflowId":"...","projectId":"...","boundedContext":"..."}}}' \
  | jq -r '.result.content[0].text' > swagger.yaml
```

### Entities from workflow → JSON file
```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_workflow","arguments":{"workflowId":"...","projectId":"..."}}}' \
  | jq -r '.result.content[0].text | fromjson | .specification.schemas.entities' > entities.json
```

### Domain events from workflow → JSON file
```bash
curl -s "$MCP_URL" -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_workflow","arguments":{"workflowId":"...","projectId":"..."}}}' \
  | jq -r '.result.content[0].text | fromjson | .specification.domainEvents' > events.json
```

## When to use what

| Data size          | Method    | Example                                 |
|--------------------|-----------|-----------------------------------------|
| Small (< 50 lines) | MCP tool  | `list_workflows`                        |
| Large (> 50 lines) | curl + jq | `get_workflow`, `generate_openapi_spec` |
| Any "save to file" | curl + jq | Always, regardless of size              |

## Finding IDs first

Use MCP tools for small lookups:
- `list_workflows` → get workflow ID and project ID

Then use curl for the actual data fetch.
