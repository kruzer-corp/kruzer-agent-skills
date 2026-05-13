# MongoDbDataSource

Executes MongoDB operations configured in the Kruzer iPaaS connector. The operation type (find, insertOne, insertMany, aggregate, updateOne, updateMany) is **defined in the connector resource on the platform** — not in the code. The `alias` identifies the resource; the code passes the correct payload for the configured operation.

## Import and Instantiation

```typescript
import { MongoDbDataSource } from "@kruzer/idk";

const mongo = new MongoDbDataSource("connector-name-on-platform");
```

## Method: `command(alias, payload?)`

The alias corresponds to a resource configured on the MongoDB connector in the platform. The payload structure depends on what operation that resource is configured to perform.

### find — Query documents

```typescript
// No filter
const documents = await mongo.command("find-active-users");

// With filter and options
const orders = await mongo.command("find-orders", {
  query: { status: "pending" },
  options: {
    sort: { createdAt: -1 },
    projection: { _id: 1, status: 1, total: 1 }
  }
});
```

### insertOne — Insert a single document

```typescript
await mongo.command("create-user", {
  document: {
    name: "John Smith",
    email: "john@example.com",
    active: true
  }
});
```

### insertMany — Insert multiple documents

```typescript
await mongo.command("import-products", {
  documents: [
    { name: "Product A", price: 10.00 },
    { name: "Product B", price: 20.00 }
  ]
});
```

### aggregate — Aggregation pipeline

```typescript
const report = await mongo.command("monthly-sales-aggregation", {
  pipeline: [
    { $match: { year: 2024 } },
    { $group: { _id: "$month", total: { $sum: "$amount" } } }
  ]
});
```

### updateOne / updateMany — Update documents

```typescript
// updateOne
await mongo.command("update-user", {
  query: { _id: "507f1f77bcf86cd799439011" },
  document: { name: "John Updated", email: "updated@example.com" }
  // $set is applied automatically by the platform API
});

// updateMany
await mongo.command("deactivate-users", {
  query: { lastAccess: { $lt: "2023-01-01" } },
  document: { active: false }
});
```

## Payload Interface

```typescript
interface MongoDbOptions {
  options?: {
    sort?: { [key: string]: 1 | -1 };
    projection?: { [key: string]: 0 | 1 };
    [key: string]: any;
  };
  query?: { [key: string]: any };          // for find, updateOne, updateMany
  document?: { [key: string]: any };       // for insertOne, updateOne, updateMany
  documents?: { [key: string]: any }[];   // for insertMany
  pipeline?: { [key: string]: any }[];    // for aggregate
}
```

## Key Points

- The operation type is configured on the platform, not in the code. The alias encodes the intent (e.g., `"find-orders"` is a find, `"create-user"` is an insertOne).
- For updates, `$set` is applied automatically — pass only the fields to change in `document`.
- Use the correct payload key for the operation: `query` for filters, `document` for single-document writes, `documents` for batch inserts, `pipeline` for aggregation.

## Datasource Wrapper Example

```typescript
// src/datasources/orders.datasource.ts
import { MongoDbDataSource, KrzLogger } from "@kruzer/idk";

export default class OrdersDatasource {
  private client: MongoDbDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new MongoDbDataSource(datasourceName);
  }

  async findPendingOrders(): Promise<Order[]> {
    return this.client.command("find-pending-orders", {
      query: { status: "pending" },
      options: { sort: { createdAt: -1 } }
    });
  }

  async insertOrder(order: Order): Promise<void> {
    await this.client.command("create-order", { document: order });
  }

  async markAsSynced(orderId: string): Promise<void> {
    await this.client.command("update-order", {
      query: { _id: orderId },
      document: { synced: true, syncedAt: new Date().toISOString() }
    });
  }

  async findPaginated(page: number, pageSize: number): Promise<Order[]> {
    return this.client.command("find-orders", {
      options: {
        sort: { createdAt: -1 },
        skip: (page - 1) * pageSize,
        limit: pageSize
      }
    });
  }
}
```

## Pagination

Always pass `skip` and `limit` in `options` when querying collections that can grow large. Never run a `find` without a limit on large collections.

```typescript
// Offset-based: skip N documents, return next M
await this.client.command("find-orders", {
  query: { status: "pending" },
  options: { sort: { createdAt: -1 }, skip: 0, limit: 50 }
});

// Cursor-based: use a field value as a cursor (more stable for large datasets)
await this.client.command("find-orders", {
  query: { _id: { $gt: lastId }, status: "pending" },
  options: { sort: { _id: 1 }, limit: 50 }
});
```
