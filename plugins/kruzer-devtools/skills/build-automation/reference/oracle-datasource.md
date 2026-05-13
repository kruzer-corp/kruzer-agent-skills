# OracleDataSource

Executes SQL queries against Oracle Database using a connector configured on the Kruzer iPaaS platform.

## Import and Instantiation

```typescript
import { OracleDataSource } from "@kruzer/idk";

const oracle = new OracleDataSource("connector-name-on-platform");
```

## Method: `query(sql, options?)`

```typescript
// Simple query
const funcionarios = await oracle.query(
  "SELECT * FROM FUNCIONARIOS WHERE ROWNUM <= 100"
);

// With bind variables — CRITICAL SYNTAX: each value must be { val: value }
const orders = await oracle.query(
  "SELECT * FROM ORDERS WHERE CUSTOMER_ID = :customerId AND STATUS = :status",
  {
    binds: {
      customerId: { val: 456 },
      status: { val: "ACTIVE" }
    }
  }
);
```

## Critical Difference: Oracle vs MySQL Bind Variables

This is the most common source of bugs when switching between database connectors.

| | MySQL (`MySqlDataSource`) | Oracle (`OracleDataSource`) |
|---|---|---|
| Bind syntax in SQL | `:variableName` | `:variableName` |
| Bind values object key | `variables` | `binds` |
| Bind value format | `{ variables: { name: value } }` | `{ binds: { name: { val: value } } }` |

**MySQL — value is direct:**
```typescript
await mysql.query("SELECT * FROM t WHERE id = :id", {
  variables: { id: 123 }
  //              ^^^
  //              direct value
});
```

**Oracle — value must be wrapped in `{ val: ... }`:**
```typescript
await oracle.query("SELECT * FROM T WHERE ID = :id", {
  binds: { id: { val: 123 } }
  //            ^^^^^^^^^^^
  //            must be { val: value }
});
```

Passing the value directly in `binds` (without `{ val: ... }`) will cause a runtime error in Oracle.

## Method: `command(alias, options?)`

Executes a pre-configured query alias defined on the iPaaS connector.

```typescript
// Simple
const stock = await oracle.command("query-current-stock");

// With binds
const sales = await oracle.command("sales-by-period", {
  binds: {
    startDate: { val: "2024-01-01" },
    endDate: { val: "2024-06-30" }
  }
});
```

## Bind Interface

```typescript
interface OracleOptions {
  binds?: {
    [key: string]: { val: any };  // every bind must have the "val" property
  };
  [key: string]: any;
}
```

## Datasource Wrapper Example

```typescript
// src/datasources/erp.datasource.ts
import { OracleDataSource, KrzLogger } from "@kruzer/idk";

export default class ErpDatasource {
  private client: OracleDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new OracleDataSource(datasourceName);
  }

  async findEmployeeById(employeeId: number): Promise<Employee> {
    const result = await this.client.query(
      "SELECT * FROM FUNCIONARIOS WHERE ID = :employeeId",
      {
        binds: { employeeId: { val: employeeId } }
      }
    );
    return result[0];
  }

  async findOrdersByStatus(status: string, limit: number): Promise<Order[]> {
    return this.client.query(
      "SELECT * FROM ORDERS WHERE STATUS = :status AND ROWNUM <= :limit",
      {
        binds: {
          status: { val: status },
          limit: { val: limit }
        }
      }
    );
  }
}
```
