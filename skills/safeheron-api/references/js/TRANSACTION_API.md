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

### V2 (standard — returns txKey only)

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

### V3 (extended — returns txKey + idempotency info)

```typescript
const resp = await transactionApi.createTransactionsV3({
  // same request object as createTransactions
  customerRefId: crypto.randomUUID(),
  coinKey: 'ETHEREUM_ETH',
  txAmount: '0.01',
  sourceAccountKey: accountKey,
  sourceAccountType: 'VAULT_ACCOUNT',
  destinationAccountType: 'ONE_TIME_ADDRESS',
  destinationAddress: '0xRecipientAddress',
  txFeeLevel: 'MIDDLE',
});

const txKey = resp.txKey;
const isDuplicate = resp.idempotentRequest; // true = duplicate customerRefId, original txKey returned
```

**CreateTransactionV3Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `customerRefId` | string | Your unique business ID echoed back |
| `idempotentRequest` | boolean | `true` if a duplicate `customerRefId` was detected — the original `txKey` is returned instead of creating a new transaction |

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
| `txFeeLevel` | No | string | `LOW` / `MIDDLE` / `HIGH` — Choose between transaction fees. If the transaction fee rate is preset, it will take priority |
| `feeRateDto` | No | object | Transaction fee rate, either `TxFeeLevel` or `FeeRateDto`. `feeRateDto.feeRate`: fee per byte for UTXO, gas price for EVM chains, free limit for TRON (optional), gas price for SUI and **drops for XRPL** (V3; the deprecated V2 docs do not list XRPL) |
| `maxTxFeeRate` | No | string | Max acceptable fee rate |
| `note` | No | string | Transaction note (max 180 chars) |
| `customerExt1` | No | string | Custom field 1 (max 255 chars) |
| `customerExt2` | No | string | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | boolean | If true, fee is deducted from `txAmount` |
| `memo` | No | string | Memo/tag, max 100 chars. Only valid on TON and XRP Ledger; for XRP Ledger it must be a numeric string in the range 0–4294967295 |
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
| `amlLock` | string | Whether the source address contains AML address(es): `YES` (contain) / `NO` (not contain) |
| `speedUpHistory` | Array | Speed-up transaction history |

---

## List Transactions (V2 -- Cursor-based, Recommended)

> ⚠️ **Time window:** if `createTimeMin` is not provided it defaults to `createTimeMax` minus **24 hours** (`createTimeMax` defaults to now), so a call without time bounds only returns transactions created in the last 24 hours. For reconciliation or history, pass explicit `createTimeMin` / `createTimeMax` and page with `fromId` until a page returns fewer than `limit` items.

```typescript
const txList = await transactionApi.listTransactionsV2({
  limit: 20,
  transactionStatus: 'COMPLETED',   // optional filter
  coinKey: 'ETHEREUM_ETH',          // optional filter
  createTimeMin: Date.now() - 7 * 24 * 3600 * 1000,  // without it only the last 24 hours are returned
  // createTimeMax: endMs,          // default: now
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
| `createTimeMin` | number | Start time, UNIX timestamp (ms). Default: `createTimeMax` minus 24 hours |
| `createTimeMax` | number | End time, UNIX timestamp (ms). Default: current UTC time |
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
// The API also accepts `memo` (TON / XRP Ledger; XRPL: numeric string 0–4294967295) for a more accurate
// estimate, but SDK v1.0.32's TransactionsFeeRateRequest does not declare it yet.

console.log('Low:   ', resp.lowFeeRate.feeRate, '(fee:', resp.lowFeeRate.fee + ')');
console.log('Middle:', resp.middleFeeRate.feeRate, '(fee:', resp.middleFeeRate.fee + ')');
console.log('High:  ', resp.highFeeRate.feeRate, '(fee:', resp.highFeeRate.fee + ')');
```

**FeeRate Object Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `feeRate` | string | Fee rate: UTXO fee per byte, EVM gas price, TRON fee limit, SUI gas price and XRPL drops |
| `fee` | string | Estimated total fee amount |
| `gasLimit` | string | Gas limit (EVM chains only) |

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
  txHash: originalTxHash,   // required: hash of the transaction to speed up
  coinKey: 'ETHEREUM_ETH',  // required
  txFeeLevel: 'HIGH',
});

const newTxKey = resp.txKey;
```

> Provide at least one of `txFeeLevel` / `feeRateDto` (service-side validation).

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
| `REJECTED` | Rejected by a team member's approval, or rejected due to a Connect recipient's approval |

> `BROADCASTING` for incoming transactions: only Bitcoin network transactions support it (zero confirmation).

### Transaction Substatus

`transactionSubStatus` gives the detailed sub-state / failure reason:

| Substatus | Description |
|-----------|-------------|
| `CONFIRMED` | Confirmed |
| `UN_KNOW_ERROR` | Unexpected error |
| `REVIEW_REJECTED` | Rejected |
| `TRADING_STRATEGY_BLOCKING` | Transaction policy has been blocked |
| `MATCHING_RULE_NO_AUDITOR` | Matched with the policy, but the editor does not possess the approval role |
| `TX_INFO_ERROR` | Transaction failed |
| `NONCE_TOO_LOW` | Nonce too low |
| `INTRINSIC_GAS_TOO_LOW` | Gas limit too low, cannot perform transaction |
| `INSUFFICIENT_FUNDS` | Insufficient amount for withdrawal |
| `AMOUNT_TOO_SMALL` | The withdrawal amount is below the minimum limit |
| `TRANSACTION_UNDERPRICED` | Transaction fee too low, transaction is rejected by node |
| `MISSING_INPUTS` | Invalid input |
| `INVALID_SIGNATURE` | Invalid transaction signature |
| `INTERNAL_ERROR` | An internal error occurred while processing the transaction |
| `TIMEOUT` | The transaction request timed out |
| `PENDING_BLOCKCHAIN_CONFIRMATIONS` | Pending blockchain confirmation |
| `TAPOS_ERROR` | Tapos check error |
| `NODE_SERVER_BUSY` | Node service is busy |
| `DUP_TRANSACTION_ERROR` | The transaction is on-chain, and a duplicate submission has been made |
| `TRANSACTION_EXPIRATION_ERROR` | Transaction expired |
| `TRANSACTION_EXECUTION_FAILED` | Transaction execution failed |
| `BANDWITH_ERROR` | For Tron transactions: Insufficient bandwidth/energy or TRX. You can either increase the transaction fee or have other accounts delegate resources to this account before initiating the transaction |
| `ACCOUNT_DOES_NOT_EXIST` | Account doesn't exist |
| `NO_MATCHING_RULE_FOUND` | Transaction failed, configure transaction policy first |
| `WHITELIST_ADDRESS_STATUS_ILLEGAL` | The whitelisted address is not approved |
| `AML_SCREENING_REJECTED` | Rejected by AML check |
| `CONTRACT_VALIDATE_ERROR` | Transaction verification error |
| `THE_DEAL_WAS_ABANDONED` | Transaction abandoned |
| `MANUAL_TX_FAIL` | Manual processing |
| `CANCELLED_BY_USER` | User cancelled transaction |
| `RISK_CHECK_ILLEGAL` | Exceptions detected by risk control |
| `WALLET_NOT_ALLOWED` | The wallet doesn't support current operation |
| `USER_NO_PERMISSION` | User doesn't have permission |
| `EXCHANGE_EMAIL_CONFIRMING` | Pending email confirmation |
| `EXCHANGE_UN_KNOW_ERROR` | Unexpected error |
| `EXCHANGE_INVALID_INFO` | Invalid API key, IP, or operational permission |
| `EXCHANGE_AMOUNT_OVER_LIMIT` | The transfer amount exceeds the maximum |
| `EXCHANGE_BALANCE_LACK` | Insufficient balance |
| `RESET_AUTO_CANCEL` | Automatic cancellation |
| `REVIEW_OTHERS_REJECTED` | Rejected by the counterparty |
| `TON_BOUNCE_TRANSACTION` | The transaction is bounced |
| `TON_SEQUENCE_INVALID` | Invalid sequence |
| `UTXO_DUST` | Dust Transaction |
| `MPC_TIMEOUT` | The MPC signature timed out |
| `SOL_ERROR_PROCESSING_INSTRUCTION` | Transaction execution failed |
| `DEST_ADDRESS_TOKEN_BLACKLISTED` | The destination address has been frozen by the token issuer |
| `SOURCE_ADDRESS_TOKEN_BLACKLISTED` | The source address has been frozen by the token issuer |

### Transaction Subtype (`transactionType`)

| transactionType | Description |
|-----------------|-------------|
| `NORMAL` | Normal transaction |
| `UTXO_COLLECTION` | UTXO-sweeping transaction |
| `ABNORMAL` | Transfers to external destinations (not in the Safeheron platform) or transactions signed by the original private key recovered by key shards |
| `AUTO_SWEEP` | Auto-sweeping transaction |
| `AUTO_FUEL` | Auto-fueling transaction |
| `UTXO_BATCH_TRANSFER` | UTXO multi-address transfer |
| `BOUNCED_TRANSFER` | TON's bounced transaction |
| `GAS_SERVICE_DEPOSIT` | Gas Service Deposit |
| `GAS_SERVICE_FUEL` | Gas Service Fuel |

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

## Get Transaction Approval Details

```typescript
const resp = await transactionApi.approvalDetailTransactions({
  txKeyList: [txKey],  // max 20 keys
});

for (const detail of resp.approvalDetailList) {
  console.log(detail.txKey, detail.approvalStatus, detail.policyName);
}
```

**ApprovalDetailTransactionsRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKeyList` | Yes | Array\<string\> | List of transaction keys to query (max 20) |

**ApprovalDetail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `approvalStatus` | string | Current approval status (see below) |
| `policyName` | string | Name of the matched approval policy |
| `approvalProgress` | object | Approval progress details |

**`approvalStatus` Values:**

| Value | Description |
|-------|-------------|
| `PENDING_APPROVAL` | Awaiting approver action |
| `APPROVED` | Approved — transaction proceeds to signing |
| `REJECTED` | Denied by an approver |
| `CANCELLED` | Cancelled before completion |
| `BLOCKED_BY_POLICY` | Blocked by a policy rule |
| `FAILED` | Approval process failed |

> **Note:** Results exclude transactions using old policies, non-existent txKeys, and incoming fund transactions.

---

## Chain-Specific Transaction Behaviors

### Speed-Up Support

| Category | Supported Chains |
|----------|-----------------|
| EVM chains | e.g. ETH, ETC, BSC, MATIC |
| UTXO chains | Bitcoin, Dogecoin (RBF and CPFP) and other UTXO-based chains supporting CPFP |
| Other | Filecoin, Aptos, XRP Ledger |
| NOT supported (not listed in the published docs as supported) | NEAR, SUI, TRON, SOL, TON |

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
