# Testing

Tests are written for **use cases** and **data transformations** only. Automations and datasource wrappers are not unit tested.

## What to Test

| Layer | Test? | Reason |
|---|---|---|
| Use cases (`*.usecase.ts`) | **Yes** — primary target | Contains all business logic |
| Data transformations (`*.dt.ts`) | **Yes** | Pure functions — trivially testable |
| Automations (`*.automation.ts`) | No | Wiring only (Composition Root), no logic to validate |
| Datasource wrappers (`*.datasource.ts`) | No | Infrastructure wrapper over `@kruzer/idk` — no business logic |
| Domain errors (`domain-errors.ts`) | No | Trivial typed classes, no logic |

## Stack

From ADR-0001 — already configured in the project `package.json`:

| Tool | Purpose |
|---|---|
| Jest 29 + `@swc/jest` | Test runner with fast TypeScript compilation |
| `jest.fn()` / `jest.Mocked<T>` | Type-safe mocking of datasource dependencies |
| `@faker-js/faker` + Factory pattern | Realistic, maintainable test data |

## File Structure

```
tests/
├── unit/
│   ├── usecases/                          # One spec file per use case
│   │   └── change-vtex-order.usecase.spec.ts
│   └── data-transformations/              # One spec file per DT function
│       └── vtexOrderToOmsOrder.dt.spec.ts
└── helpers/
    └── factories/                         # One factory per entity/input type
        └── order.factory.ts
```

## Test Naming

Pattern: **verb + object + condition**, in English, without `should` prefix (redundant — a test is already an expectation).

```ts
it('throws BadRequestError when orderId is missing', ...)
it('maps VTEX order fields to OMS format', ...)
it('calls changeOrder with resolved items when order is in handling status', ...)
```

Wrap tests in `describe` by class name, then by method:

```ts
describe('ChangeVtexOrderUseCase', () => {
  describe('execute()', () => {
    it('...', ...)
  })
})
```

## AAA Pattern (Arrange, Act, Assert)

Every test has three explicit sequential blocks. Each `it` has exactly one Act.

```ts
it('throws NotFoundError when item is not found in VTEX order', async () => {
  // Arrange
  const input = makeChangeOrderInput({ items: [{ code: 'SKU-999', quantity: 1 }] });
  vtexDatasource.getOrderById.mockResolvedValue(makeVtexOrder({ items: [] }));
  omsDatasource.getOrderById.mockResolvedValue(makeOmsOrder({ status_code: '4' }));

  // Act
  const act = () => useCase.execute(input);

  // Assert
  await expect(act()).rejects.toThrow(NotFoundError);
});
```

## Mocking Use Case Dependencies

The same DI pattern from the automation applies to tests: inject mocked datasources via constructor.

```ts
// tests/unit/usecases/change-vtex-order.usecase.spec.ts
import { KrzLogger } from "@kruzer/idk";
import ChangeVtexOrderUseCase from "../../../src/domain/usecases/change-vtex-order.usecase";
import VtexOrdersDatasource from "../../../src/datasources/vtex-orders.datasource";
import OmsDatasource from "../../../src/datasources/oms.datasource";
import { BadRequestError, NotFoundError, UnprocessableEntityError } from "../../../src/domain/errors/domain-errors";
import { makeChangeOrderInput, makeVtexOrder, makeOmsOrder } from "../../helpers/factories/order.factory";

const mockLogger = {
  info: jest.fn(),
  error: jest.fn(),
  warn: jest.fn(),
  debug: jest.fn(),
} as unknown as KrzLogger;

describe('ChangeVtexOrderUseCase', () => {
  let useCase: ChangeVtexOrderUseCase;
  let vtexDatasource: jest.Mocked<VtexOrdersDatasource>;
  let omsDatasource: jest.Mocked<OmsDatasource>;

  beforeEach(() => {
    vtexDatasource = {
      getOrderById: jest.fn(),
      changeOrder: jest.fn(),
    } as unknown as jest.Mocked<VtexOrdersDatasource>;

    omsDatasource = {
      getOrderById: jest.fn(),
      updateOrderStatus: jest.fn(),
    } as unknown as jest.Mocked<OmsDatasource>;

    useCase = new ChangeVtexOrderUseCase(mockLogger, vtexDatasource, omsDatasource);
  });

  afterEach(() => {
    jest.resetAllMocks(); // resets both call history AND implementation — do not use clearAllMocks
  });

  describe('execute()', () => {
    it('throws BadRequestError when orderId is missing', async () => {
      // Arrange
      const input = makeChangeOrderInput({ orderId: '' });

      // Act
      const act = () => useCase.execute(input);

      // Assert
      await expect(act()).rejects.toThrow(BadRequestError);
    });

    it('throws UnprocessableEntityError when order is not in a changeable state', async () => {
      // Arrange
      const input = makeChangeOrderInput();
      omsDatasource.getOrderById.mockResolvedValue(makeOmsOrder({ status_code: '1' }));
      vtexDatasource.getOrderById.mockResolvedValue(makeVtexOrder({ status: 'invoiced' }));

      // Act
      const act = () => useCase.execute(input);

      // Assert
      await expect(act()).rejects.toThrow(UnprocessableEntityError);
    });

    it('calls changeOrder and updateOrderStatus when execution succeeds', async () => {
      // Arrange
      const input = makeChangeOrderInput({ items: [{ code: 'SKU-001', quantity: 1 }] });
      const omsOrder = makeOmsOrder({ status_code: '4', external_code: 'vtex-123' });
      const vtexOrder = makeVtexOrder({
        orderId: 'vtex-123',
        status: 'handling',
        items: [{ refId: 'SKU-001', uniqueId: 'u1', price: 100 }],
      });
      omsDatasource.getOrderById.mockResolvedValue(omsOrder);
      vtexDatasource.getOrderById.mockResolvedValue(vtexOrder);

      // Act
      await useCase.execute(input);

      // Assert
      expect(vtexDatasource.changeOrder).toHaveBeenCalledWith(
        'vtex-123',
        expect.objectContaining({ reason: input.reason })
      );
      expect(omsDatasource.updateOrderStatus).toHaveBeenCalledWith(input.orderId, '9');
    });
  });
});
```

## Factory Pattern

Factories generate realistic data with sensible defaults. Override only what matters for each scenario.

```ts
// tests/helpers/factories/order.factory.ts
import { faker } from "@faker-js/faker";

export const makeChangeOrderInput = (overrides = {}) => ({
  orderId: faker.string.alphanumeric(10),
  reason: faker.lorem.sentence(),
  items: [{ code: faker.string.alphanumeric(8), quantity: faker.number.int({ min: 1, max: 10 }) }],
  ...overrides,
});

export const makeOmsOrder = (overrides = {}) => ({
  id: faker.string.alphanumeric(10),
  status_code: '4',
  external_code: `vtex-${faker.string.alphanumeric(8)}`,
  ...overrides,
});

export const makeVtexOrder = (overrides = {}) => ({
  orderId: `vtex-${faker.string.alphanumeric(8)}`,
  status: 'handling',
  items: [{
    refId: faker.string.alphanumeric(8),
    uniqueId: faker.string.uuid(),
    price: faker.number.float({ min: 1, max: 999, fractionDigits: 2 }),
  }],
  ...overrides,
});
```

## Data Transformation Tests

Pure functions require no mocking:

```ts
// tests/unit/data-transformations/vtexOrderToOmsOrder.dt.spec.ts
import vtexOrderToOmsOrder from "../../../data-transformation/orders/vtexOrderToOmsOrder.dt";
import { makeVtexOrder } from "../../helpers/factories/order.factory";

describe('vtexOrderToOmsOrder()', () => {
  it('maps orderId to external_code', () => {
    // Arrange
    const vtexOrder = makeVtexOrder({ orderId: 'vtex-abc-123' });

    // Act
    const result = vtexOrderToOmsOrder(vtexOrder);

    // Assert
    expect(result.external_code).toBe('vtex-abc-123');
  });
});
```

## Coverage Thresholds

From ADR-0001 — applied to the unit test suite (`npm run test:cov`):

| Metric | Minimum |
|---|---|
| Statements | 80% |
| Functions | 80% |
| Lines | 80% |
| Branches | 75% |

Coverage only grows — no PR may reduce global coverage relative to the base branch.

## Key Points

- `jest.resetAllMocks()` in `afterEach` — resets both call history and implementation. Never use `clearAllMocks` (it keeps mock implementations active, causing non-deterministic failures that vary by test execution order).
- Mock the datasource instance methods, not the `@kruzer/idk` classes — the use case never touches IDK directly.
- Each `it` creates its own data via factories — never share mutable state between tests.
- `beforeEach` is for infrastructure setup (instantiating the use case and mocks). Never create test data in `beforeEach`.
