# Tools API Reference (AML Address Checker)

## Overview

The Tools API provides utilities for address-level AML risk assessment *before* initiating a transaction. This is distinct from transaction-level KYT reports (see [COMPLIANCE_API.md](COMPLIANCE_API.md)).

Use Case: Check whether a recipient address is flagged as high-risk **before** sending funds.

---

## Imports

```typescript
import { ToolsApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instance

```typescript
const toolsApi = new ToolsApi(config);
```

---

## AML Address Risk Assessment (Two-Step Async Flow)

Assessment is asynchronous: submit the address, then poll for results.

### Step 1: Submit Address for AML Assessment

```typescript
const resp = await toolsApi.amlCheckerRequest({
  network: 'Ethereum',       // "Bitcoin", "Ethereum", or "Tron"
  address: '0xAbCd1234...',  // Address to assess
});

const requestId = resp.requestId;  // save this for polling
```

### AmlCheckerRequestRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `network` | Yes | string | Blockchain network: `Bitcoin`, `Ethereum`, `Tron` |
| `address` | Yes | string | Blockchain address to assess |

### AmlCheckerRequestResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | string | Async assessment request ID -- use to poll for results |

---

### Step 2: Retrieve AML Assessment Result

```typescript
const result = await toolsApi.amlCheckerRetrieves({
  requestId: requestId,
});

console.log('Risk Level:', result.mistTrack.riskLevel);
console.log('Status:', result.mistTrack.status);
```

### AmlCheckerRetrievesResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | string | The assessment request ID |
| `createTime` | string | Timestamp when the request was created (ms) |
| `network` | string | Blockchain network |
| `address` | string | Assessed address |
| `isMaliciousAddress` | boolean | Whether the address is flagged as malicious |
| `mistTrack` | object | MistTrack risk assessment result |
| `mistTrack.status` | string | Assessment status: `EVALUATING` / `SUCCESS` |
| `mistTrack.evaluationTime` | string | Time taken for evaluation (ms) |
| `mistTrack.score` | string | Risk score value |
| `mistTrack.riskLevel` | string | `Low`, `Moderate`, `High`, `Severe` |
| `mistTrack.detailList` | string[] | List of risk detail tags |
| `mistTrack.riskDetail` | object[] | Detailed risk breakdown items |

**mistTrack.riskDetail Item Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `riskType` | string | Type of risk detected |
| `entity` | string | Associated entity name |
| `hopNum` | string | Number of hops from risk source |
| `exposureType` | string | Direct or indirect exposure |
| `volume` | string | Transaction volume involved |
| `percent` | string | Percentage of risk exposure |

---

## Full Example: Check Address Before Sending

```typescript
import { ToolsApi, TransactionApi } from '@safeheron/api-sdk';
import crypto from 'crypto';

async function sendWithAmlCheck(
  toolsApi: ToolsApi,
  transactionApi: TransactionApi,
  destinationAddress: string,
  accountKey: string,
) {
  // 1. Submit address for assessment
  const submitResp = await toolsApi.amlCheckerRequest({
    network: 'Ethereum',
    address: destinationAddress,
  });
  const requestId = submitResp.requestId;

  // 2. Poll until result is ready
  let result: any = null;
  for (let i = 0; i < 30; i++) {
    result = await toolsApi.amlCheckerRetrieves({ requestId });
    if (result.mistTrack.status === 'SUCCESS') {
      break;
    }
    await new Promise(resolve => setTimeout(resolve, 2000));
  }

  // 3. Block high-risk addresses
  if (result?.mistTrack?.riskLevel === 'High' || result?.mistTrack?.riskLevel === 'Severe') {
    throw new Error('AML check failed: destination address is high-risk');
  }

  // 4. Proceed with transaction
  await transactionApi.createTransactions({
    customerRefId: crypto.randomUUID(),
    destinationAddress,
    destinationAccountType: 'ONE_TIME_ADDRESS',
    coinKey: 'ETHEREUM_ETH',
    txAmount: '0.01',
    sourceAccountKey: accountKey,
    sourceAccountType: 'VAULT_ACCOUNT',
    txFeeLevel: 'MIDDLE',
  });
}
```

---

## Supported Networks

| Network Value | Supported Chains |
|---------------|-----------------|
| `Bitcoin` | Bitcoin (BTC) |
| `Ethereum` | Ethereum, ERC-20 tokens, and EVM-compatible chains |
| `Tron` | TRON (TRX), TRC-20 tokens |

---

## Best Practices

- AML checker uses external providers -- results may take several seconds.
- For transaction-level KYT reports (after a transaction completes), use `complianceApi.kytReport()` -- see [COMPLIANCE_API.md](COMPLIANCE_API.md).
- The `failOnAml=true` flag in `createTransactions()` (default) enables automatic server-side AML blocking -- the Tools API is for manual pre-screening before sending.
- API endpoint: `POST /v1/tools/aml-checker/request` and `POST /v1/tools/aml-checker/retrieves`.
