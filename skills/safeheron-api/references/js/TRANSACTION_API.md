# Transaction API Reference

## Imports

```typescript
import { TransactionApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
import crypto from 'crypto';
```

## Create API Instance

```typescript
const transactionApi = new TransactionApi(config);
```

---

## Create a Transaction

```typescript
const resp = await transactionApi.createTransactions({
  customerRefId: crypto.randomUUID(),   // your unique ID
  coinKey: 'ETHEREUM_ETH',
  txAmount: '0.01',                     // string, never float
  sourceAccountKey: accountKey,
  sourceAccountType: 'VAULT_ACCOUNT',
  destinationAccountType: 'ONE_TIME_ADDRESS',
  destinationAddress: '0xRecipientAddress',
  txFeeLevel: 'MIDDLE',               // "LOW" | "MIDDLE" | "HIGH"
  // Optional: note: 'Payment for invoice #123',
  // Optional: customerExt1: 'order-id-456',
});

const txKey = resp.txKey;  // save this -- Safeheron transaction identifier
```

### CreateTransactionRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `customerRefId` | Yes | string | Your unique ID for idempotency (max 100 chars) |
| `coinKey` | Yes | string | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `txAmount` | Yes | string | Amount as string -- never use float/double |
| `sourceAccountKey` | Yes | string | Sender wallet key |
| `sourceAccountType` | Yes | string | `VAULT_ACCOUNT` |
| `destinationAccountType` | Yes | string | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `destinationAddress` | Cond. | string | Required when `destinationAccountType=ONE_TIME_ADDRESS` |
| `destinationAccountKey` | Cond. | string | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `txFeeLevel` | No | string | `LOW` / `MIDDLE` / `HIGH` (priority over feeRateDto) |
| `feeRateDto` | No | object | Custom fee rate (requires both `gasLimit` AND `feeRate`) |
| `maxTxFeeRate` | No | string | Max acceptable fee rate |
| `note` | No | string | Transaction note (max 180 chars) |
| `customerExt1` | No | string | Custom field 1 (max 255 chars) |
| `customerExt2` | No | string | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | boolean | If true, fee is deducted from `txAmount` |
| `memo` | No | string | Memo/tag (for TON, XRP, Stellar -- max 100 chars) |
| `failOnAml` | No | boolean | Default true -- block high-risk addresses via AML |
| `failOnContract` | No | boolean | Block contract addresses as destination |
| `nonce` | No | number | Custom nonce for EVM chains |
| `isRbf` | No | boolean | Enable Replace-by-Fee for Bitcoin |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` -- providing only `gasLimit` will cause the transaction to fail.

---

## Retrieve a Single Transaction

```typescript
const resp = await transactionApi.oneTransactions({
  txKey: txKey,
  // OR: customerRefId: customerRefId,
});

console.log('Status:', resp.transactionStatus);
console.log('Sub-status:', resp.transactionSubStatus);
console.log('TxHash:', resp.txHash);
```

**OneTransactionsResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `txHash` | string | On-chain transaction hash |
| `coinKey` | string | Coin identifier |
| `txAmount` | string | Transaction amount |
| `txFee` | string | Actual fee paid |
| `transactionStatus` | string | See status table below |
| `transactionSubStatus` | string | Sub-status |
| `sourceAccountKey` | string | Sender wallet key |
| `sourceAddress` | string | Sender address |
| `destinationAddress` | string | Recipient address |
| `createTime` | number | Unix timestamp (ms) |
| `completedTime` | number | Unix timestamp (ms) of completion |
| `customerRefId` | string | Your reference ID |
| `amlLock` | string | `YES` / `NO` -- AML blocked |
| `speedUpHistory` | Array | Speed-up transaction history |

---

## List Transactions (V2 -- Cursor-based, Recommended)

```typescript
const txList = await transactionApi.listTransactionsV2({
  limit: 20,
  transactionStatus: 'COMPLETED',   // optional filter
  coinKey: 'ETHEREUM_ETH',          // optional filter
  // createTimeMin: startMs,
  // createTimeMax: endMs,
});

for (const tx of txList) {
  console.log(tx.txKey, tx.transactionStatus);
}
```

**ListTransactionsV2Request Fields (cursor-based):**

| Field | Type | Description |
|-------|------|-------------|
| `direct` | string | Page direction: `NEXT` (default) |
| `limit` | number | Items per page, max 500 |
| `fromId` | string | txKey of last record from previous page; omit for first page |
| `sourceAccountKey` | string | Source account key |
| `coinKey` | string | Coin key, multiple separated by commas |
| `transactionStatus` | string | Transaction status filter |
| `createTimeMin` | number | Start time, UNIX timestamp (ms) |
| `createTimeMax` | number | End time, UNIX timestamp (ms) |
| `transactionDirection` | string | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER`; default: all |

---

## Estimate Transaction Fee

```typescript
const resp = await transactionApi.transactionFeeRate({
  coinKey: 'ETHEREUM_ETH',
  value: '0.01',
  sourceAccountKey: accountKey,
  destinationAddress: '0xRecipient',
});

console.log('Low:', resp.lowFeeRate);
console.log('Middle:', resp.middleFeeRate);
console.log('High:', resp.highFeeRate);
```

> **BTC fee** = feeRate x bytesSize. **ETH fee** = feeRate x gasLimit.
> **TRON fee**: `sourceAddress` must be provided to check on-chain staking/energy.

---

## Cancel a Transaction

```typescript
await transactionApi.cancelTransactions({
  txKey: txKey,
});
```

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

```typescript
const resp = await transactionApi.recreateTransactions({
  txKey: originalTxKey,
  txFeeLevel: 'HIGH',
});

const newTxKey = resp.txKey;
```

> The original transaction and speed-up are independent. The speed-up will invalidate the original (original -> FAILED).

---

## Transaction Status Reference

| Status | Description |
|--------|-------------|
| `SUBMITTED` | Received by Safeheron |
| `SIGNING` | MPC signing in progress — entered after approval by team members or API Co-Signer |
| `BROADCASTING` | Submitted to blockchain |
| `CONFIRMING` | On-chain, awaiting block confirmations |
| `COMPLETED` | Confirmed on-chain — assets successfully transferred |
| `CANCELLED` | Final state — transaction cancelled by team member or system |
| `FAILED` | Transaction failed |
| `REJECTED` | Rejected by approver |

### Status Flow

```
SUBMITTED
    -> SIGNING
    -> BROADCASTING
    -> CONFIRMING
    -> COMPLETED
    |-- FAILED

    At SUBMITTED or SIGNING:
    -> REJECTED (denied by approver)
    -> CANCELLED (cancelled by team member)
```

### Transaction Direction

| Scenario | `destinationAccountType` | `sourceAccountType` |
|----------|------------------------|-------------------|
| Withdrawal (outflow) | `ONE_TIME_ADDRESS` | `VAULT_ACCOUNT` |
| Internal transfer | `VAULT_ACCOUNT` | `VAULT_ACCOUNT` |
| Deposit (inflow) | -- | `UNKNOWN` |

---

## Chain-Specific Transaction Behaviors

### Speed-Up Support

| Category | Supported Chains |
|----------|-----------------|
| UTXO chains | BTC, BCH, DASH, LTC, DOGE |
| Non-UTXO chains | All EVM chains, FIL, Aptos, CFX |
| NOT supported | NEAR, SUI, TRON, SOL, TON |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred.
- **TRON**: Fee has no LOW/MID/HIGH tiers -- pre-executed on-chain; all tiers return the same fee.
- **Contract addresses**: API transfers to contract addresses require `failOnContract = false`.
- **destinationAccountKey rules**:
  - `VAULT_ACCOUNT` -> pass `accountKey`
  - `WHITELISTING_ACCOUNT` -> pass `whitelistKey`
  - `ONE_TIME_ADDRESS` -> do NOT pass `destinationAccountKey`; use `destinationAddress`

---

## Best Practices

1. Always set `customerRefId` from your own DB record ID before calling create.
2. Store the returned `txKey` -- it's Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call `oneTransactions()` periodically as fallback.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) -- never use float/double.
6. On timeout, **retry with the same `customerRefId`** -- Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
