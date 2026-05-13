# RestDataSource

Executes pre-configured REST endpoints on the Kruzer iPaaS by alias. The base URL, HTTP method, and default headers are set in the connector — the code only passes dynamic data.

## Import and Instantiation

```typescript
import { RestDataSource } from "@kruzer/idk";

const rest = new RestDataSource("connector-name-on-platform");
```

The string `"connector-name-on-platform"` must match exactly the connector name configured in the Kruzer iPaaS panel for the tenant.

## Method: `request(alias, options?)`

`alias` is the name of the operation (resource) configured on the connector in the platform. The code does not contain the URL — only the alias and runtime data.

```typescript
// Simple call — no dynamic data
const products = await rest.request("list-products");

// With query params
const customers = await rest.request("list-customers", {
  params: { page: 1, limit: 20, status: "active" }
});

// With request body
const newOrder = await rest.request("create-order", {
  body: {
    customerId: 123,
    products: [{ id: 1, quantity: 2 }]
  }
});

// With URL params (path substitution)
// Connector endpoint configured as: /products/{id}
const details = await rest.request("product-details", {
  urlParams: { id: "12345" }
  // Resolved URL: /products/12345
});

// With custom headers
const result = await rest.request("protected-endpoint", {
  headers: {
    "X-Custom-Header": "value",
    "Accept-Language": "en-US"
  },
  body: { data: "example" }
});

// Preserve the original Authorization header from the connector
const response = await rest.request("authenticated-api", {
  body: { info: "data" },
  options: { preserveAuthorizationHeader: true }
});
```

## Options Interface

```typescript
interface RestOptions {
  body?: any;
  params?: { [key: string]: any };         // Query string parameters
  headers?: { [key: string]: any };        // Additional/override headers
  urlParams?: { [key: string]: any };      // Path parameter substitution
  baseURL?: string;                        // Override connector base URL at runtime
  options?: {
    preserveAuthorizationHeader?: boolean;
  };
}
```

## Full Datasource Wrapper Example

```typescript
// src/datasources/vtex-orders.datasource.ts
import { RestDataSource, KrzLogger } from "@kruzer/idk";

export default class VtexOrdersDatasource {
  private client: RestDataSource;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new RestDataSource(datasourceName);
  }

  async getOrderById(orderId: string): Promise<VtexOrderDetails> {
    return this.client.request("getOrderById", {
      urlParams: { orderId }
    });
  }

  async changeOrder(orderId: string, body: ChangeOrderBody): Promise<void> {
    return this.client.request("changeOrder", {
      urlParams: { changeOrderId: orderId },
      body
    });
  }

  async listOrders(page: number, status: string): Promise<VtexOrderDetails[]> {
    return this.client.request("listOrders", {
      params: { page, status }
    });
  }
}
```

## Pagination

When consuming paginated APIs, pass `page` / `limit` (or whatever pagination parameters the external API uses) via `params`:

```typescript
async listOrdersPaginated(page: number, limit: number): Promise<Order[]> {
  return this.client.request("list-orders", {
    params: { page, limit, sort: "createdAt:desc" }
  });
}
```

For large datasets, iterate pages in a loop inside the use case until the external API signals the end of results (e.g., empty array, `hasMore: false`, or total count reached).

## Key Points

- Never use axios, fetch, or node-http to call external APIs. Always go through `RestDataSource`.
- The `alias` passed to `request()` must match the resource/operation name configured in the connector on the platform.
- URL construction, authentication, **retry, and timeout are handled automatically by the platform connector** — do not implement manual retry logic on top of `RestDataSource`.
