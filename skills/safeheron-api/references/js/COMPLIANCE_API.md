# Compliance (KYT / AML) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / AML (Anti-Money Laundering) providers to assess transaction risk.

Two main integration points:
1. **ComplianceApi** -- retrieve KYT risk reports for completed transactions.
2. **ToolsApi (AML Checker)** -- proactively assess a blockchain address before transacting. See [TOOLS_API.md](TOOLS_API.md).

---

## Imports

```typescript
import { ComplianceApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instance

```typescript
const complianceApi = new ComplianceApi(config);
```

---

## Retrieve KYT Risk Report for a Transaction

After a transaction completes, retrieve its AML/KYT risk assessment:

```typescript
const resp = await complianceApi.kytReport({
  txKey: 'your-tx-key',
  // Optional: customerRefId: 'your-ref-id',
});

for (const aml of resp.amlList) {
  console.log('Provider:', aml.provider);
  console.log('Risk Level:', aml.riskLevel);
  console.log('Status:', aml.status);
}
```

### KytReportRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKey` | Yes* | string | Safeheron transaction key |
| `customerRefId` | Yes* | string | Your reference ID (max 100 chars) |

> *Either `txKey` or `customerRefId` must be provided.

### KytReportResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `customerRefId` | string | Your reference ID |
| `amlList` | Array | List of AML assessment results |

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | string | AML provider name (e.g. "Elliptic", "Chainalysis") |
| `timestamp` | string | Assessment timestamp (milliseconds) |
| `status` | string | Assessment status |
| `riskLevel` | string | Risk level: `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `lastUpdateTime` | string | Last update timestamp |
| `payload` | object | Raw provider-specific data |

---

## AML Lock Behavior

When `failOnAml=true` (default) in `createTransactions()`, Safeheron will:
1. Automatically run AML checks on the destination address.
2. Block the transaction if the address scores high-risk (`amlLock = "YES"`).

To disable AML blocking (not recommended for production):

```typescript
await transactionApi.createTransactions({
  // ...
  failOnAml: false,  // disable AML lock -- use with caution
});
```

The transaction response `amlLock` field indicates `"YES"` or `"NO"`.

---

## AML Risk Assessment in Transaction Response

The transaction response (from list/query transaction APIs) includes inline AML data:

```typescript
const tx = await transactionApi.oneTransactions({ txKey });
for (const aml of tx.amlList) {
  console.log(`${aml.provider} risk: ${aml.riskLevel}`);
}
console.log('AML locked:', tx.amlLock);  // "YES" or "NO"
```

---

## Best Practices

- For address-level risk checks *before* initiating a transaction, use the Tools AML Checker -- see [TOOLS_API.md](TOOLS_API.md).
- Safeheron's server timezone is UTC+0.
- The `failOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
