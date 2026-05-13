# Webhook / API Gateway Automation

Triggered by an external HTTP call — either a third-party webhook or an API Gateway endpoint. Receives a full HTTP context object.

## Mandatory Signature

```typescript
export default async function run(ctx: HttpRequest) { ... }
```

## HttpRequest Type

Defined in `src/utils/http-handlers.ts`:

```typescript
export type HttpRequest = {
  body?: any;                              // HTTP request body (JSON)
  params?: { [key: string]: any };         // Path parameters, e.g. /orders/:id
  query?: { [key: string]: any };          // Query string parameters
  headers?: { [key: string]: any };        // HTTP headers
  metadata?: { [key: string]: any };       // Platform-injected metadata
};
```

## Required Response Handlers

Also defined in `src/utils/http-handlers.ts`:

```typescript
export function onSuccessHandler(data: unknown) {
  return { status: 200, body: { data } };
}

export function onErrorHandler(error: Error) {
  // Maps domain errors to HTTP status codes:
  // BadRequestError          → 400
  // NotFoundError            → 404
  // UnprocessableEntityError → 422
  // all others               → 500
}
```

Both `onSuccessHandler` and `onErrorHandler` are mandatory. The automation must always return a structured HTTP response.

## Canonical Example — Single Use Case

```typescript
// automations/changeVtexOrder.automation.ts
import { KrzLogger } from "@kruzer/idk";
import { onSuccessHandler, onErrorHandler, HttpRequest } from "../src/utils/http-handlers";
import VtexOrdersDatasource from "../src/datasources/vtex-orders.datasource";
import OmsDatasource from "../src/datasources/oms.datasource";
import ChangeVtexOrderUseCase from "../src/domain/usecases/change-vtex-order.usecase";

export default async function run(ctx: HttpRequest) {
  const logger = new KrzLogger();
  try {
    const vtexDatasource = new VtexOrdersDatasource("vtex-orders", logger);
    const omsDatasource  = new OmsDatasource("oms-api", logger);
    const useCase        = new ChangeVtexOrderUseCase(logger, vtexDatasource, omsDatasource);

    const result = await useCase.execute(ctx.body);
    return onSuccessHandler(result);
  } catch (error) {
    return onErrorHandler(error as Error);
  }
}
```

## Example — Multiple Use Cases with Routing

When a single automation needs to route to different use cases based on the input, the automation itself handles the routing logic directly. There is no service layer for this.

```typescript
// automations/changeVtexOrder.automation.ts
import { KrzLogger } from "@kruzer/idk";
import { onSuccessHandler, onErrorHandler, HttpRequest } from "../src/utils/http-handlers";
import { BadRequestError } from "../src/domain/errors/domain-errors";
import VtexOrdersDatasource from "../src/datasources/vtex-orders.datasource";
import OmsDatasource from "../src/datasources/oms.datasource";
import CreateOrderUseCase from "../src/domain/usecases/create-order.usecase";
import CancelOrderUseCase from "../src/domain/usecases/cancel-order.usecase";
import InvoiceOrderUseCase from "../src/domain/usecases/invoice-order.usecase";

export default async function run(ctx: HttpRequest) {
  const logger = new KrzLogger();
  try {
    const vtexDatasource = new VtexOrdersDatasource("vtex-orders", logger);
    const omsDatasource  = new OmsDatasource("oms-api", logger);

    const createOrder  = new CreateOrderUseCase(logger, omsDatasource);
    const cancelOrder  = new CancelOrderUseCase(logger, vtexDatasource, omsDatasource);
    const invoiceOrder = new InvoiceOrderUseCase(logger, vtexDatasource, omsDatasource);

    const { State } = ctx.body;

    if (State === "payment-approved") return onSuccessHandler(await createOrder.execute(ctx.body));
    if (State === "canceled")         return onSuccessHandler(await cancelOrder.execute(ctx.body));
    if (State === "invoiced")         return onSuccessHandler(await invoiceOrder.execute(ctx.body));

    throw new BadRequestError(`Unhandled state: ${State}`);
  } catch (error) {
    return onErrorHandler(error as Error);
  }
}
```

## Rules

- **Always wrap in `try/catch` with `onErrorHandler`** — the HTTP response must always be controlled. An unhandled exception would return an unformatted error to the caller.
- **No business logic in the automation** — routing on `ctx.body.State` is control flow, not business logic. Business rules live in use cases.
- **No service layer** — if the automation grows too large due to the number of use cases, the correct signal is to split it into multiple smaller automations, not to create a service class to hide the complexity.
- **`ctx.params` vs `ctx.query` vs `ctx.body`** — use the right field for the right data. Path params go in `ctx.params`, query string in `ctx.query`, request body in `ctx.body`.
