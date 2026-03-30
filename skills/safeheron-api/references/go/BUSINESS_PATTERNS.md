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
req := api.CreateAccountRequest{
    AccountName: fmt.Sprintf("user-%s", userId),
    HiddenOnUI:  true,    // hide from App/Console UI
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
func processWebhookEvent(event map[string]interface{}) {
    eventType, _ := event["eventType"].(string)
    if eventType != "TRANSACTION_STATUS_CHANGED" {
        return
    }

    txDirection, _ := event["transactionDirection"].(string)
    txType, _ := event["transactionType"].(string)
    status, _ := event["transactionStatus"].(string)

    if txType == "NORMAL" && txDirection == "INFLOW" {
        handleDepositStatusChange(event)
    }
}
```

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | On-chain, 0 confirmations | Optionally show "pending" |
| `CONFIRMING` | >=1 confirmation | Optionally show confirmation count |
| `COMPLETED` | >=N confirmations | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules:**
- Return HTTP 200 **immediately** -- put work on a goroutine or channel
- **Never roll back status** -- if DB already has `COMPLETED`, ignore a late `CONFIRMING`
- Implement **idempotency by `txKey`**

**Anti-dust attack:** Configure minimum deposit amounts and ignore events below the threshold.

### 3-2. REST API Polling -- Fallback Path

```go
func pollTransactions(transactionApi api.TransactionApi, lastPolledMs int64) {
    req := api.ListTransactionsV2Request{
        Limit:             50,
        CreateTimeMin:     lastPolledMs,
        TransactionStatus: "COMPLETED",
    }

    var txList []api.TransactionsResponse
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
    TxKey:      stuckTxKey,
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

Chains that support speed-up: all EVM chains, BTC, BCH, DASH, LTC, DOGE, FIL, Aptos, CFX.
Chains that do NOT support speed-up: NEAR, SUI, TRON, SOL, TON.

---

## 8. AML / Risk Control Integration

### At withdrawal time:
- Run destination address through your internal risk engine
- Optionally use the Safeheron Tools API -- see [TOOLS_API.md](TOOLS_API.md)

### In the Approval Callback Service:

```go
func checkAmlRisk(amlList []interface{}) bool {
    for _, aml := range amlList {
        amlMap, _ := aml.(map[string]interface{})
        if riskLevel, _ := amlMap["riskLevel"].(string); riskLevel == "HIGH" {
            return false // reject
        }
    }
    return true // approve
}
```
