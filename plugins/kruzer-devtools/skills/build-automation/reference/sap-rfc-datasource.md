# SapRFCDatasource

Calls SAP NetWeaver RFC functions.

## Credential Model — Different from Other Datasources

`SapRFCDatasource` does **not** use the platform's per-tenant credential system. Credentials are passed explicitly in every `call()` invocation and must be sourced from environment variables at runtime.

This is the only datasource where credentials appear in code. All other datasources (`RestDataSource`, `MySqlDataSource`, `MongoDbDataSource`, etc.) have their credentials managed by the platform and injected automatically — SAP RFC is the exception.

```
Other datasources:
  new RestDataSource("connector-name")
  → platform injects credentials automatically per tenant

SapRFCDatasource:
  new SapRFCDatasource("connector-name")
  → credentials must be passed explicitly in call()
  → credentials must be read from environment variables
```

## Import and Instantiation

```typescript
import { SapRFCDatasource, SapRFCCredentials } from "@kruzer/idk";

const sap = new SapRFCDatasource("connector-name-on-platform");

const credentials: SapRFCCredentials = {
  host: process.env.SAP_HOST,
  user: process.env.SAP_USER,
  password: process.env.SAP_PASSWORD,
  client: process.env.SAP_CLIENT,
  sysnr: process.env.SAP_SYSNR,
};
```

The constructor name must still match a connector configured on the Kruzer platform (used for routing). Credentials, however, are not managed through the platform — they come from environment variables.

## Method: `call(functionName, params, credentials)`

```typescript
const result = await sap.call(functionName, params, credentials);
```

- `functionName` — the RFC or BAPI function name (e.g., `"BAPI_CUSTOMER_GETDETAIL"`)
- `params` — object with IMPORT and TABLES parameters
- `credentials` — `SapRFCCredentials` object — always from environment variables, never hardcoded

## Parameter Types

| SAP Type | Code representation |
|---|---|
| IMPORT (scalar) | Top-level string/number properties in the params object |
| TABLES (table input) | Array-valued properties in the params object |

## Examples

### Simple BAPI — fetch customer details

```typescript
const customer = await sap.call(
  "BAPI_CUSTOMER_GETDETAIL",
  { CUSTOMERNO: "0000012345" },
  credentials
);
```

### BAPI with TABLES parameter

```typescript
const materials = await sap.call(
  "BAPI_MATERIAL_GETLIST",
  {
    MATNRSELECTION: [
      { SIGN: "I", OPTION: "CP", MATNR_LOW: "MAT*" }
    ],
    MAXROWS: 100
  },
  credentials
);
```

### Custom RFC with IMPORT and TABLES parameters

```typescript
const result = await sap.call(
  "Z_CUSTOM_RFC_FUNCTION",
  {
    I_PARAM1: "value1",
    I_PARAM2: 12345,
    I_TABLE: [
      { FIELD1: "A", FIELD2: 100 },
      { FIELD1: "B", FIELD2: 200 }
    ]
  },
  credentials
);
```

## Credentials Interface

```typescript
type SapRFCCredentials = {
  host: string;      // SAP server hostname
  user: string;      // RFC user
  password: string;  // RFC password — never log this value
  client: string;    // SAP client number (e.g., "100")
  sysnr: string;     // System number (e.g., "00")
};
```

## Datasource Wrapper Example

```typescript
// src/datasources/sap-erp.datasource.ts
import { SapRFCDatasource, SapRFCCredentials, KrzLogger } from "@kruzer/idk";

export default class SapErpDatasource {
  private client: SapRFCDatasource;
  private credentials: SapRFCCredentials;

  constructor(datasourceName: string, private logger: KrzLogger) {
    this.client = new SapRFCDatasource(datasourceName);
    this.credentials = {
      host: process.env.SAP_HOST,
      user: process.env.SAP_USER,
      password: process.env.SAP_PASSWORD,
      client: process.env.SAP_CLIENT,
      sysnr: process.env.SAP_SYSNR,
    };
  }

  async getCustomerDetails(customerNo: string): Promise<SapCustomer> {
    this.logger.info("Fetching SAP customer", { customerNo });
    return this.client.call(
      "BAPI_CUSTOMER_GETDETAIL",
      { CUSTOMERNO: customerNo },
      this.credentials
    );
  }

  async syncOrders(orderIds: string[]): Promise<SapSyncResult> {
    this.logger.info("Syncing SAP orders", { count: orderIds.length });
    return this.client.call(
      "Z_SYNC_ORDERS",
      {
        I_MODE: "SYNC",
        IT_ORDERS: orderIds.map(id => ({ ORDER_ID: id }))
      },
      this.credentials
    );
  }
}
```

The credentials are read from environment variables in the constructor. The automation instantiates the datasource:

```typescript
// In the automation file:
const sapDatasource = new SapErpDatasource("sap-connector", logger);
// No credentials passed here — datasource reads env vars internally
```

## Key Points

- SAP RFC is the **only datasource where credentials appear in code**. All other datasources use platform-managed credentials.
- Always read credentials from environment variables (`process.env.*`) — never hardcode them.
- Never log or include credentials in error messages.
- The constructor name must still match a connector configured on the platform.
- TABLES parameters are arrays of objects. IMPORT parameters are scalar values at the top level of the params object.
