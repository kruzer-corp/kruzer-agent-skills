# Use Case Layer

The use case contains the business rule. It receives all dependencies via the constructor and exposes a single `execute(input)` method.

## Pattern

```typescript
export default class MyUseCase {
  constructor(
    private logger: KrzLogger,
    private someDatasource: SomeDatasource,
    // additional datasources as needed
  ) {}

  async execute(input: MyInput): Promise<MyOutput> {
    // 1. Validate input
    // 2. Call datasources
    // 3. Apply business rules
    // 4. Return result or throw domain error
  }
}
```

## Rules

- **No direct access to `@kruzer/idk`** — the use case only calls methods on injected datasource instances. It never imports or instantiates `RestDataSource`, `MongoDbDataSource`, etc.
- **No HTTP knowledge** — the use case does not know about status codes, `ctx.body`, or HTTP handlers. It only throws typed domain errors.
- **No internal instantiation of dependencies** — everything comes through the constructor. The use case never calls `new SomeDatasource()` internally.
- **Single public method: `execute(input)`** — one use case, one responsibility.
- **Throw domain errors** — validation failures and business rule violations become `BadRequestError`, `NotFoundError`, `UnprocessableEntityError`, or `InternalServerError`. See `domain-errors.md`.

## Canonical Example

```typescript
// src/domain/usecases/change-vtex-order.usecase.ts
import { KrzLogger } from "@kruzer/idk";
import VtexOrdersDatasource from "../../datasources/vtex-orders.datasource";
import OmsDatasource from "../../datasources/oms.datasource";
import { BadRequestError, NotFoundError, UnprocessableEntityError } from "../errors/domain-errors";
import { ChangeOrderRequestItem } from "../entities/change-order.entity";

interface ChangeVtexOrderInput {
  orderId: string;
  reason: string;
  items: ChangeOrderRequestItem[];
}

export default class ChangeVtexOrderUseCase {
  constructor(
    private logger: KrzLogger,
    private vtexDatasource: VtexOrdersDatasource,
    private omsDatasource: OmsDatasource
  ) {}

  async execute(input: ChangeVtexOrderInput) {
    const { orderId, reason, items } = input;

    // 1. Input validation
    if (!orderId) throw new BadRequestError("orderId is required");
    if (!items?.length) throw new BadRequestError("items cannot be empty");

    // 2. Fetch external data
    const omsOrder  = await this.omsDatasource.getOrderById(orderId);
    const vtexOrder = await this.vtexDatasource.getOrderById(omsOrder.external_code);

    // 3. Business rule validation
    const validStatus = omsOrder.status_code === "4" || vtexOrder.status === "handling";
    if (!validStatus) throw new UnprocessableEntityError("Order is not in a changeable state");

    // 4. Resolve items (raises NotFoundError if item is missing)
    const resolvedItems = items.map(item => {
      const vtexItem = vtexOrder.items.find(i => i.refId === item.code);
      if (!vtexItem) throw new NotFoundError(`Item ${item.code} not found in VTEX order`);
      return { id: vtexItem.uniqueId, quantity: item.quantity, price: vtexItem.price };
    });

    // 5. Persist changes
    await this.vtexDatasource.changeOrder(vtexOrder.orderId, { reason, remove: { items: resolvedItems } });
    await this.omsDatasource.updateOrderStatus(orderId, "9");

    this.logger.info("Order changed successfully", { orderId, vtexOrderId: vtexOrder.orderId });
    return { orderId, vtexOrderId: vtexOrder.orderId };
  }
}
```

## Dependency Injection Pattern

The automation is the Composition Root — it instantiates everything and injects via constructor:

```typescript
// In the automation file:
const vtexDatasource = new VtexOrdersDatasource("vtex-orders", logger);
const omsDatasource  = new OmsDatasource("oms-api", logger);
const useCase        = new ChangeVtexOrderUseCase(logger, vtexDatasource, omsDatasource);

const result = await useCase.execute(ctx.body);
```
