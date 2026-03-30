# Business Integration Patterns

Practical patterns for building a crypto custody / exchange system on top of Safeheron. Covers deposit detection, withdrawal flow, asset consolidation, and transaction acceleration.

---

## 1. Wallet Architecture Concepts

### Wallet Account Structure

Each Safeheron wallet account is a **full-chain wallet** — it holds one address per coin (except UTXO chains like BTC which support multiple addresses per account).

```
Wallet Account 1          Wallet Account 2          Wallet Account ...
    ├── BTC Address           ├── ETH Address           ...
    ├── ETH Address           ├── USDT-ERC20 Address
    └── USDT-ERC20 Address    └── ...
```

**Recommendation: bind each end-user to exactly one Safeheron wallet account (1:1).**

### Transaction Directions

From the team's perspective, every transaction is one of three directions:

| Direction | Definition | Example |
|-----------|-----------|---------|
| `INFLOW` | External → team wallet (`sourceType` ≠ `VAULT_ACCOUNT`, `destType` = `VAULT_ACCOUNT`) | User deposit |
| `OUTFLOW` | Team wallet → external (`sourceType` = `VAULT_ACCOUNT`, `destType` ≠ `VAULT_ACCOUNT`) | User withdrawal |
| `INTERNAL_TRANSFER` | Team wallet → team wallet (both = `VAULT_ACCOUNT`) | Consolidation, gas top-up |

---

## 2. Allocating Deposit Addresses to Users

Two steps:

### Step 1 — Create wallet account, bind to user

```java
// POST /v2/account/batch/create
CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("user-" + userId);
req.setHiddenOnUI(true);          // hide from App/Console UI — user deposit wallets should be hidden
req.setAccountTag("DEPOSIT");     // required for Auto-Sweep to process this wallet automatically

CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));
String accountKey = resp.getAccountKey();  // store: bind this accountKey ↔ userId in your DB
```

**`hiddenOnUI: true`** — prevents clutter in the Safeheron UI.

**`accountTag: "DEPOSIT"`** — required if you use Auto-Sweep (the sweep engine only targets wallets tagged `DEPOSIT`).

### Step 2 — Add coin, get deposit address

```java
// POST /v2/account/coin/create
CreateAccountCoinV2Request coinReq = new CreateAccountCoinV2Request();
coinReq.setAccountKey(accountKey);
coinReq.setCoinKeyList(Arrays.asList("ETHEREUM_ETH", "USDT(ERC20)_ETHEREUM_USDT"));

CreateAccountCoinV2Response res = ServiceExecutor.execute(accountApi.createAccountCoinV2(coinReq));
String depositAddress = res.getCoinAddressList().get(0).getAddressList().get(0).getAddress(); // show this to the user as their deposit address
```

---

## 3. Deposit Detection (User Credit)

Use **Webhook as primary + REST API polling as fallback**. Never rely on only one.

### 3-1. Webhook — Primary Path

Subscribe to `TRANSACTION_STATUS_CHANGED`. Filter for deposit (inflow) transactions:

```java
// In your webhook handler:
String eventType = event.get("eventType").asText();
if (!"TRANSACTION_STATUS_CHANGED".equals(eventType)) return;

String txType      = event.get("transactionType").asText();       // "NORMAL"
String txDirection = event.get("transactionDirection").asText();  // "INFLOW"
String status      = event.get("transactionStatus").asText();

if ("NORMAL".equals(txType) && "INFLOW".equals(txDirection)) {
    handleDepositStatusChange(event);
}
```

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | On-chain, 0 confirmations | Optionally show "pending" to user |
| `CONFIRMING` | ≥1 confirmation | Optionally show confirmation count |
| `COMPLETED` | ≥N confirmations (Safeheron default) | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules for webhook handlers:**
- Return HTTP 200 **immediately** before any processing — put work on an async queue
- **Never roll back status** — if DB already has `COMPLETED`, ignore a late `CONFIRMING` event
- `BROADCASTING` and `CONFIRMING` may be skipped — handle a direct jump to `COMPLETED`
- Implement **idempotency by `txKey`** — Safeheron may deliver the same event more than once

**Anti-dust attack:** Configure a minimum deposit amount in your system and ignore webhook events below that threshold. This filters address-pollution / dusting attacks.

### 3-2. REST API Polling — Fallback Path

Poll the transaction list periodically to catch any missed webhooks:

```java
// Run every few minutes as a background job
ListTransactionsV2Request req = new ListTransactionsV2Request();
req.setLimit(50L);
req.setCreateTimeMin(lastPolledTimestampMs);
req.setTransactionStatus("COMPLETED");   // or omit to get all statuses

List<TransactionsResponse> txList = ServiceExecutor.execute(
        transactionApi.listTransactionsV2(req));

for (TransactionsResponse tx : txList) {
    if ("INFLOW".equals(tx.getTransactionDirection())
            && "COMPLETED".equals(tx.getTransactionStatus())) {
        creditUserIfNotAlready(tx.getTxKey(), tx);
    }
}
```

---

## 4. Asset Consolidation (Sweep) & Gas Top-Up

After users deposit, the platform consolidates funds from individual deposit wallets into hot wallets.

```
User Hot Wallet 1 ──→ (add gas ETH) ──→ Sweep USDT ──→ Hot Wallet
User Hot Wallet 2 ──→ (add gas ETH) ──→ Sweep USDT ──┘
```

### Option A — Auto-Sweep (Recommended, Zero Code)

Configure in Safeheron Console → Auto-Sweep. Requirements:
- API Co-Signer must be deployed
- Deposit wallets must have `accountTag = "DEPOSIT"` set at creation time
- Configure sweep rules (coin, threshold, destination wallet) in Console

Auto-Sweep handles gas top-up (加油) and sweep (归集) automatically.

### Option B — Manual Sweep via API

If you implement sweeping yourself:

1. Listen for `COMPLETED` inflow transactions via webhook
2. Check if the wallet has enough gas for the fee (use Gas Station API if gas is insufficient)
3. Create an `INTERNAL_TRANSFER` transaction from the deposit wallet to the hot wallet

```java
// Internal transfer to hot wallet
CreateTransactionRequest req = new CreateTransactionRequest();
req.setCustomerRefId(UUID.randomUUID().toString());  // store this before calling API
req.setCoinKey("USDT(ERC20)_ETHEREUM_USDT");
req.setTxAmount(sweepAmount);
req.setSourceAccountKey(depositWalletAccountKey);
req.setSourceAccountType("VAULT_ACCOUNT");
req.setDestinationAccountType("VAULT_ACCOUNT");
req.setDestinationAccountKey(hotWalletAccountKey);
req.setTxFeeLevel("MIDDLE");

TxKeyResult resp = ServiceExecutor.execute(transactionApi.createTransactions(req));
```

---

## 5. User Withdrawal (Outflow) — Recommended Flow

### Architecture: Async with DB-first pattern

```
Frontend → [1. Save withdrawal order to DB, status=PENDING]
         → [Return "request received" to user]

Background Job → [2. Fetch PENDING orders from DB]
               → [3. Call Safeheron createTransaction, get txKey]
               → [4. Update order: txKey, status=SUBMITTED]

Webhook Handler → [5. Receive TRANSACTION_STATUS_CHANGED]
                → [6. Update order status by txKey/customerRefId]
                → [7. Notify user of completion]
```

### Key principles:

**Pre-validate before submitting:**
- Check user balance in your system
- Validate withdrawal address format
- Apply risk/compliance checks
- Check destination against whitelist if required

**Generate and store `customerRefId` before calling Safeheron:**

```java
// Step 1: save order first
String customerRefId = UUID.randomUUID().toString();
withdrawalOrderDao.save(new WithdrawalOrder(userId, amount, toAddress, customerRefId, "PENDING"));

// Step 2: submit to Safeheron
CreateTransactionRequest req = new CreateTransactionRequest();
req.setCustomerRefId(customerRefId);   // idempotency key — reuse on retry
req.setCoinKey(coinKey);
req.setTxAmount(amount);
req.setSourceAccountKey(hotWalletAccountKey);
req.setSourceAccountType("VAULT_ACCOUNT");
req.setDestinationAccountType("ONE_TIME_ADDRESS");
req.setDestinationAddress(toAddress);
req.setTxFeeLevel("MIDDLE");
req.setFailOnAml(true);        // default: block AML-flagged destinations
req.setFailOnContract(true);   // default: block contract address destinations (set false only if your use case requires it)

try {
    TxKeyResult resp = ServiceExecutor.execute(transactionApi.createTransactions(req));
    withdrawalOrderDao.updateTxKey(customerRefId, resp.getTxKey(), "SUBMITTED");
} catch (Exception e) {
    if (isDuplicateRefIdError(e)) {
        // Error 9001: already submitted — query by customerRefId
        OneTransactionsRequest query = new OneTransactionsRequest();
        query.setCustomerRefId(customerRefId);
        OneTransactionsResponse existing = ServiceExecutor.execute(transactionApi.oneTransactions(query));
        withdrawalOrderDao.updateTxKey(customerRefId, existing.getTxKey(), "SUBMITTED");
    } else {
        // Genuine failure — log and retry later
        log.error("Failed to submit withdrawal", e);
    }
}
```

### Withdrawal status sync via Webhook:

```java
// In webhook handler for TRANSACTION_STATUS_CHANGED
String txKey         = event.get("txKey").asText();
String customerRefId = event.get("customerRefId").asText();
String newStatus     = event.get("transactionStatus").asText();

WithdrawalOrder order = withdrawalOrderDao.findByTxKeyOrCustomerRefId(txKey, customerRefId);
if (order == null) return;  // unknown tx — ignore

// Never roll back status
if (isTerminalStatus(order.getStatus())) return;

order.setStatus(newStatus);
withdrawalOrderDao.save(order);

if ("COMPLETED".equals(newStatus)) {
    notifyUserWithdrawalCompleted(order);
} else if ("FAILED".equals(newStatus) || "REJECTED".equals(newStatus) || "CANCELLED".equalsIgnoreCase(newStatus)) {
    refundUserBalance(order);
    notifyUserWithdrawalFailed(order);
}
```

---

## 6. Small Auto / Large Manual — Approval Tier Pattern

For exchange-style deployments, configure the Safeheron policy engine to route transactions by amount:

| Amount Range (24H cumulative) | Approval |
|-------------------------------|---------|
| Single tx ≤ 100,000 | API Co-Signer auto-approve |
| Single tx > 100,000 | Ops team 2-of-3 |
| 24H total 100K–500K | Ops team 2-of-3 |
| 24H total 500K–2M | Finance team 2-of-2 |
| 24H total > 2M | Executive 1-of-2 |

This requires:
1. API Co-Signer deployed (for auto tier)
2. Policy Engine configured in Web Console
3. Approval Callback Service implemented (for Co-Signer to callback your business logic)

See [POLICY_STRATEGY.md](POLICY_STRATEGY.md) for the complete configuration reference.

---

## 7. Transaction Acceleration (Speed-Up)

Blockchain gas fees fluctuate. A stuck transaction blocks all subsequent transactions from the same account.

**Monitor for stuck transactions:**
- If a transaction remains in `BROADCASTING` or `CONFIRMING` for too long (define a threshold per chain), trigger an internal alert
- Call the speed-up API:

```java
RecreateTransactionRequest req = new RecreateTransactionRequest();
req.setTxKey(stuckTxKey);
req.setTxFeeLevel("HIGH");   // use a higher fee to replace the stuck tx

TxKeyResult resp = ServiceExecutor.execute(transactionApi.recreateTransactions(req));
String newTxKey = resp.getTxKey();

// The original tx will eventually reach FAILED status
// Track the speed-up tx via newTxKey
// Use replacedTxKey / replacedCustomerRefId fields to link back to original
```

**Important:** A user withdrawal order may end up linked to multiple Safeheron `txKey` values (original + one or more speed-ups). Your DB schema must support this 1:many relationship. The final completed `txKey` is the canonical one.

Chains that support speed-up: all EVM chains, BTC, BCH, DASH, LTC, DOGE, FIL, Aptos, CFX.
Chains that do NOT support speed-up: NEAR, SUI, TRON, SOL, TON.

---

## 8. AML / Risk Control Integration

### At withdrawal time (before submitting to Safeheron):
- Run destination address through your internal risk engine
- Optionally use the Safeheron Tools API (AML address checker) — see [TOOLS_API.md](TOOLS_API.md)
- Reject withdrawals to high-risk or blacklisted addresses before they reach Safeheron

### Via Safeheron's built-in AML:
- `failOnAml: true` (default) — Safeheron blocks the transaction if destination is flagged by its AML provider
- `amlLock` field in webhook/transaction response indicates whether AML blocked a deposit

### In the Approval Callback Service (Co-Signer):
```java
// Check AML risk in callback
for (Aml aml : req.getAmlList()) {
    if ("HIGH".equalsIgnoreCase(aml.getRiskLevel())) {
        return "REJECT";
    }
}
```
