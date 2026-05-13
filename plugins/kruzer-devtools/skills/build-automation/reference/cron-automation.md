# Cron Automation

Triggered by a schedule configured on the Kruzer platform. The platform stores trigger parameters as a JSON-stringified string in the database, which is why the function signature receives `param: string`.

## Mandatory Signature

```typescript
export default async function run(param: string) { ... }
```

Never use `run(params: object)` or `run(params: any)` — the platform always delivers a JSON string.

## Why `param` is `string`

When a Cron trigger is configured on the platform, the parameters are saved as JSON in the database. At execution time, the platform passes the raw stringified value. The automation is responsible for parsing it with `JSON.parse(param)`.

## Required Pattern

1. Guard against missing `param` before parsing.
2. Declare the expected parameter interface in the same file as the automation.
3. Parse with `JSON.parse(param)`.
4. Instantiate datasources and use case.
5. Call `useCase.execute(parsedParams)`.
6. Re-throw errors so the platform registers the execution failure.

## Canonical Example

```typescript
// automations/sync-inventory.automation.ts
import { KrzLogger } from "@kruzer/idk";
import InventoryDatasource from "../src/datasources/inventory.datasource";
import SyncInventoryUseCase from "../src/domain/usecases/sync-inventory.usecase";

interface SyncInventoryParams {
  startDate: string;
  pageSize?: number;
}

export default async function run(param: string) {
  const logger = new KrzLogger();

  try {
    if (!param) throw new Error("Missing params");

    const parsedParams: SyncInventoryParams = JSON.parse(param);

    const datasource = new InventoryDatasource("erp-database", logger);
    const useCase    = new SyncInventoryUseCase(logger, datasource);

    await useCase.execute(parsedParams);

    logger.info("Sync completed successfully");
  } catch (err) {
    logger.error("Automation error", { err });
    throw err;
  }
}
```

## Rules

- **No HTTP response** — Cron automations have no HTTP contract. The return value is ignored by the platform.
- **`throw err` is mandatory** — re-throwing lets the platform detect and record execution failures. Swallowing exceptions silently means failures go unnoticed.
- **No business logic in the automation** — the automation only wires dependencies and delegates. All logic lives in the use case.
- **Declare the params interface locally** — it documents what the trigger expects without creating a separate types file for a single automation.
- **The datasource name string** passed to the constructor must match the connector name configured on the Kruzer platform for the tenant. Never hardcode URLs or credentials.
