# Kruzer Agent Skills

A collection of skills composed as Claude Code plugins for building integrations on the Kruzer DevTools platform with best practices and architectural patterns.

- [Available Plugins](#available-plugins)
- [Installation for Claude Code](#installation-for-claude-code)
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

## Usage

Once installed, the `build-automation` skill loads automatically whenever you are working on a Kruzer automation project. It provides:

- Architecture guidance for Cron and Webhook/API Gateway automations
- Canonical patterns for use cases, datasources, domain errors, and data transformations
- Reference documentation for all `@kruzer/idk` connectors (REST, MongoDB, MySQL, SQL Server, Oracle, SAP RFC)
- Testing conventions (Jest + `@swc/jest`) adapted for the automation architecture
- Deployment instructions and package dependency constraints

## Privacy

The Kruzer plugins do not collect, store, or transmit any user data or conversation information. All instructional content is provided locally through skill files.
