# Platform Setup — Prerequisites Before Writing Code

Before writing any datasource or automation code, the following must already be configured on the Kruzer iPaaS platform. Code that references a connector or alias that does not exist on the platform will fail at runtime with no compile-time warning.

## Required Setup Per Datasource Type

### REST API

| What must exist on the platform | Where it maps in code |
|---|---|
| A **Connector** with a name | `new RestDataSource("connector-name")` |
| One **Resource** per operation, each with an alias | `rest.request("alias")` |
| **Credentials** configured per tenant for that connector | injected automatically at runtime |

### MySQL / SQL Server

| What must exist on the platform | Where it maps in code |
|---|---|
| A **Connector** with a name | `new MySqlDataSource("connector-name")` |
| **Credentials** configured per tenant | injected automatically |
| *(Optional)* One **Query alias** per pre-configured query | `mysql.execute("alias")` |

`query()` sends SQL directly and does not require a pre-configured alias. `execute()` does.

### MongoDB

| What must exist on the platform | Where it maps in code |
|---|---|
| A **Connector** with a name | `new MongoDbDataSource("connector-name")` |
| One **Resource** per operation, each with an alias and operation type | `mongo.command("alias", payload)` |
| **Credentials** configured per tenant | injected automatically |

The operation type (find, insertOne, updateMany, aggregate, etc.) is defined on the resource in the platform — not in the code.

### Oracle

| What must exist on the platform | Where it maps in code |
|---|---|
| A **Connector** with a name | `new OracleDataSource("connector-name")` |
| **Credentials** configured per tenant | injected automatically |
| *(Optional)* One **Query alias** per pre-configured query | `oracle.command("alias")` |

### SAP RFC

| What must exist on the platform | Where it maps in code |
|---|---|
| A **Connector** with a name | `new SapRFCDatasource("connector-name")` |

SAP RFC credentials are **not** managed by the platform — they are passed directly in `call()` via environment variables. See `sap-rfc-datasource.md` for details.

---

## Connector and Alias Names — Never Invent Them

Connector names and alias names cannot be inferred from the code or guessed. They are configured manually on the platform by the team responsible for the environment.

**Before writing any datasource code, ask the user:**

1. What is the exact name of the connector on the platform?
2. For REST and MongoDB: what are the alias/resource names configured on that connector?

Example questions to ask:
- "What is the name of the REST connector for the payments API on the platform?"
- "What aliases are configured on the `vtex-orders` connector?"

**Do not suggest connector or alias names as if they exist.** A connector named `"vtex-api"` and one named `"vtex-orders"` are completely different entries on the platform.

> **Kruzer Docs MCP**: use `search_kruzer` or `query_docs_filesystem_kruzer` to look up platform concepts, connector types, and feature documentation at any time.

---

## Credentials Setup (Non-SAP Datasources)

Credentials are configured per tenant on the platform — the code never handles them. For a datasource to work:

1. The connector must exist on the platform
2. Each tenant that will execute the automation must have credentials configured for that connector
3. `krz select tenant` (CLI) selects which tenant's credentials are used in local development

If a datasource call fails with an authentication or "connector not found" error at runtime, the likely cause is a missing connector or missing credentials for the executing tenant — not a code problem.
