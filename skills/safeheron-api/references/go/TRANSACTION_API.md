# Transaction API Reference

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
    "github.com/google/uuid"
)
```

## Create API Instance

```go
transactionApi := api.TransactionApi{Client: sc}
```

---

## Create a Transaction

### V2 (standard — returns txKey only)

```go
req := api.CreateTransactionsRequest{
    CustomerRefId:          uuid.New().String(), // your unique ID
    CoinKey:                "ETHEREUM_ETH",
    TxAmount:               "0.01",              // string, never float
    SourceAccountKey:       accountKey,
    SourceAccountType:      "VAULT_ACCOUNT",
    DestinationAccountType: "ONE_TIME_ADDRESS",
    DestinationAddress:     "0xRecipientAddress",
    TxFeeLevel:             "MIDDLE", // "LOW" | "MIDDLE" | "HIGH"
}

var resp api.TxKeyResult
if err := transactionApi.CreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create transaction: %w", err))
}

txKey := resp.TxKey // save this -- Safeheron transaction identifier
fmt.Printf("Transaction created, txKey: %s\n", txKey)
```

### V3 (extended — returns txKey + idempotency info)

```go
var v3Resp api.CreateTransactionV3Response
if err := transactionApi.CreateTransactionsV3(req, &v3Resp); err != nil {
    panic(fmt.Errorf("failed to create transaction (V3): %w", err))
}

txKey := v3Resp.TxKey
isDuplicate := v3Resp.IdempotentRequest // true = duplicate CustomerRefId, original TxKey returned
```

**CreateTransactionV3Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | string | Safeheron transaction key |
| `CustomerRefId` | string | Your unique business ID echoed back |
| `IdempotentRequest` | bool | `true` if a duplicate `CustomerRefId` was detected — the original `TxKey` is returned instead of creating a new transaction |

### CreateTransactionsRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `CustomerRefId` | Yes | `string` | Your unique ID for idempotency (max 100 chars) |
| `CoinKey` | Yes | `string` | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `TxAmount` | Yes | `string` | Amount as string -- never use float |
| `SourceAccountKey` | Yes | `string` | Sender wallet key |
| `SourceAccountType` | Yes | `string` | `VAULT_ACCOUNT` |
| `DestinationAccountType` | Yes | `string` | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `DestinationAddress` | Cond. | `string` | Required when `DestinationAccountType=ONE_TIME_ADDRESS` |
| `DestinationAccountKey` | Cond. | `string` | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `TxFeeLevel` | No | `string` | `LOW` / `MIDDLE` / `HIGH` — Preset fee tier; takes precedence over `FeeRateDto` when both are set |
| `FeeRateDto` | No | `FeeRateDto` | Custom fee rate; used when `TxFeeLevel` is not set |
| `MaxTxFeeRate` | No | `string` | Maximum acceptable fee rate (transaction rejected if exceeded) |
| `TreatAsGrossAmount` | No | `bool` | If true, `TxAmount` is the total including fees |
| `Memo` | No | `string` | Memo field (XRP, EOS, COSMOS-family chains) |
| `DestinationTag` | No | `string` | Destination tag (XRP) |
| `IsRbf` | No | `*bool` | Enable Replace-By-Fee for Bitcoin transactions |
| `FailOnContract` | No | `*bool` | Reject if destination is a contract address (default true) |
| `FailOnAml` | No | `*bool` | Reject if destination fails AML check (default true) |
| `Nonce` | No | `int64` | Custom nonce for EVM chains |
| `SequenceNumber` | No | `int64` | Custom sequence number (Aptos, Stellar) |
| `BalanceVerifyType` | No | `string` | Balance verification mode |
| `Note` | No | `string` | Internal transaction note (max 180 chars) |
| `CustomerExt1` | No | `string` | Custom field 1 (max 255 chars) |
| `CustomerExt2` | No | `string` | Custom field 2 (max 255 chars) |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` — providing only `gasLimit` will cause the transaction to fail.

---

**FeeRateDto Fields:**

| Field | Type | Chain |
|-------|------|-------|
| `FeeRate` | `string` | Bitcoin / UTXO (sat/vByte) |
| `GasLimit` | `string` | EVM |
| `MaxPriorityFee` | `string` | EVM EIP-1559 (wei) |
| `MaxFee` | `string` | EVM EIP-1559 (wei) |
| `GasPremium` | `string` | Filecoin |
| `GasFeeCap` | `string` | Filecoin |
| `GasBudget` | `string` | Aptos |
| `GasUnitPrice` | `string` | Aptos |
| `MaxGasAmount` | `string` | Aptos |

---

## Retrieve a Single Transaction

```go
req := api.OneTransactionsRequest{
    TxKey: txKey,
    // OR: CustomerRefId: customerRefId,
}

var resp api.OneTransactionsResponse
if err := transactionApi.OneTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get transaction: %w", err))
}

fmt.Printf("Status: %s\n", resp.TransactionStatus)
fmt.Printf("Sub-status: %s\n", resp.TransactionSubStatus)
fmt.Printf("TxHash: %s\n", resp.TxHash)
```

---

## List Transactions (V2 -- Cursor-based)

```go
req := api.ListTransactionsV2Request{
    Limit:             20,
    TransactionStatus: "COMPLETED",   // optional filter
    CoinKey:           "ETHEREUM_ETH", // optional filter
}

var txList api.TransactionsResponseV2
if err := transactionApi.ListTransactionsV2(req, &txList); err != nil {
    panic(fmt.Errorf("failed to list transactions: %w", err))
}

for _, tx := range txList {
    fmt.Printf("%s %s\n", tx.TxKey, tx.TransactionStatus)
}
```

---

## Estimate Transaction Fee

```go
req := api.TransactionsFeeRateRequest{
    CoinKey:            "ETHEREUM_ETH",
    Value:              "0.01",
    SourceAccountKey:   accountKey,
    DestinationAddress: "0xRecipient",
}

var resp api.TransactionsFeeRateResponse
if err := transactionApi.TransactionFeeRate(req, &resp); err != nil {
    panic(fmt.Errorf("failed to estimate fee: %w", err))
}

fmt.Printf("Low:    %s (fee: %s)\n", resp.LowFeeRate.FeeRate, resp.LowFeeRate.Fee)
fmt.Printf("Middle: %s (fee: %s)\n", resp.MiddleFeeRate.FeeRate, resp.MiddleFeeRate.Fee)
fmt.Printf("High:   %s (fee: %s)\n", resp.HighFeeRate.FeeRate, resp.HighFeeRate.Fee)
```

**FeeRate Object Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `FeeRate` | string | Fee rate (gas price or sat/vbyte) |
| `Fee` | string | Estimated total fee amount |
| `GasLimit` | string | Gas limit (EVM chains only) |

---

## Cancel a Transaction

```go
req := api.CancelTransactionRequest{
    TxKey: txKey,
}

var resp api.ResultResponse
if err := transactionApi.CancelTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to cancel transaction: %w", err))
}
```

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

```go
req := api.RecreateTransactionRequest{
    TxKey:      originalTxKey,
    TxHash:     originalTxHash,
    CoinKey:    originalCoinKey,
    TxFeeLevel: "HIGH",
}

var resp api.TxKeyResult
if err := transactionApi.RecreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to speed up transaction: %w", err))
}

newTxKey := resp.TxKey
```

**RecreateTransactionRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | string | Original transaction key to speed up |
| `TxHash` | string | Original transaction hash |
| `CoinKey` | string | Coin key |
| `TxFeeLevel` | string | Fee level: `LOW` / `MIDDLE` / `HIGH` |
| `FeeRateDto` | FeeRateDto | Custom fee rate (either `TxFeeLevel` or `FeeRateDto`) |

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

---

## Get Transaction Approval Details

```go
req := api.ApprovalDetailTransactionsRequest{
    TxKeyList: []string{txKey},  // max 20 keys
}

var resp api.ApprovalDetailTransactionsResponse
if err := transactionApi.ApprovalDetailTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get approval details: %w", err))
}

for _, detail := range resp.ApprovalDetailList {
    fmt.Printf("txKey: %s, status: %s, policy: %s\n",
        detail.TxKey, detail.ApprovalStatus, detail.PolicyName)
}
```

**ApprovalDetailTransactionsRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `TxKeyList` | Yes | `[]string` | List of transaction keys to query (max 20) |

**ApprovalDetail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | string | Safeheron transaction key |
| `ApprovalStatus` | string | Current approval status (see below) |
| `PolicyName` | string | Name of the matched approval policy |
| `ApprovalProgress` | object | Current progress of the approval flow |

**`ApprovalStatus` Values:**

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
| UTXO chains | BTC, BCH, DASH, LTC, DOGE |
| Non-UTXO chains | All EVM chains, FIL, Aptos, CFX |
| NOT supported | NEAR, SUI, TRON, SOL, TON |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred.
- **TRON**: Fee has no LOW/MID/HIGH tiers; `sourceAddress` required for fee estimation.
- **Contract addresses**: API transfers to contract addresses require `FailOnContract = false`.
- **destinationAccountKey rules**:
  - `VAULT_ACCOUNT` -> pass `accountKey`
  - `WHITELISTING_ACCOUNT` -> pass `whitelistKey`
  - `ONE_TIME_ADDRESS` -> do NOT pass `DestinationAccountKey`; use `DestinationAddress`

---

## Best Practices

1. Always set `CustomerRefId` from your own DB record ID before calling create.
2. Store the returned `TxKey` -- it is Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call `/v1/transactions/one` periodically as fallback.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) -- never use float.
6. On timeout, **retry with the same `CustomerRefId`** -- Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
