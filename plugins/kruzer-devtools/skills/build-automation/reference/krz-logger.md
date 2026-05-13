# KrzLogger

Structured logger based on Winston with OpenTelemetry integration. Automatically includes the `trace-id` when an active span is present.

## Instantiation

```typescript
import { KrzLogger } from "@kruzer/idk";

const logger = new KrzLogger();
```

Instantiate in the automation and pass via constructor to use cases and datasources. Never instantiate inside a use case.

## Log Levels

| Level | When to use |
|---|---|
| `error` | Failures that prevent correct operation |
| `warn` | Abnormal situations that do not prevent operation |
| `info` | Relevant flow information (default in production) |
| `debug` | Detail for debugging (use in development) |

## Methods

```typescript
// Informational
logger.info("Application started");
logger.info("User authenticated", { userId: 123, email: "user@example.com" });

// Error with context
logger.error("Failed to connect to database");
logger.error("Error processing order", {
  orderId: 456,
  err: error.message,
  stack: error.stack
});

// Warning
logger.warn("High request rate", { requestsPerMinute: 150 });

// Debug (visible only when level = 'debug')
logger.debug("Received payload", { body: requestBody });
```

## Structured Logging with Objects

Pass a plain object as the second argument to attach structured data to the log entry:

```typescript
logger.error("Sync failed", { err });         // attach error object
logger.info("Batch processed", { total, skipped, errors });
logger.warn("Retry attempt", { attempt, maxRetries, url });
```

## Output Format (JSON)

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "severity": "info",
  "message": "User authenticated",
  "metadata": {
    "service": "service-name",
    "trace-id": "abc123def456..."
  },
  "data": {
    "userId": 123,
    "email": "user@example.com"
  }
}
```

## Where to Instantiate

- **Automation**: always instantiate `new KrzLogger()` at the top of `run()`.
- **Use cases**: receive `logger` via constructor, do not create their own instance.
- **Datasources**: receive `logger` via constructor, log infrastructure-level events.

```typescript
// Automation wires the logger through all layers:
export default async function run(param: string) {
  const logger = new KrzLogger();

  const datasource = new MyDatasource("connector-name", logger);
  const useCase    = new MyUseCase(logger, datasource);

  await useCase.execute(parsedParams);
}
```
