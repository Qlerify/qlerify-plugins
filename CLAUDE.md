# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Qlerify Plugins is a plugin distribution repository that packages AI agent skills for working with Qlerify — a domain modeling and workflow management platform. The plugins enable AI assistants (Claude Code, Gemini CLI, Cursor) to synchronize domain-driven design (DDD) models between local codebases and Qlerify workflows.

There is **no build system, no dependencies, and no tests**. This is a pure documentation/configuration project distributed via Git.

## Architecture

The repository has two layers:

1. **Marketplace registry** (`.claude-plugin/marketplace.json`) — registers the plugin collection for Claude Code's marketplace system.
2. **Plugin package** (`plugins/mcp-companion/`) — the actual plugin containing skills, permissions, and metadata.

### Plugin Structure

```
plugins/mcp-companion/
├── .claude-plugin/plugin.json   # Plugin manifest (name, version, keywords)
├── settings.json                # Allowlist-based permission model
└── skills/
    ├── sync/SKILL.md            # Bidirectional domain model sync skill
    └── download/SKILL.md        # Fast data export via curl+jq skill
```

### Key Design Decisions

- **Skills are markdown files** with YAML frontmatter — no code compilation needed.
- **settings.json uses an explicit allowlist** — only listed tools/commands are permitted. The plugin has read-only file access and no write/edit capabilities.
- **Multi-platform**: The same skill markdown files are shared across Claude Code (marketplace plugin), Gemini CLI (git path install), and Cursor (remote rules). Installation mechanisms differ per platform (see README.md).
- **The download skill bypasses AI context processing** by piping `curl | jq` directly to files for ~100x faster large data exports.

## Versioning (don't forget the bump)

**Every releasable change to the plugin requires a version bump — this is easy to forget, so treat it as a required step of the change, not an afterthought.** Any edit that will be published (a skill, a reference doc, a tool description, `settings.json`) counts.

Bump the plugin's version in **both** files, keeping them **identical**:

- `plugins/mcp-companion/.claude-plugin/plugin.json` → `version`
- `.claude-plugin/marketplace.json` → the `version` **inside the `mcp-companion` entry** of the `plugins` array. Do **not** touch the marketplace's own top-level `version` (currently `1.0.0`) — that tracks the marketplace itself, not the plugin.

Follow semver: **patch** (`0.4.14 → 0.4.15`) for doc fixes and wording tweaks, **minor** (`0.4.x → 0.5.0`) for new skills, tools, or capabilities. Do the bump in the same commit as the change so a release is never published with a stale version.

## Domain Concepts

The skills operate on DDD concepts from Qlerify workflows: **Entities** (persistent domain objects), **Commands** (state-changing operations), **Read Models/Queries** (data retrieval views), **Aggregate Roots**, **Bounded Contexts**, **Domain Events**, **Lanes** (actors/systems), and **Groups** (workflow phases).

## Integration

The plugin requires a configured Qlerify MCP server (`https://mcp.qlerify.com`) with API key authentication. The MCP server provides 29+ tools for CRUD operations on workflow elements.
