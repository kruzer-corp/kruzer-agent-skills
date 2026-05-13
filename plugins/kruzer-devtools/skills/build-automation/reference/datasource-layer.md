# Datasource Layer

The datasource is a wrapper over a `@kruzer/idk` data source class. Each datasource class encapsulates all operations for a single external system.

## Pattern

```typescript
import { RestDataSource, KrzLogger } from "@kruzer/idk";

export default class MySystemDatasource {
  private client: RestDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new RestDataSource(datasourceName);
  }

  async someOperation(param: string): Promise<SomeType> {
    const response = await this.client.request("alias-configured-on-platform", { urlParams: { param } });
    return response.data; // always unwrap the platform envelope
  }
}
```

## Rules

- **One datasource per external system** — `VtexOrdersDatasource` handles all Vtex operations, `OmsDatasource` handles all OMS operations. Never mix systems in a single datasource class.
- **The `@kruzer/idk` class is instantiated internally** — the datasource's constructor creates the IDK instance. Use cases never create or hold IDK instances.
- **The `datasourceName` string must match the connector name on the platform** — this is the name configured in the Kruzer iPaaS panel for the tenant. Never hardcode URLs or credentials.
- **Each method = one operation** — `getOrderById`, `changeOrder`, `updateOrderStatus` are separate methods. Methods never mix responsibilities.
- **The datasource receives `logger` via constructor** — log infrastructure errors at the datasource level if needed; always re-throw.

## Canonical Example

```typescript
// src/datasources/vtex-orders.datasource.ts
import { RestDataSource, KrzLogger } from "@kruzer/idk";
import { VtexOrderDetails } from "../domain/entities/vtex-order.entity";
import { ChangeOrderBody } from "../domain/entities/change-order.entity";

export default class VtexOrdersDatasource {
  private client: RestDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new RestDataSource(datasourceName);
  }

  async getOrderById(orderId: string): Promise<VtexOrderDetails> {
    this.logger.info("Fetching VTEX order", { orderId });
    const response = await this.client.request("getOrderById", { urlParams: { orderId } });
    return response.data; // unwrap platform envelope
  }

  async changeOrder(orderId: string, body: ChangeOrderBody): Promise<void> {
    this.logger.info("Changing VTEX order", { orderId });
    await this.client.request("changeOrder", {
      urlParams: { changeOrderId: orderId },
      body
    });
  }
}
```

## Instantiation in the Automation

The automation creates the datasource instance, passing the connector name and logger:

```typescript
// In the automation file:
const vtexDatasource = new VtexOrdersDatasource("vtex-orders", logger);
const useCase = new ChangeVtexOrderUseCase(logger, vtexDatasource);
```

The string `"vtex-orders"` is the exact name of the connector configured on the Kruzer platform for the tenant. The platform injects the correct credentials automatically at runtime — no credentials appear in code.

## Applicable IDK Data Sources

The same wrapper pattern applies to any `@kruzer/idk` data source:

| IDK Class | Use for |
|---|---|
| `RestDataSource` | REST APIs configured on the platform |
| `MongoDbDataSource` | MongoDB connectors |
| `MySqlDataSource` | MySQL databases |
| `MsSqlDataSource` | SQL Server databases |
| `OracleDataSource` | Oracle databases |
| `SapRFCDatasource` | SAP RFC function calls |

See the individual reference files (`rest-datasource.md`, `mongodb-datasource.md`, etc.) for the specific API of each IDK class.
