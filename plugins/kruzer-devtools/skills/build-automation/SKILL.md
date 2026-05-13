---
name: build-automation
description: >
  Load automatically when planning, creating, reviewing, or debugging any Kruzer
  automation project. Covers project structure, automation types (Cron vs Webhook/API Gateway),
  use case layer, datasource layer, domain errors, and all @kruzer/idk connectors.
  All external calls must go through @kruzer/idk datasources — using axios, fetch, or
  node-http directly is an anti-pattern.
---

# build-automation Skill

## MANDATORY SETUP — Run this before anything else

Before reading any other section, before planning, before writing a single line of code — run:

```bash
[ -f package.json ] && grep -q '"@kruzer/idk"' package.json && [ -f tsconfig.json ] && [ -d automations ] && echo "KRUZER_REPO_VALID" || echo "KRUZER_REPO_INVALID"
```

- If the output is `KRUZER_REPO_INVALID`: stop immediately and tell the user. See `reference/platform-setup.md` — Step 0.
- If the output is `KRUZER_REPO_VALID`: check whether `node_modules` exists.

```bash
[ -d node_modules ] && echo "DEPS_OK" || echo "DEPS_MISSING"
```

If the output is `DEPS_MISSING`, run `npm install` before doing anything else:

```bash
npm install
```

Do not create or edit any file until dependencies are confirmed present.

---

## Available MCP Tools

Two MCP integrations are available to assist development. Use them proactively instead of guessing or asking unnecessarily.

| MCP | Tools | When to use |
|---|---|---|
| **Kruzer Docs MCP** | `search_kruzer`, `query_docs_filesystem_kruzer` | Looking up platform concepts (tenants, triggers, connectors, API Gateway, consumers), understanding how a feature works, or clarifying any doubt about the Kruzer iPaaS platform |

**Always prefer the Kruzer Docs MCP over guessing** platform behavior. For connector and alias names, ask the user — see `reference/platform-setup.md`.

---

## When to Apply

Load this skill whenever you are:

- Creating a new Kruzer automation project from scratch
- Adding a new automation file (Cron or Webhook/API Gateway)
- Writing or reviewing a use case, datasource, or domain error class
- Using any `@kruzer/idk` connector (REST, MongoDB, MySQL, MSSQL, Oracle, SAP RFC)
- Debugging a failing automation on the Kruzer iPaaS platform
- Structuring or refactoring the folder layout of an existing project
- Reviewing code for architecture compliance (anti-patterns, layer violations)
- Writing tests for use cases or data transformation functions
- Asking about deployment, branch configuration, or package dependencies
- Looking up platform concepts (use the Kruzer Docs MCP)

---

## After Implementing — REQUIRED before reporting the task as done

Once all files are written, run the build to verify the TypeScript compiles:

```bash
npm run build
```

**Do not report the task as complete until this command exits with code 0.** A build failure means the code will also fail on the platform at deploy time. Fix all compilation errors before finishing.

---

## Anti-Patterns — Never Do These

| Anti-pattern | Why it is wrong |
|---|---|
| Writing or editing any `.ts` file before confirming `node_modules` exists | If `node_modules/` is absent, type errors are invisible during coding but fail at build time. Always check first; run `npm install` only if missing. |
| Reporting the task as done without running `npm run build` | TypeScript may compile locally but fail on deploy. The task is not complete until the build exits with code 0. |
| `return this.client.request(...)` directly from a datasource method | `request()` returns a platform envelope `{ data: <payload> }`, not the payload itself. Callers will receive `undefined` on every field. Always `const r = await this.client.request(...); return r.data`. |
| `import axios from "axios"` or `fetch(url)` in any file | All external HTTP calls must go through `RestDataSource` from `@kruzer/idk`. The platform handles auth, retry, and routing — bypassing it breaks multi-tenancy. |
| Business logic in the automation file | The automation is a controller. Rules, validations, and transformations belong in the use case. |
| A service layer between automation and use cases | There is no service layer in this architecture. Routing logic lives directly in the automation. If an automation grows too large, split it into multiple automations. |
| `new RestDataSource(...)` inside a use case | Use cases must not instantiate `@kruzer/idk` classes. The datasource wrapper is instantiated in the automation and injected via constructor. |
| Calling `@kruzer/idk` directly in a use case | Use cases only call methods on their injected datasource instances — never `@kruzer/idk` classes directly. |
| Hardcoded URLs, credentials, or tokens in datasources | The connector name string is the only identifier in the code. The platform injects the correct credentials at runtime. |
| `run(params: object)` in a Cron automation | Cron automations always receive `param: string`. Always use `run(param: string)` and `JSON.parse(param)`. |
| Missing `throw err` in a Cron automation's catch block | The platform detects execution failures only if the error propagates. Swallowing it silently marks the execution as successful. |
| Missing `try/catch` with `onErrorHandler` in a Webhook automation | Unhandled exceptions do not produce a valid HTTP response. Always wrap in `try/catch` and return `onErrorHandler(error as Error)`. |
| Adding external packages (`zod`, `joi`, `lodash`, etc.) to `package.json` | Automation pods use only the original `package.json`. New dependencies are not installed automatically — they will be missing at runtime. Use only `@kruzer/idk` and native Node.js modules. |
| Branching on tenant name in code (`if tenant === "x"`) | The platform handles credential resolution per tenant automatically. Tenant-specific behavior belongs in platform configuration, not in code. See `multi-tenancy.md`. |
| Inventing or assuming connector names or alias names | Connector and alias names are configured manually on the platform. They cannot be inferred. Before writing any datasource code, ask the user for the exact names. See `platform-setup.md`. |

---

## Mandatory Flow

Every Kruzer automation follows this strict layering:

```
[Kruzer Platform]
       │
       ▼
automations/*.automation.ts     ← Entrypoint / Controller
  - Receives platform context (param: string OR ctx: HttpRequest)
  - Instantiates datasources and use case (Composition Root)
  - Delegates to useCase.execute()
  - Returns response or re-throws
       │
       ▼
src/domain/usecases/*.usecase.ts  ← Business Rules
  - Receives all dependencies via constructor
  - Validates input, applies rules, orchestrates calls
  - Throws typed domain errors
  - Single public method: execute(input)
       │
       ▼
src/datasources/*.datasource.ts   ← External System Access
  - Wraps one @kruzer/idk class per external system
  - Instantiates the IDK class in its own constructor
  - Exposes named methods per operation
       │
       ▼
[@kruzer/idk]                     ← Platform SDK
  RestDataSource | MongoDbDataSource | MySqlDataSource
  MsSqlDataSource | OracleDataSource | SapRFCDatasource
```

---

## Quick Reference — File Suffixes

| Suffix | Example | Responsibility |
|---|---|---|
| `.automation.ts` | `sync-inventory.automation.ts` | Platform entrypoint, wires dependencies |
| `.usecase.ts` | `sync-inventory.usecase.ts` | Business rules, `execute(input)` method |
| `.datasource.ts` | `inventory.datasource.ts` | Wrapper over one `@kruzer/idk` connector |
| `.entity.ts` | `order.entity.ts` | Pure TypeScript type, no logic |
| `.dt.ts` | `vtexOrderToOms.dt.ts` | Pure data transformation function |

File names: **kebab-case**. Class names: **PascalCase**.

---

## Reference Files

Read these files for detailed patterns and canonical code examples. Each covers one specific topic.

| File | Read when... |
|---|---|
| `reference/project-structure.md` | Setting up a new project, deciding where a file belongs, or reviewing folder conventions |
| `reference/cron-automation.md` | Creating or reviewing a Cron-triggered automation (`run(param: string)`) |
| `reference/webhook-automation.md` | Creating or reviewing a Webhook/API Gateway automation (`run(ctx: HttpRequest)`) |
| `reference/usecase-layer.md` | Writing a use case class: constructor injection, `execute()`, domain errors, no direct IDK access |
| `reference/datasource-layer.md` | Writing a datasource wrapper class: pattern, rules, and how it wires into the automation |
| `reference/domain-errors.md` | Defining or throwing typed domain errors (`BadRequestError`, `NotFoundError`, etc.) and their HTTP mapping |
| `reference/krz-logger.md` | Using `KrzLogger` from `@kruzer/idk`: instantiation, levels, structured logging, where to create the instance |
| `reference/rest-datasource.md` | Using `RestDataSource` from `@kruzer/idk`: `request()`, aliases, urlParams, body, query, headers, pagination |
| `reference/mongodb-datasource.md` | Using `MongoDbDataSource` from `@kruzer/idk`: `command()`, find/insert/update/aggregate payloads, pagination |
| `reference/mysql-datasource.md` | Using `MySqlDataSource` or `MsSqlDataSource` from `@kruzer/idk`: `query()`, `:variableName` bind syntax, pagination |
| `reference/oracle-datasource.md` | Using `OracleDataSource` from `@kruzer/idk`: `query()`, `{ val: value }` bind syntax — critical difference from MySQL |
| `reference/sap-rfc-datasource.md` | Using `SapRFCDatasource` from `@kruzer/idk`: `call()`, IMPORT/TABLES parameters, credentials |
| `reference/platform-setup.md` | **Read first when starting any new project or writing datasource code.** How to create and clone the repository via the platform, CLI setup, and what connectors/aliases/credentials must be pre-configured before code will work |
| `reference/multi-tenancy.md` | Understanding how tenant credential isolation works, the optional tenant constructor parameter, and local development with `krz select tenant` |
| `reference/deploy.md` | After finishing any implementation (run `npm run build` before committing), understanding how deployment works (build + commit-triggered), package dependency constraints, and branch configuration |
| `reference/testing.md` | Writing unit tests for use cases and data transformations: stack (Jest + `@swc/jest`), mocking pattern, AAA, factory pattern, coverage thresholds |
