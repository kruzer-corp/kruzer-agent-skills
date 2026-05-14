# Kruzer Agent Skills

A collection of skills composed as Claude Code plugins for building integrations on the Kruzer DevTools platform with best practices and architectural patterns.

- [Available Plugins](#available-plugins)
- [Installation for Claude Code](#installation-for-claude-code)
- [Usage with Other Agents](#usage-with-other-agents)
- [Usage](#usage)
- [Privacy](#privacy)

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [kruzer-devtools](plugins/kruzer-devtools/) | Skills for building automations on the Kruzer iPaaS platform: project structure, automation types (Cron and Webhook/API Gateway), use case and datasource layers, all `@kruzer/idk` connectors, testing, and deployment. |

## Installation for Claude Code

1. Start Claude:

```bash
claude
```

2. Add the Kruzer marketplace to Claude Code:

```bash
/plugin marketplace add kruzer-corp/kruzer-agent-skills
```

3. Install the plugin:

```bash
/plugin install kruzer-devtools@kruzer
```

4. Verify the plugin is loaded:

```bash
/plugin
```

You should see the Kruzer plugin listed under the Installed tab.

## Usage with Other Agents

These skills can be used with any agent. The `/plugin` installation commands are specific to Claude Code, but the skill content — architecture patterns, canonical code examples, and reference documentation — can be integrated manually into any tool that supports project-level context or rules files.

### Manual Installation

Clone this repository and copy the desired skill directory into your project:

```bash
git clone https://github.com/kruzer-corp/kruzer-agent-skills.git
```

```bash
cp -r kruzer-agent-skills/plugins/kruzer-devtools/skills/build-automation <destination>
```

### Cursor

Copy the skill directory to `.cursor/rules/` in your project. Cursor loads all files in that directory as project-level rules.

```bash
cp -r kruzer-agent-skills/plugins/kruzer-devtools/skills/build-automation .cursor/rules/kruzer-build-automation
```

The `SKILL.md` file becomes the primary rule, and the files inside `reference/` are available for the agent to read on demand. To configure MCP servers (for `search_kruzer` and `query_docs_filesystem_kruzer`), add the server entries to `.cursor/mcp.json`.

### OpenCode and Other Agents

Copy the skill directory to the location your agent reads as project context. Most agents that support a project-level system prompt or rules directory will pick up the `SKILL.md` content automatically. For agents with MCP support, configure the MCP server entries from `.claude/settings.local.json` in your agent's equivalent MCP configuration file.

## Usage

Once installed, the `build-automation` skill loads automatically whenever you are working on a Kruzer automation project. It provides:

- Architecture guidance for Cron and Webhook/API Gateway automations
- Canonical patterns for use cases, datasources, domain errors, and data transformations
- Reference documentation for all `@kruzer/idk` connectors (REST, MongoDB, MySQL, SQL Server, Oracle, SAP RFC)
- Testing conventions (Jest + `@swc/jest`) adapted for the automation architecture
- Deployment instructions and package dependency constraints

## Privacy

The Kruzer plugins do not collect, store, or transmit any user data or conversation information. All instructional content is provided locally through skill files.
