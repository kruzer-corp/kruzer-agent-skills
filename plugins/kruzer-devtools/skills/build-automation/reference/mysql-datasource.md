# MySqlDataSource / MsSqlDataSource

Executes SQL queries against MySQL or SQL Server databases using a connector configured on the Kruzer iPaaS platform.

## Import and Instantiation

```typescript
// MySQL
import { MySqlDataSource } from "@kruzer/idk";
const mysql = new MySqlDataSource("connector-name-on-platform");

// SQL Server
import { MsSqlDataSource } from "@kruzer/idk";
const mssql = new MsSqlDataSource("connector-name-on-platform");
```

The constructor string must match the connector name configured in the Kruzer iPaaS panel for the tenant.

## MySqlDataSource

### `query(sql, options?)`

Executes SQL directly via the platform connector.

```typescript
// Simple query
const users = await mysql.query("SELECT * FROM users WHERE active = 1");

// With variable binding — use :variableName in SQL
const result = await mysql.query(
  "SELECT * FROM orders WHERE status = :status AND date >= :date",
  { variables: { status: "pending", date: "2024-01-01" } }
);
```

### Variable Binding Syntax (MySQL)

Use `:variableName` placeholders in the SQL string and pass the values in `variables`:

```typescript
const order = await mysql.query(
  "SELECT * FROM orders WHERE id = :id AND tenant_id = :tenantId",
  { variables: { id: 123, tenantId: "tenant-abc" } }
);
```

### `execute(alias, options?)`

Executes a pre-configured query alias defined on the iPaaS connector.

```typescript
const customers = await mysql.execute("find-active-customers");

const orders = await mysql.execute("orders-by-period", {
  variables: { startDate: "2024-01-01", endDate: "2024-12-31" }
});
```

## MsSqlDataSource

### `query(sql, options?)`

```typescript
// Simple
const users = await mssql.query("SELECT TOP 100 * FROM Users");

// With variable binding — use @variableName for MSSQL
const orders = await mssql.query(
  "SELECT * FROM Orders WHERE CustomerId = @customerId",
  { variables: { customerId: 123 } }
);
```

### `procedure(params)`

Executes a stored procedure with input and output parameters.

```typescript
const result = await mssql.procedure({
  name: "sp_ProcessOrder",
  inputs: [
    { name: "OrderId", value: 12345 },
    { name: "Status", value: "Approved" }
  ],
  outputs: [
    { name: "Success" },
    { name: "Message" }
  ]
});

console.log(result.Success);
console.log(result.Message);
```

## Datasource Wrapper Example

```typescript
// src/datasources/orders.datasource.ts
import { MySqlDataSource, KrzLogger } from "@kruzer/idk";

export default class OrdersDatasource {
  private client: MySqlDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new MySqlDataSource(datasourceName);
  }

  async findByStatus(status: string): Promise<Order[]> {
    return this.client.query(
      "SELECT * FROM orders WHERE status = :status ORDER BY created_at DESC",
      { variables: { status } }
    );
  }

  async updateStatus(orderId: number, status: string): Promise<void> {
    await this.client.query(
      "UPDATE orders SET status = :status, updated_at = NOW() WHERE id = :id",
      { variables: { status, id: orderId } }
    );
  }

  async findByPeriod(startDate: string, endDate: string): Promise<Order[]> {
    return this.client.query(
      "SELECT * FROM orders WHERE created_at BETWEEN :startDate AND :endDate",
      { variables: { startDate, endDate } }
    );
  }
}
```

## Key Points

- MySQL uses `:variableName` for bind parameters (e.g., `:status`, `:id`).
- SQL Server uses `@variableName` for bind parameters (e.g., `@customerId`).
- Always use bind parameters — never interpolate values directly into SQL strings.
- The connector handles authentication, connection pooling, and SSL configuration. The code only provides the SQL and variables.

## Pagination

Always add `LIMIT` and `OFFSET` (or an equivalent cursor) when querying tables that can grow large. Never query without a bound on large collections.

```typescript
// Offset-based pagination
async findOrders(page: number, pageSize: number): Promise<Order[]> {
  return this.client.query(
    "SELECT * FROM orders ORDER BY created_at DESC LIMIT :limit OFFSET :offset",
    { variables: { limit: pageSize, offset: (page - 1) * pageSize } }
  );
}

// Cursor-based pagination (more stable for large datasets)
async findOrdersAfter(lastId: number, pageSize: number): Promise<Order[]> {
  return this.client.query(
    "SELECT * FROM orders WHERE id > :lastId ORDER BY id ASC LIMIT :limit",
    { variables: { lastId, limit: pageSize } }
  );
}
```

> See `oracle-datasource.md` for the critical syntax difference between MySQL and Oracle bind variables.
