# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code marketplace plugin that loads architecture conventions, patterns, and `@kruzer/idk` documentation into the AI assistant context when a developer builds integrations on the Kruzer DevTools platform.

Users install it via:
```bash
/plugin marketplace add kruzer/kruzer-agent-skills
/plugin install kruzer-devtools@kruzer
```

There is no build, lint, or test step — this repository contains only Markdown documentation files and JSON configuration. Changes take effect when pushed to the configured branch.

## Repository Structure

```
kruzer-agent-skills/
├── .claude-plugin/marketplace.json          # Marketplace root — lists available plugins
├── docs/                                    # Internal design documents (not shipped to users)
└── plugins/
    └── kruzer-devtools/
        ├── .claude-plugin/plugin.json       # Plugin metadata
        └── skills/
            └── build-automation/
                ├── SKILL.md                 # Skill entrypoint — loaded by the Claude Code runtime
                └── reference/              # Detailed reference files linked from SKILL.md
```

## How Skills Work

A skill is a Markdown file with YAML frontmatter consumed by the Claude Code runtime. The `name` and `description` fields determine when the skill loads automatically. Reference files are plain Markdown — the skill links to them via a table in `SKILL.md` with "read when..." triggers so the agent knows which file to read in each situation.

**To add a new skill:** create a new directory under `skills/` with a `SKILL.md` and optionally a `reference/` subdirectory. Register it in `plugin.json` if needed.

**To add a new reference file:** create the `.md` file in `reference/` and add a row to the Reference Files table in `SKILL.md`.

**To modify existing content:** edit the relevant `reference/*.md` file directly. The `SKILL.md` is the entrypoint — keep it as a summary and dispatcher, not a content dump.

## Plugin Configuration

`plugins/kruzer-devtools/.claude-plugin/plugin.json` — plugin metadata. The `mcpServers` field will be added here when the Kruzer DevTools MCP is published as an npm package.

`docs/` contains internal design documents used during development of this plugin. They are not shipped to end users and should not be referenced from skill files.

## MCP Tools Available in This Repo

The **Kruzer Docs MCP** is configured in `.claude/settings.local.json` and available in this workspace:

- `search_kruzer` — semantic search across Kruzer platform documentation
- `query_docs_filesystem_kruzer` — read specific documentation pages by path

Use these tools to verify platform behavior before updating skill content.

## Content Conventions

- All skill and reference files are written in English.
- Anti-patterns in `SKILL.md` follow the format: `| what not to do | why it is wrong |`.
- Reference files open with a one-line description of what the file covers, then sections with code examples.
- Key points go at the bottom of each reference file under `## Key Points`.
- The `docs/functionalities.md` file is the source of truth for `@kruzer/idk` API details. If a reference file and `functionalities.md` conflict, verify against the IDK source at `/home/chrysler-kruzer/Documents/kruzer/projects/idk/`.
