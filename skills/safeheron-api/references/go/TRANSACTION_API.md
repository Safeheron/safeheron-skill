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
| `Memo` | No | `string` | Memo of the destination address (up to 100 characters) — only for TON and XRP Ledger. For XRP Ledger it must be a numeric string in 0–4294967295 |
| ~~`DestinationTag`~~ | No | `string` | **Deprecated** — use `Memo` instead |
| `IsRbf` | No | `*bool` | Enable Replace-By-Fee for Bitcoin transactions |
| `FailOnContract` | No | `*bool` | Reject if destination is a contract address (default true) |
| `FailOnAml` | No | `*bool` | Reject if destination fails AML check (default true) |
| `Nonce` | No | `int64` | Custom nonce for EVM chains |
| `SequenceNumber` | No | `int64` | Custom sequence number (Aptos and XRP Ledger) |
| `BalanceVerifyType` | No | `string` | Balance verification mode |
| `Note` | No | `string` | Internal transaction note (max 180 chars) |
| `CustomerExt1` | No | `string` | Custom field 1 (max 255 chars) |
| `CustomerExt2` | No | `string` | Custom field 2 (max 255 chars) |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` — providing only `gasLimit` will cause the transaction to fail.

---

**FeeRateDto Fields:**

| Field | Type | Chain / Description |
|-------|------|---------------------|
| `FeeRate` | `string` | UTXO / EVM / TRON / SUI / XRPL — fee per byte for UTXO, gas price for EVM chains, fee limit for TRON (optional), gas price for SUI; drops for XRPL (V3 only) |
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

> **Time window (published docs):** `CreateTimeMin` defaults to `CreateTimeMax` minus **24 hours** and `CreateTimeMax` defaults to now — without time bounds only the last 24 hours are returned. `Limit` is at most 500; page with `Direct` (`NEXT` by default) + `FromId` = `txKey` of the last record of the previous page. Go SDK fields are `omitempty`, so pass `CreateTimeMin: 1` (not 0) to search from the beginning.

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

> When estimating a speed-up, pass the original `TxHash`. The API also accepts a `memo` (TON / XRP Ledger; up to 100 characters, XRPL: numeric string 0–4294967295) for fee estimation, but Go SDK v1.0.15 `TransactionsFeeRateRequest` has no `Memo` field yet.

**FeeRate Object Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `FeeRate` | string | Fee rate, UTXO fee per byte, EVM gas price, TRON fee limit, SUI gas price and XRPL drops |
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

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `TxKey` | Yes | string | Original transaction key to speed up |
| `TxHash` | Yes | string | Original transaction hash |
| `CoinKey` | Yes | string | Coin key |
| `TxFeeLevel` | No | string | Fee level: `LOW` / `MIDDLE` / `HIGH`; takes priority if set |
| `FeeRateDto` | No | FeeRateDto | Custom fee rate |

> Provide at least one of `TxFeeLevel` / `FeeRateDto` (service-side validation rejects the request when both are empty).

---

## Transaction Status Reference

| Status | Description |
|--------|-------------|
| `SUBMITTED` | Received by Safeheron |
| `SIGNING` | MPC signing in progress — entered after approval by team members or API Co-Signer |
| `BROADCASTING` | Being broadcast to the blockchain network. For incoming transactions, only Bitcoin network transactions have this state (zero confirmation) |
| `CONFIRMING` | On-chain, awaiting block confirmations |
| `COMPLETED` | Confirmed on-chain — assets successfully transferred |
| `CANCELLED` | Final state — transaction cancelled by team member or system |
| `FAILED` | Final state — no assets were transferred; see `TransactionSubStatus` for the reason |
| `REJECTED` | Rejected by a team member's approval, or rejected due to a Connect recipient's approval |

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

### Transaction Substatus

`TransactionSubStatus` gives the detailed sub-state / failure reason of a transaction:

| Dictionary Code | Description |
|---|---|
| CONFIRMED | Confirmed |
| UN_KNOW_ERROR | Unexpected error |
| REVIEW_REJECTED | Rejected |
| TRADING_STRATEGY_BLOCKING | Transaction policy has been blocked |
| MATCHING_RULE_NO_AUDITOR | Matched with the policy, but the editor does not possess the approval role |
| TX_INFO_ERROR | Transaction failed |
| NONCE_TOO_LOW | Nonce too low |
| INTRINSIC_GAS_TOO_LOW | Gas limit too low, cannot perform transaction |
| INSUFFICIENT_FUNDS | Insufficient amount for withdrawal |
| AMOUNT_TOO_SMALL | The withdrawal amount is below the minimum limit |
| TRANSACTION_UNDERPRICED | Transaction fee too low, transaction is rejected by node |
| MISSING_INPUTS | Invalid input |
| INVALID_SIGNATURE | Invalid transaction signature |
| INTERNAL_ERROR | An internal error occurred while processing the transaction |
| TIMEOUT | The transaction request timed out |
| PENDING_BLOCKCHAIN_CONFIRMATIONS | Pending blockchain confirmation |
| TAPOS_ERROR | Tapos check error |
| NODE_SERVER_BUSY | Node service is busy |
| DUP_TRANSACTION_ERROR | The transaction is on-chain, and a duplicate submission has been made |
| TRANSACTION_EXPIRATION_ERROR | Transaction expired |
| TRANSACTION_EXECUTION_FAILED | Transaction execution failed |
| BANDWITH_ERROR | For Tron transactions: Insufficient bandwidth/energy or TRX. You can either increase the transaction fee or have other accounts delegate resources to this account before initiating the transaction |
| ACCOUNT_DOES_NOT_EXIST | Account doesn't exist |
| NO_MATCHING_RULE_FOUND | Transaction failed, configure transaction policy first |
| WHITELIST_ADDRESS_STATUS_ILLEGAL | The whitelisted address is not approved |
| AML_SCREENING_REJECTED | Rejected by AML check |
| CONTRACT_VALIDATE_ERROR | Transaction verification error |
| THE_DEAL_WAS_ABANDONED | Transaction abandoned |
| MANUAL_TX_FAIL | Manual processing |
| CANCELLED_BY_USER | User cancelled transaction |
| RISK_CHECK_ILLEGAL | Exceptions detected by risk control |
| WALLET_NOT_ALLOWED | The wallet doesn't support current operation |
| USER_NO_PERMISSION | User doesn't have permission |
| EXCHANGE_EMAIL_CONFIRMING | Pending email confirmation |
| EXCHANGE_UN_KNOW_ERROR | Unexpected error |
| EXCHANGE_INVALID_INFO | Invalid API key, IP, or operational permission |
| EXCHANGE_AMOUNT_OVER_LIMIT | The transfer amount exceeds the maximum |
| EXCHANGE_BALANCE_LACK | Insufficient balance |
| RESET_AUTO_CANCEL | Automatic cancellation |
| REVIEW_OTHERS_REJECTED | Rejected by the counterparty |
| TON_BOUNCE_TRANSACTION | The transaction is bounced |
| TON_SEQUENCE_INVALID | Invalid sequence |
| UTXO_DUST | Dust Transaction |
| MPC_TIMEOUT | The MPC signature timed out |
| SOL_ERROR_PROCESSING_INSTRUCTION | Transaction execution failed |
| DEST_ADDRESS_TOKEN_BLACKLISTED | The destination address has been frozen by the token issuer |
| SOURCE_ADDRESS_TOKEN_BLACKLISTED | The source address has been frozen by the token issuer |

### Transaction Subtype (`TransactionType`)

| Dictionary Code | Description |
|---|---|
| NORMAL | Normal transaction |
| UTXO_COLLECTION | UTXO-sweeping transaction |
| ABNORMAL | Transfers to external destinations (not in the Safeheron platform) or transactions signed by the original private key recovered by key shards |
| AUTO_SWEEP | Auto-sweeping transaction |
| AUTO_FUEL | Auto-fueling transaction |
| UTXO_BATCH_TRANSFER | UTXO multi-address transfer |
| BOUNCED_TRANSFER | TON's bounced transaction |
| GAS_SERVICE_DEPOSIT | Gas Service Deposit |
| GAS_SERVICE_FUEL | Gas Service Fuel |

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

Supported (per Safeheron "Speed Up Transactions"):

- EVM chains: such as ETH, ETC, BSC, MATIC, etc.
- UTXO-supported chains: Bitcoin, Dogecoin (supports RBF and CPFP) and other UTXO-based chains supporting CPFP
- FileCoin
- Aptos
- XRP Ledger

Not listed in the published docs as supported (no speed-up): e.g. NEAR, SUI, TRON, SOL, TON.

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
