# Domain Errors

Typed errors that represent business failures. They belong to the domain because they are thrown by use cases — not by the HTTP layer. The `onErrorHandler` in a Webhook/API Gateway automation catches them and maps them to the correct HTTP status code.

## The Four Domain Errors

| Class | HTTP Status | When to use |
|---|---|---|
| `BadRequestError` | 400 | Invalid or missing input — the caller sent bad data |
| `NotFoundError` | 404 | A required resource does not exist |
| `UnprocessableEntityError` | 422 | Input is valid but violates a business rule |
| `InternalServerError` | 500 | Unexpected failure that is not a user error |

## Implementation

```typescript
// src/domain/errors/domain-errors.ts

export class BadRequestError extends Error {
  readonly statusCode = 400;
  constructor(message: string) {
    super(message);
    this.name = "BadRequestError";
  }
}

export class NotFoundError extends Error {
  readonly statusCode = 404;
  constructor(message: string) {
    super(message);
    this.name = "NotFoundError";
  }
}

export class UnprocessableEntityError extends Error {
  readonly statusCode = 422;
  constructor(message: string) {
    super(message);
    this.name = "UnprocessableEntityError";
  }
}

export class InternalServerError extends Error {
  readonly statusCode = 500;
  constructor(message: string) {
    super(message);
    this.name = "InternalServerError";
  }
}
```

## Usage in a Use Case

```typescript
import { BadRequestError, NotFoundError, UnprocessableEntityError } from "../errors/domain-errors";

async execute(input: { orderId: string; items: Item[] }) {
  // Missing required field → 400
  if (!input.orderId) throw new BadRequestError("orderId is required");

  // Resource does not exist → 404
  const order = await this.ordersDatasource.getById(input.orderId);
  if (!order) throw new NotFoundError(`Order ${input.orderId} not found`);

  // Business rule violation → 422
  if (order.status !== "pending") {
    throw new UnprocessableEntityError("Only pending orders can be processed");
  }

  // proceed...
}
```

## Mapping in onErrorHandler

The `onErrorHandler` in `src/utils/http-handlers.ts` reads `error.statusCode` (or `error.name`) to produce the correct HTTP response. Use cases and datasources only throw; the mapping happens exclusively in the automation layer.

```typescript
export function onErrorHandler(error: Error) {
  const status = (error as any).statusCode ?? 500;
  return { status, body: { error: error.message } };
}
```

## Important

- **Cron automations** do not use `onErrorHandler`. They re-throw with `throw err` so the platform registers the failure. Domain errors can still be thrown in use cases called by Cron automations — they will propagate and be caught by the platform as execution failures.
- **Never throw raw `Error`** when a typed domain error is more precise. Typed errors make the failure observable and map to meaningful HTTP responses in Webhook/API Gateway automations.
