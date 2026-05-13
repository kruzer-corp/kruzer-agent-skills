# Multi-Tenancy

Tenants are logical separators within the Kruzer platform that isolate credentials, triggers, and executions for different contexts — clients, environments, or business units.

## Concept

A tenant represents an isolated execution context. Every connector (datasource) has credentials configured per tenant. The **platform injects credentials automatically** at runtime — the code never handles credentials directly.

```
[Automation Code]
       │
       ▼
[RestDataSource("api-pagamentos")]
       │   connector name only — no credentials in code
       ▼
[Kruzer Platform] ← resolves and injects the correct credentials
       │           based on which tenant is executing
       ▼
[External System]
```

## How It Works in Code

The same code runs for all tenants without modification:

```typescript
import { RestDataSource, KrzLogger } from "@kruzer/idk";

const logger = new KrzLogger("info", "processar-pagamento");
const api = new RestDataSource("api-pagamentos");

export default async function run(param: string) {
  const params = JSON.parse(param);

  // The platform injects the correct API key / token / password
  // for the tenant that triggered this execution — no code change needed
  const response = await api.request("create-charge", {
    body: { amount: params.amount, description: params.description }
  });

  logger.info("Payment processed", { chargeId: response.id });
}
```

There is **no `if (tenant === "x")` branching in automation code**. If you find yourself checking a tenant name to decide which credentials to use, the architecture is wrong — credentials belong in the platform configuration, not in the code.

## Typical Tenant Organization

| Context | Tenant name (examples) |
|---|---|
| Production environment | `producao` |
| Staging environment | `homologacao` |
| Development environment | `desenvolvimento` |
| Specific client | `cliente-a`, `empresa-xyz` |
| Business unit | `filial-sp`, `depto-financeiro` |

## Explicit Tenant in the Constructor (Advanced)

All datasource constructors accept an optional tenant parameter. Use this only when a single automation must access **multiple tenants in the same execution**:

```typescript
// Reads from tenant A's database
const dbClienteA = new MySqlDataSource("crm-db", "cliente-a");

// Reads from tenant B's database in the same execution
const dbClienteB = new MySqlDataSource("crm-db", "cliente-b");
```

This is an advanced pattern. In most automations the executing tenant is determined by the trigger configuration — the optional parameter is not needed.

## Local Development

Use the CLI to select the active tenant before running automations locally:

```bash
krz select tenant   # interactive prompt — lists all available tenants
krz run my-automation -p ./params.json
```

The CLI injects the selected tenant's credentials automatically. There is no `.env` file to create or manage.

## Key Points

- Never hardcode credentials, tokens, API keys, or passwords anywhere in the code.
- Never branch on tenant name inside automation code.
- The connector name string (`"api-pagamentos"`) is the only identifier in the code. Everything else is platform-managed.
- To test with different tenants locally, run `krz select tenant` and re-run the automation — no code change needed.
