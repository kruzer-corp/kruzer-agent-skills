# Project Structure

## Standard Folder Layout

The repository is provisioned by the Kruzer platform from a standard template — `package.json`, `tsconfig.json`, and the base folder structure are created automatically. Never start from a blank repository.

```
meu-projeto/
├── automations/                    # Entrypoints executed by the Kruzer platform
│   └── changeVtexOrder.automation.ts
│
├── data-transformation/            # Pure conversion functions between formats
│   └── orders/
│       └── vtexOrderToOmsOrder.dt.ts
│
├── src/
│   ├── utils/                      # Infrastructure utilities (Webhook/API Gateway only)
│   │   └── http-handlers.ts        # HttpRequest type, onSuccessHandler, onErrorHandler
│   │
│   ├── datasources/                # Wrappers over @kruzer/idk connectors
│   │   └── vtex-orders.datasource.ts
│   │
│   └── domain/
│       ├── errors/                 # Typed domain errors
│       │   └── domain-errors.ts
│       ├── entities/               # Pure TypeScript types (no logic)
│       │   └── change-order.entity.ts
│       └── usecases/               # Business rules, isolated
│           └── change-vtex-order.usecase.ts
│
├── tests/
│   ├── unit/
│   │   ├── usecases/               # One spec file per use case
│   │   │   └── change-vtex-order.usecase.spec.ts
│   │   └── data-transformations/   # One spec file per DT function
│   │       └── vtexOrderToOmsOrder.dt.spec.ts
│   └── helpers/
│       └── factories/              # One factory per entity/input type
│           └── order.factory.ts
│
├── package.json
└── tsconfig.json
```

## Naming Conventions

| Suffix | Example | Responsibility |
|---|---|---|
| `.automation.ts` | `changeVtexOrder.automation.ts` | Platform entrypoint (controller) |
| `.usecase.ts` | `change-vtex-order.usecase.ts` | Business rule with `execute()` |
| `.datasource.ts` | `vtex-orders.datasource.ts` | External system access via `@kruzer/idk` |
| `.entity.ts` | `change-order.entity.ts` | Pure domain type (no methods) |
| `.dt.ts` | `vtexOrderToOmsOrder.dt.ts` | Data Transformation (pure function) |

- File names: **kebab-case**
- Class names: **PascalCase**
- Folder names: **kebab-case**

## Layer Responsibilities

| Layer | Location | What it does |
|---|---|---|
| Automation | `automations/` | Receives platform context, wires dependencies, delegates to use case |
| Utils | `src/utils/` | `HttpRequest` type + HTTP response handlers (Webhook/API Gateway only) |
| Datasource | `src/datasources/` | Wraps `@kruzer/idk` data sources, one class per external system |
| UseCase | `src/domain/usecases/` | Pure business logic, no HTTP knowledge, no direct IDK access |
| Entity | `src/domain/entities/` | TypeScript types shared between layers |
| Domain Errors | `src/domain/errors/` | Typed errors thrown by use cases |
| Data Transformation | `data-transformation/` | Pure functions to convert between external and domain formats |

## Key Rules

- The `automations/` file is the **Composition Root** — the only place that instantiates concrete dependencies.
- Use cases receive dependencies via constructor; they never instantiate anything.
- Datasources instantiate `@kruzer/idk` classes internally in their constructor.
- The `src/utils/http-handlers.ts` file is only relevant for Webhook/API Gateway automations. Cron automations do not use it.
