# Business Integration Patterns

Practical patterns for building a crypto custody / exchange system on top of Safeheron. Covers deposit detection, withdrawal flow, asset consolidation, and transaction acceleration.

---

## 1. Wallet Architecture Concepts

### Wallet Account Structure

Each Safeheron wallet account is a **full-chain wallet** -- it holds one address per coin (except UTXO chains like BTC which support multiple addresses per account).

```
Wallet Account 1          Wallet Account 2          Wallet Account ...
    +-- BTC Address           +-- ETH Address           ...
    +-- ETH Address           +-- USDT-ERC20 Address
    +-- USDT-ERC20 Address    +-- ...
```

**Recommendation: bind each end-user to exactly one Safeheron wallet account (1:1).**

### Transaction Directions

| Direction | Definition | Example |
|-----------|-----------|---------|
| `INFLOW` | External -> team wallet | User deposit |
| `OUTFLOW` | Team wallet -> external | User withdrawal |
| `INTERNAL_TRANSFER` | Team wallet -> team wallet | Consolidation, gas top-up |

---

## 2. Allocating Deposit Addresses to Users

### Step 1 -- Create wallet account, bind to user

```go
h := true
req := api.CreateAccountRequest{
    AccountName: fmt.Sprintf("user-%s", userId),
    HiddenOnUI:  &h,    // hide from App/Console UI
    AccountTag:  "DEPOSIT", // required for Auto-Sweep
}

var resp api.CreateAccountResponse
if err := accountApi.CreateAccount(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create account: %w", err))
}
accountKey := resp.AccountKey // store: bind accountKey <-> userId in your DB
```

### Step 2 -- Add coin, get deposit address

```go
addCoinReq := api.AddCoinRequest{
    AccountKey: accountKey,
    CoinKey:    "ETHEREUM_ETH",
}
var addCoinResp api.AddCoinResponse
if err := accountApi.AddCoin(addCoinReq, &addCoinResp); err != nil {
    panic(fmt.Errorf("failed to add coin: %w", err))
}
depositAddress := addCoinResp[0].Address // show this to the user
```

---

## 3. Deposit Detection (User Credit)

Use **Webhook as primary + REST API polling as fallback**. Never rely on only one.

### 3-1. Webhook -- Primary Path

Subscribe to `TRANSACTION_STATUS_CHANGED`. Filter for deposit (inflow) transactions:

```go
// event = decrypted bizContent: {"eventType": ..., "eventDetail": {...}} — transaction fields are under eventDetail
func processWebhookEvent(event map[string]interface{}) {
    eventType, _ := event["eventType"].(string)
    if eventType != "TRANSACTION_STATUS_CHANGED" {
        return
    }
    detail, ok := event["eventDetail"].(map[string]interface{})
    if !ok {
        log.Printf("ERROR: TRANSACTION_STATUS_CHANGED without eventDetail")
        return
    }

    txDirection, _ := detail["transactionDirection"].(string)
    txType, _ := detail["transactionType"].(string)

    switch {
    case txType == "NORMAL" && txDirection == "INFLOW":
        handleDepositStatusChange(detail)
    case txDirection == "INTERNAL_TRANSFER":
        // Team wallet -> team wallet. A user withdrawing to ANOTHER user's deposit address in your own
        // platform is INTERNAL_TRANSFER, not OUTFLOW + INFLOW: settle the withdrawal order AND credit the receiver here
        handleInternalTransfer(detail)
    }
}
```

**Which address was credited:** for UTXO coins (e.g. BTC) use `destinationAddressList[]` (`address` / `amount`) — `destinationAddress` may be empty; for account-model coins use `destinationAddress` + `txAmount`.

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | Being broadcast, 0 confirmations — for deposits, only Bitcoin network transactions have this state | Optionally show "pending" |
| `CONFIRMING` | >=1 confirmation | Optionally show confirmation count |
| `COMPLETED` | >=N confirmations | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules:**
- Verify the signature and **durably** store the event (DB inbox / durable MQ), then ack `{"code":"200","message":"SUCCESS"}`; process asynchronously -- see [WEBHOOK.md](WEBHOOK.md). Never ack after only an in-memory enqueue.
- **Never roll back status** -- if DB already has `COMPLETED`, ignore a late `CONFIRMING`
- Implement **idempotency by `txKey`** — write the idempotency marker and the credit in the **same DB transaction**

**Anti-dust attack:** Configure minimum deposit amounts and ignore events below the threshold.

### 3-2. REST API Polling -- Fallback Path

```go
func pollTransactions(transactionApi api.TransactionApi, lastPolledMs int64) {
    req := api.ListTransactionsV2Request{
        Limit:             50,
        CreateTimeMin:     lastPolledMs,
        TransactionStatus: "COMPLETED",
    }

    var txList api.TransactionsResponseV2
    if err := transactionApi.ListTransactionsV2(req, &txList); err != nil {
        log.Printf("Failed to poll transactions: %v", err)
        return
    }

    for _, tx := range txList {
        if tx.TransactionDirection == "INFLOW" && tx.TransactionStatus == "COMPLETED" {
            creditUserIfNotAlready(tx.TxKey, tx)
        }
    }
}
```

---

## 4. Asset Consolidation (Sweep) & Gas Top-Up

### Option A -- Auto-Sweep (Recommended, Zero Code)

Configure in Safeheron Console -> Auto-Sweep. Requirements:
- API Co-Signer must be deployed
- Deposit wallets must have `AccountTag = "DEPOSIT"`
- Configure sweep rules in Console

### Option B -- Manual Sweep via API

```go
req := api.CreateTransactionsRequest{
    CustomerRefId:          uuid.New().String(),
    CoinKey:                "USDT(ERC20)_ETHEREUM_USDT",
    TxAmount:               sweepAmount,
    SourceAccountKey:       depositWalletAccountKey,
    SourceAccountType:      "VAULT_ACCOUNT",
    DestinationAccountType: "VAULT_ACCOUNT",
    DestinationAccountKey:  hotWalletAccountKey,
    TxFeeLevel:             "MIDDLE",
}

var resp api.TxKeyResult
if err := transactionApi.CreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to sweep: %w", err))
}
```

---

## 5. User Withdrawal (Outflow) -- Recommended Flow

### Architecture: Async with DB-first pattern

```
Frontend -> [1. Save withdrawal order to DB, status=PENDING]
         -> [Return "request received" to user]

Background Job -> [2. Fetch PENDING orders from DB]
               -> [3. Call Safeheron CreateTransactions, get txKey]
               -> [4. Update order: txKey, status=SUBMITTED]

Webhook Handler -> [5. Receive TRANSACTION_STATUS_CHANGED]
                -> [6. Update order status by txKey/customerRefId]
                -> [7. Notify user of completion]
```

### Generate and store customerRefId before calling Safeheron

```go
// Step 1: save order first
customerRefId := uuid.New().String()
saveWithdrawalOrder(userId, amount, toAddress, customerRefId, "PENDING")

// Step 2: submit to Safeheron
req := api.CreateTransactionsRequest{
    CustomerRefId:          customerRefId,
    CoinKey:                coinKey,
    TxAmount:               amount,
    SourceAccountKey:       hotWalletAccountKey,
    SourceAccountType:      "VAULT_ACCOUNT",
    DestinationAccountType: "ONE_TIME_ADDRESS",
    DestinationAddress:     toAddress,
    TxFeeLevel:             "MIDDLE",
}

var resp api.TxKeyResult
err := transactionApi.CreateTransactions(req, &resp)
if err != nil {
    // Check if duplicate customerRefId (error 9001)
    // Query by customerRefId to get the original txKey
    log.Printf("Failed to submit withdrawal: %v", err)
    return
}
updateWithdrawalOrderTxKey(customerRefId, resp.TxKey, "SUBMITTED")
```

### Withdrawal status sync via Webhook

```go
func handleWithdrawalWebhook(event map[string]interface{}) {
    txKey, _ := event["txKey"].(string)
    customerRefId, _ := event["customerRefId"].(string)
    newStatus, _ := event["transactionStatus"].(string)

    order := findOrderByTxKeyOrCustomerRefId(txKey, customerRefId)
    if order == nil {
        return // unknown tx
    }

    // Never roll back status
    if isTerminalStatus(order.Status) {
        return
    }

    // Speed-up: the original tx ends FAILED while its replacement may still complete — keep the order open and let the replacement decide
    // (check BEFORE moving the order to a terminal state, otherwise the replacement's COMPLETED event is discarded)
    if (newStatus == "FAILED" || newStatus == "REJECTED" || newStatus == "CANCELLED") && hasActiveReplacement(order, txKey) {
        return
    }

    order.Status = newStatus
    saveOrder(order)

    switch newStatus {
    case "COMPLETED":
        notifyUserWithdrawalCompleted(order)
    case "FAILED", "REJECTED", "CANCELLED":
        refundUserBalance(order)
        notifyUserWithdrawalFailed(order)
    }
}

func isTerminalStatus(status string) bool {
    switch status {
    case "COMPLETED", "FAILED", "REJECTED", "CANCELLED":
        return true
    }
    return false
}
```

---

## 6. Small Auto / Large Manual -- Approval Tier Pattern

| Amount Range (24H cumulative) | Approval |
|-------------------------------|---------|
| Single tx <= 100,000 | API Co-Signer auto-approve |
| Single tx > 100,000 | Ops team 2-of-3 |
| 24H total 100K-500K | Ops team 2-of-3 |
| 24H total 500K-2M | Finance team 2-of-2 |
| 24H total > 2M | Executive 1-of-2 |

See [POLICY_STRATEGY.md](POLICY_STRATEGY.md) for the complete configuration reference.

---

## 7. Transaction Acceleration (Speed-Up)

```go
req := api.RecreateTransactionRequest{
    TxKey:      stuckTxKey,    // required
    TxHash:     stuckTxHash,   // required — hash of the stuck transaction
    CoinKey:    stuckCoinKey,  // required
    TxFeeLevel: "HIGH",
}

var resp api.TxKeyResult
if err := transactionApi.RecreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to speed up: %w", err))
}
newTxKey := resp.TxKey
// The original tx will eventually reach FAILED status
// Track the speed-up tx via newTxKey
```

Chains that support speed-up: EVM chains (e.g. ETH, ETC, BSC, MATIC); UTXO chains — Bitcoin and Dogecoin (RBF and CPFP) and other UTXO chains supporting CPFP; FileCoin; Aptos; XRP Ledger.
Chains not listed in the published docs as supported do not support speed-up (e.g. NEAR, SUI, TRON, SOL, TON).

---

## 8. AML / Risk Control Integration

### At withdrawal time:
- Run destination address through your internal risk engine
- Screen the destination address with **KYA** (`ComplianceApi.KyaScreeningCreate` → poll `KyaScreeningOne`) -- see [COMPLIANCE_API.md](COMPLIANCE_API.md). Do **not** use the deprecated AML Checker (`/v1/tools/aml-checker/*`, `ToolsApi.AmlCheckerRequest/AmlCheckerRetrieves`), and `CheckAml` in `CheckCoinAddress` is not a substitute for KYA
- Reject withdrawals to `HIGH` / `SEVERE` (and, by default, `UNKNOWN`) risk addresses before they reach Safeheron

### In the Approval Callback Service:

Use `evaluateAml(approvalId, tx)` in the COSIGNER.md handler ([COSIGNER.md](COSIGNER.md#aml--kyt-handling-in-the-callback)):
- Only `riskLevel` `LOW` / `MEDIUM` pass by default — `HIGH`, `SEVERE`, `UNKNOWN` and unexpected values are rejected
- While `amlScreeningTriggeredState = IN_PROGRESS` or any `status = PENDING`, return neither APPROVE nor REJECT (Co-Signer retries every 5s); fall back to reject / manual review ~5 min after the first callback for that approval
- `status = FAILED`, a missing / unknown status, or `TRIGGERED` with an empty `amlList` → reject and alert
