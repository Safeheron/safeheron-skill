# Compliance (KYT / KYA) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / KYA (Know Your Address) AML providers to assess transaction and address risk.

Two main integration points:
1. **KYT** — retrieve AML risk reports for completed transactions (`kytReport`).
2. **KYA** — proactively screen a blockchain address before transacting (`createKyaScreening` / `kyaScreeningOne` / `kyaScreeningOrderOne`).

---

## Imports

```typescript
import { ComplianceApi, SafeheronConfig } from '@safeheron/api-sdk';
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
  // OR: customerRefId: 'your-ref-id',
});

console.log('Screening state:', resp.amlScreeningTriggeredState);
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

> *Either `txKey` or `customerRefId` must be provided. If both are provided, `txKey` takes precedence.

### KytReportResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `customerRefId` | string | Your reference ID |
| `amlScreeningTriggeredState` | string | `IN_PROGRESS` / `TRIGGERED` / `UNTRIGGERED` — see below |
| `amlList` | `AmlReport[]` | List of AML assessment results (empty when `UNTRIGGERED`) |

**`amlScreeningTriggeredState` values:**

| Value | Meaning |
|-------|---------|
| `IN_PROGRESS` | Evaluating — screening not yet determined; `amlList` unavailable, poll again |
| `TRIGGERED` | Screening initiated; check `amlList` for risk results |
| `UNTRIGGERED` | Screening not triggered; `amlList` is empty |

### AmlReport Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | string | AML provider name: `Elliptic`, `Chainalysis`, `MistTrack` |
| `timestamp` | string | Screening creation time (UNIX ms timestamp) |
| `status` | string | `PENDING` / `COMPLETED` / `SKIPPED` / `FAILED` |
| `riskLevel` | string | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `lastUpdateTime` | string | Last update time (UNIX ms timestamp) |
| `payload` | object | Raw provider-specific report (structure varies by provider) |

---

## Create KYA Screening Request

Initiate an address screening against one or more AML providers in parallel:

```typescript
const resp = await complianceApi.createKyaScreening({
  address: '0xRecipientAddress',
  chainType: 'ETH',
  // network: 'ethereum',  // required when providers includes MistTrack
  providers: ['Elliptic', 'Chainalysis'],
});

console.log('Screen ID:', resp.screenId);
for (const order of resp.orders) {
  console.log(`Order ID: ${order.screenOrderId}, Provider: ${order.provider}`);
}
```

### CreateKyaScreeningRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `address` | Yes | string | On-chain address to screen (not limited to Safeheron addresses) |
| `chainType` | Yes | string | Chain type — see `kyaSupportedNetworks()` for valid values |
| `network` | Conditional | string | Network identifier — required when `providers` includes `MistTrack` |
| `providers` | Yes | string[] | At least one of: `MistTrack`, `Elliptic`, `Chainalysis`; no duplicates |

### CreateKyaScreeningResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | string | Screening request ID (use for polling) |
| `address` | string | Screened address |
| `chainType` | string | Chain type |
| `network` | string | Network identifier |
| `orders` | `KyaScreeningOrder[]` | One order per provider |
| `createTime` | number | Creation time (UNIX ms timestamp) |

**KyaScreeningOrder fields:** `screenOrderId` (string), `provider` (string)

---

## Retrieve KYA Screening Summary

Poll until `status` is `FINISHED`:

```typescript
const resp = await complianceApi.kyaScreeningOne({
  screenId: 'your-screen-id',
});

console.log('Overall status:', resp.status); // PROCESSING / FINISHED
for (const order of resp.orders) {
  console.log(`  ${order.provider}: ${order.status} (risk=${order.riskLevel})`);
}
```

### KyaScreeningOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | string | Screening request ID |
| `address` | string | Screened address |
| `chainType` | string | Chain type |
| `network` | string | Network identifier |
| `status` | string | `PROCESSING` (some orders pending) / `FINISHED` (all completed) |
| `createTime` | number | Creation time (UNIX ms timestamp) |
| `orders` | `KyaScreeningOrderSummary[]` | Per-provider order summaries |

**KyaScreeningOrderSummary fields:**

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | string | Screening order ID |
| `provider` | string | Provider identifier |
| `status` | string | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | string | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | number | Completion time (UNIX ms timestamp); only set in terminal state |

---

## Retrieve KYA Screening Order Details

Get the full provider report for a single order (includes raw `payload`):

```typescript
const resp = await complianceApi.kyaScreeningOrderOne({
  screenOrderId: 'your-order-id',
});

console.log('Risk Level:', resp.riskLevel);
console.log('Address Type:', resp.addressType);
console.log('Payload:', resp.payload);
```

### KyaScreeningOrderOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | string | Screening order ID |
| `screenId` | string | Parent screening request ID |
| `provider` | string | Provider identifier |
| `address` | string | Screened address |
| `addressType` | string | `VAULT_ACCOUNT` / `WHITELISTING_ACCOUNT` / `ONE_TIME_ADDRESS` |
| `chainType` | string | Chain type |
| `network` | string | Network identifier |
| `status` | string | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | string | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | number | Completion time (UNIX ms timestamp) |
| `createTime` | number | Screening initiation time (UNIX ms timestamp) |
| `payload` | object | Provider's raw report (available after completion; structure varies by provider) |

---

## Retrieve Supported Networks & Providers

Query which chain/network combinations are supported by each AML provider:

```typescript
const networks = await complianceApi.kyaSupportedNetworks();

for (const n of networks) {
  console.log(`Network: ${n.network}, ChainType: ${n.chainType}, Providers: ${n.providers}`);
}
```

### KyaSupportedNetwork Fields

| Field | Type | Description |
|-------|------|-------------|
| `network` | string | Network identifier |
| `chainType` | string | Chain type |
| `providers` | string[] | Providers supporting this network |

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

## Full Example: Screen Address Before Sending

Use KYA screening to check a destination address before initiating any outbound transfer:

```typescript
// Step 1: Create KYA screening
const created = await complianceApi.createKyaScreening({
  address: destinationAddress,
  chainType: 'ETH',
  providers: ['Chainalysis'],
});

// Step 2: Poll until FINISHED
let result: any;
for (let i = 0; i < 30; i++) {
  result = await complianceApi.kyaScreeningOne({ screenId: created.screenId });
  if (result.status === 'FINISHED') break;
  await new Promise(resolve => setTimeout(resolve, 2000));
}

// Step 3: Block high-risk addresses
for (const order of result.orders) {
  if (order.riskLevel === 'HIGH' || order.riskLevel === 'SEVERE') {
    throw new Error(`Address is ${order.riskLevel} risk: ${destinationAddress}`);
  }
}

// Step 4: Proceed with transaction
await transactionApi.createTransactions({
  destinationAddress,
  // ... set other fields
});
```

---

## Best Practices

- Use KYA screening (`createKyaScreening`) to check an address **before** initiating a transaction. Poll `kyaScreeningOne` until `status=FINISHED`.
- Call `kyaSupportedNetworks()` to determine valid `chainType` / `network` / `provider` combinations before submitting a screening.
- `network` is required when `providers` includes `MistTrack`.
- The `failOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
- Safeheron's server timezone is UTC+0.
