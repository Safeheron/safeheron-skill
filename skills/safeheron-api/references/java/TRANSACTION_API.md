# Transaction API Reference

## Imports

```java
import com.safeheron.client.api.TransactionApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
TransactionApiService transactionApi = ServiceCreator.create(TransactionApiService.class, safeheronConfig);
```

---

## Create a Transaction

### V2 (standard — returns txKey only)

```java
CreateTransactionRequest req = new CreateTransactionRequest();
req.setCustomerRefId(UUID.randomUUID().toString()); // your unique ID
req.setCoinKey("ETHEREUM_ETH");
req.setTxAmount("0.01");                // string, never float
req.setSourceAccountKey(accountKey);
req.setSourceAccountType("VAULT_ACCOUNT");
req.setDestinationAccountType("ONE_TIME_ADDRESS");
req.setDestinationAddress("0xRecipientAddress");
req.setTxFeeLevel("MIDDLE");           // "LOW" | "MIDDLE" | "HIGH"
// Optional: req.setNote("Payment for invoice #123");
// Optional: req.setCustomerExt1("order-id-456");

TxKeyResult resp = ServiceExecutor.execute(transactionApi.createTransactions(req));
String txKey = resp.getTxKey();  // save this — Safeheron transaction identifier
```

### V3 (extended — returns txKey + idempotency info)

```java
CreateTransactionV3Response resp = ServiceExecutor.execute(
        transactionApi.createTransactionsV3(req));  // same request class
String txKey = resp.getTxKey();
```

**CreateTransactionV3Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your unique business ID echoed back |
| `idempotentRequest` | Boolean | `true` if a duplicate `customerRefId` was detected — the original `txKey` is returned instead of creating a new transaction |

### CreateTransactionRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `customerRefId` | Yes | String | Your unique ID for idempotency (max 100 chars) |
| `coinKey` | Yes | String | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `txAmount` | Yes | String | Amount as string — never use float/double |
| `sourceAccountKey` | Yes | String | Sender wallet key |
| `sourceAccountType` | Yes | String | `VAULT_ACCOUNT` |
| `destinationAccountType` | Yes | String | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `destinationAddress` | Cond. | String | Required when `destinationAccountType=ONE_TIME_ADDRESS` |
| `destinationAccountKey` | Cond. | String | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `txFeeLevel` | No | String | `LOW` / `MIDDLE` / `HIGH` (priority over feeRateDto) |
| `feeRateDto` | No | Object | Custom fee rate (requires both `gasLimit` AND `feeRate`) |
| `maxTxFeeRate` | No | String | Max acceptable fee rate |
| `note` | No | String | Transaction note (max 180 chars) |
| `customerExt1` | No | String | Custom field 1 (max 255 chars) |
| `customerExt2` | No | String | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | Boolean | If true, fee is deducted from `txAmount` |
| `memo` | No | String | Memo/tag (for TON, XRP, Stellar — max 100 chars) |
| `failOnAml` | No | Boolean | Default true — block high-risk addresses via AML |
| `failOnContract` | No | Boolean | Block contract addresses as destination |
| `nonce` | No | Long | Custom nonce for EVM chains |
| `isRbf` | No | Boolean | Enable Replace-by-Fee for Bitcoin |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` — providing only `gasLimit` will cause the transaction to fail.

---

## Retrieve a Single Transaction

```java
OneTransactionsRequest req = new OneTransactionsRequest();
req.setTxKey(txKey);
// OR: req.setCustomerRefId(customerRefId);

OneTransactionsResponse resp = ServiceExecutor.execute(transactionApi.oneTransactions(req));
System.out.println("Status: " + resp.getTransactionStatus());
System.out.println("Sub-status: " + resp.getTransactionSubStatus());
System.out.println("TxHash: " + resp.getTxHash());
```

**OneTransactionsResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `txHash` | String | On-chain transaction hash |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `txFee` | String | Actual fee paid |
| `feeCoinKey` | String | Fee coin |
| `transactionStatus` | String | See status table below |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Sender wallet key |
| `sourceAddress` | String | Sender address |
| `destinationAddress` | String | Recipient address |
| `createTime` | Long | Unix timestamp (ms) |
| `completedTime` | Long | Unix timestamp (ms) of completion |
| `customerRefId` | String | Your reference ID |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | `YES` / `NO` — AML blocked |
| `amlList` | List\<Aml\> | AML assessment results |
| `blockHeight` | Long | Block height at confirmation |
| `nonce` | String | Transaction nonce |
| `speedUpHistory` | List | Speed-up transaction history |
| `replacedTxKey` | String | Original txKey if this is a speed-up |

---

## List Transactions

### V2 (Recommended — cursor-based)

```java
ListTransactionsV2Request req = new ListTransactionsV2Request();
req.setLimit(20L);
req.setTransactionStatus("COMPLETED");  // optional filter
req.setCoinKey("ETHEREUM_ETH");       // optional filter
// req.setCreateTimeMin(startMs);     // optional timestamp filter
// req.setCreateTimeMax(endMs);

List<TransactionsResponse> txList = ServiceExecutor.execute(transactionApi.listTransactionsV2(req));
for (TransactionsResponse tx : txList) {
    System.out.println(tx.getTxKey() + " " + tx.getTransactionStatus());
}
```

**ListTransactionsV2Request Fields** (extends `LimitSearch`):

Inherited from `LimitSearch`:

| Field | Type | Description |
|-------|------|-------------|
| `direct` | String | Page direction: `NEXT` (default) |
| `limit` | Long | Items per page, max 500 |
| `fromId` | String | txKey of last record from previous page; omit for first page |

Own fields:

| Field | Type | Description |
|-------|------|-------------|
| `sourceAccountKey` | String | Source account key |
| `sourceAccountType` | String | Source account type |
| `destinationAccountKey` | String | Destination account key |
| `destinationAccountType` | String | Destination account type |
| `accountKey` | String | Query all txns under this wallet (VAULT_ACCOUNT only; overrides source/dest filters) |
| `createTimeMin` | Long | Start time, UNIX timestamp (ms) |
| `createTimeMax` | Long | End time, UNIX timestamp (ms) |
| `txAmountMin` | String | Min transaction amount |
| `txAmountMax` | String | Max transaction amount |
| `coinKey` | String | Coin key, multiple separated by commas |
| `feeCoinKey` | String | Fee coin key, multiple separated by commas |
| `transactionStatus` | String | Transaction status |
| `transactionSubStatus` | String | Transaction sub-status |
| `completedTimeMin` | Long | Min completion time, UNIX timestamp (ms) |
| `completedTimeMax` | Long | Max completion time, UNIX timestamp (ms) |
| `customerRefId` | String | Merchant unique business ID |
| `realDestinationAccountType` | String | Type of actual destination account |
| `hideSmallAmountUsd` | String | Filter out transactions below this USD amount |
| `transactionDirection` | String | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER`; default: all |

### V1 (legacy — page-based)

```java
ListTransactionsV1Request req = new ListTransactionsV1Request();
req.setPageSize(10L);
req.setPageNumber(1L);

PageResult<TransactionsResponse> result = ServiceExecutor.execute(
        transactionApi.listTransactionsV1(req));
List<TransactionsResponse> txList = result.getContent();
```

---

## Estimate Transaction Fee

```java
TransactionsFeeRateRequest req = new TransactionsFeeRateRequest();
req.setCoinKey("ETHEREUM_ETH");
req.setValue("0.01");
req.setSourceAccountKey(accountKey);
req.setDestinationAddress("0xRecipient");

// For TRON: sourceAddress is required (determines staking/energy)
// req.setSourceAddress("TsenderAddress");

TransactionsFeeRateResponse resp = ServiceExecutor.execute(
        transactionApi.transactionFeeRate(req));
System.out.println("Low:    " + resp.getLowFeeRate());
System.out.println("Middle: " + resp.getMiddleFeeRate());
System.out.println("High:   " + resp.getHighFeeRate());
```

**TransactionsFeeRateRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | String | Coin key |
| `txHash` | String | Original tx hash (for speed-up fee estimation) |
| `sourceAccountKey` | String | Source account key (required for UTXO coins) |
| `sourceAddress` | String | Source address (required for TRON; for EVM returns on-chain gas limit) |
| `destinationAddress` | String | Destination address (optional for TRON/FIL; for EVM returns on-chain gas limit) |
| `destinationAddressList` | `List<DestinationAddress>` | Destination address list |
| `value` | String | Transfer amount (improves accuracy for EVM, UTXO, SUI) |

> **BTC fee** = feeRate × bytesSize. **ETH fee** = feeRate × gasLimit.
> **TRON fee**: `sourceAddress` must be provided to check on-chain staking/energy.

---

## Cancel a Transaction

```java
CancelTransactionRequest req = new CancelTransactionRequest();
req.setTxKey(txKey);
// req.setTxType("TRANSACTION");  // default, can be omitted

ServiceExecutor.execute(transactionApi.cancelTransactions(req));
```

**CancelTransactionRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Transaction key |
| `txType` | String | Transaction type, `TRANSACTION` by default |

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

Replaces a stuck transaction with a higher fee version:

```java
RecreateTransactionRequest req = new RecreateTransactionRequest();
req.setTxKey(originalTxKey);
req.setTxFeeLevel("HIGH");  // use higher fee

TxKeyResult resp = ServiceExecutor.execute(transactionApi.recreateTransactions(req));
String newTxKey = resp.getTxKey();
```

**RecreateTransactionRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Original transaction key to speed up |
| `txHash` | String | Original transaction hash |
| `coinKey` | String | Coin key |
| `txFeeLevel` | String | Fee level: `LOW` / `MIDDLE` / `HIGH` |
| `feeRateDto` | FeeRateDto | Custom fee rate (either `txFeeLevel` or `feeRateDto`) |

> The original transaction and speed-up are independent. The speed-up will invalidate the original (original → FAILED). The speed-up transaction can be found via `speedUpHistory` in the original transaction, or `replacedTxKey` in the new one.

---

## Get Transaction Approval Details

```java
ApprovalDetailTransactionsRequest req = new ApprovalDetailTransactionsRequest();
req.setTxKeyList(Arrays.asList(txKey));

ApprovalDetailTransactionsResponse resp = ServiceExecutor.execute(
        transactionApi.approvalDetailTransactions(req));
```

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
    → SIGNING
    → BROADCASTING
    → CONFIRMING
    → COMPLETED
    └─ FAILED

    At SUBMITTED or SIGNING:
    → REJECTED (denied by approver)
    → CANCELLED (cancelled by team member)
```

### Transaction Direction (source/destination type)

| Scenario | `destinationAccountType` | `sourceAccountType` |
|----------|------------------------|-------------------|
| Withdrawal (outflow) | `ONE_TIME_ADDRESS` | `VAULT_ACCOUNT` |
| Internal transfer | `VAULT_ACCOUNT` | `VAULT_ACCOUNT` |
| Deposit (inflow) | — | `UNKNOWN` |

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create V2 | `CreateTransactionRequest` | `TxKeyResult` |
| Create V3 | `CreateTransactionRequest` | `CreateTransactionV3Response` |
| Get one | `OneTransactionsRequest` | `OneTransactionsResponse` |
| List V2 | `ListTransactionsV2Request` | `List<TransactionsResponse>` |
| List V1 | `ListTransactionsV1Request` | `PageResult<TransactionsResponse>` |
| Estimate fee | `TransactionsFeeRateRequest` | `TransactionsFeeRateResponse` |
| Cancel | `CancelTransactionRequest` | `ResultResponse` |
| Speed up | `RecreateTransactionRequest` | `TxKeyResult` |
| Approval detail | `ApprovalDetailTransactionsRequest` | `ApprovalDetailTransactionsResponse` |

---

## Chain-Specific Transaction Behaviors

### Speed-Up (Transaction Acceleration) Support

| Category | Supported Chains |
|----------|-----------------|
| UTXO chains | BTC, BCH, DASH, LTC, DOGE |
| Non-UTXO chains | All EVM chains, FIL, Aptos, CFX |
| NOT supported | NEAR, SUI, TRON, SOL, TON |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred. Max transfer = balance - 0.002 SOL.
- **Aptos**: Token transfers are NOT supported (recipient must call `register` first — not yet implemented).
- **TRON**: Fee has no LOW/MID/HIGH tiers — pre-executed on-chain; all tiers return the same fee. `sourceAddress` required for fee estimation to check staking/energy.
- **TRON negative fee**: Can occur when `bandwidth_available < 360` AND `energy_available > 360`.
- **Contract addresses**: API transfers to contract addresses require `failOnContract = false` (default `true` blocks them).
- **destinationAccountKey rules**:
  - `destinationAccountType = VAULT_ACCOUNT` → pass `accountKey`
  - `destinationAccountType = WHITELISTING_ACCOUNT` → pass `whitelistKey`
  - `destinationAccountType = ONE_TIME_ADDRESS` → do NOT pass `destinationAccountKey`; use `destinationAddress`

---

## Best Practices

1. Always set `customerRefId` from your own DB record ID before calling create.
2. Store the returned `txKey` — it's Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call `/v1/transactions/one` periodically to re-request failed deliveries.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) — never use float/double.
6. On timeout, **retry with the same `customerRefId`** — Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
