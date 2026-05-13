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

### Return structure

`request()` always returns a platform envelope object — **not** the raw external API body:

```typescript
{
  data: <external API response body>,  // the actual payload you want
  // ...other platform metadata
}
```

**Always access `.data` to get the payload.** Returning `request()` directly passes the envelope to callers, causing `undefined` on all fields. What the payload contains depends on the external API — the user must confirm the response shape before writing any mapping code.

```typescript
// WRONG — returns the envelope, not the payload
async getPokemon(name: string) {
  return this.client.request("get-pokemon", { urlParams: { name } });
}

// CORRECT — unwrap .data before returning
async getPokemon(name: string): Promise<Pokemon> {
  const response = await this.client.request("get-pokemon", { urlParams: { name } });
  return response.data;
}
```

### Usage examples

```typescript
// Simple call — no dynamic data
const response = await rest.request("list-products");
const products = response.data;

// With query params
const res = await rest.request("list-customers", {
  params: { page: 1, limit: 20, status: "active" }
});
const customers = res.data;

// With request body
const res = await rest.request("create-order", {
  body: {
    customerId: 123,
    products: [{ id: 1, quantity: 2 }]
  }
});
const newOrder = res.data;

// With URL params (path substitution)
// Connector endpoint configured as: /products/{id}
const res = await rest.request("product-details", {
  urlParams: { id: "12345" }
  // Resolved URL: /products/12345
});
const details = res.data;

// With custom headers
const res = await rest.request("protected-endpoint", {
  headers: {
    "X-Custom-Header": "value",
    "Accept-Language": "en-US"
  },
  body: { data: "example" }
});
const result = res.data;

// Preserve the original Authorization header from the connector
const res = await rest.request("authenticated-api", {
  body: { info: "data" },
  options: { preserveAuthorizationHeader: true }
});
const response = res.data;
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
    const response = await this.client.request("getOrderById", {
      urlParams: { orderId }
    });
    return response.data; // unwrap platform envelope
  }

  async changeOrder(orderId: string, body: ChangeOrderBody): Promise<void> {
    await this.client.request("changeOrder", {
      urlParams: { changeOrderId: orderId },
      body
    });
  }

  async listOrders(page: number, status: string): Promise<VtexOrderDetails[]> {
    const response = await this.client.request("listOrders", {
      params: { page, status }
    });
    return response.data; // unwrap platform envelope
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
- `request()` returns a platform envelope `{ data: <payload> }` — always unwrap `.data` inside the datasource method before returning to callers. What the payload contains is specific to each external API.
