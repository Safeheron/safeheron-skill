# Business Integration Patterns

Practical patterns for building a crypto custody / exchange system on top of Safeheron. Covers deposit detection, withdrawal flow, asset consolidation, and transaction acceleration.

---

## 1. Wallet Architecture Concepts

### Wallet Account Structure

Each Safeheron wallet account is a **full-chain wallet** -- it holds one address per coin (except UTXO chains like BTC which support multiple addresses per account).

```
Wallet Account 1          Wallet Account 2          Wallet Account ...
    |-- BTC Address           |-- ETH Address           ...
    |-- ETH Address           |-- USDT-ERC20 Address
    |-- USDT-ERC20 Address    |-- ...
```

**Recommendation: bind each end-user to exactly one Safeheron wallet account (1:1).**

### Transaction Directions

From the team's perspective, every transaction is one of three directions:

| Direction | Definition | Example |
|-----------|-----------|---------|
| `INFLOW` | External -> team wallet | User deposit |
| `OUTFLOW` | Team wallet -> external | User withdrawal |
| `INTERNAL_TRANSFER` | Team wallet -> team wallet | Consolidation, gas top-up |

---

## 2. Allocating Deposit Addresses to Users

### Step 1 -- Create wallet account, bind to user

```typescript
import { AccountApi } from '@safeheron/api-sdk';

const accountApi = new AccountApi(config);

const resp = await accountApi.createAccount({
  accountName: `user-${userId}`,
  hiddenOnUI: true,          // hide from App/Console UI
  accountTag: 'DEPOSIT',     // required for Auto-Sweep
});

const accountKey = resp.accountKey;
// store: bind this accountKey <-> userId in your DB
```

### Step 2 -- Add coin, get deposit address

```typescript
const res = await accountApi.createAccountCoinV2({
  accountKey,
  coinKeyList: ['ETHEREUM_ETH', 'USDT(ERC20)_ETHEREUM_USDT'],
});

const depositAddress = res.coinAddressList[0].addressList[0].address;
// show this to the user as their deposit address
```

---

## 3. Deposit Detection (User Credit)

Use **Webhook as primary + REST API polling as fallback**. Never rely on only one.

### 3-1. Webhook -- Primary Path

Subscribe to `TRANSACTION_STATUS_CHANGED`. Filter for deposit (inflow) transactions:

```typescript
// In your webhook handler:
function handleWebhookEvent(event: any) {
  if (event.eventType !== 'TRANSACTION_STATUS_CHANGED') return;

  const txType = event.transactionType;       // "NORMAL"
  const txDirection = event.transactionDirection;  // "INFLOW"
  const status = event.transactionStatus;

  if (txType === 'NORMAL' && txDirection === 'INFLOW') {
    handleDepositStatusChange(event);
  }
}
```

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | On-chain, 0 confirmations | Optionally show "pending" to user |
| `CONFIRMING` | >=1 confirmation | Optionally show confirmation count |
| `COMPLETED` | >=N confirmations (Safeheron default) | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules for webhook handlers:**
- Return HTTP 200 **immediately** before any processing -- put work on an async queue
- **Never roll back status** -- if DB already has `COMPLETED`, ignore a late `CONFIRMING` event
- `BROADCASTING` and `CONFIRMING` may be skipped -- handle a direct jump to `COMPLETED`
- Implement **idempotency by `txKey`** -- Safeheron may deliver the same event more than once

**Anti-dust attack:** Configure a minimum deposit amount in your system and ignore webhook events below that threshold.

### 3-2. REST API Polling -- Fallback Path

```typescript
import { TransactionApi } from '@safeheron/api-sdk';

const transactionApi = new TransactionApi(config);

// Run every few minutes as a background job
async function pollTransactions(lastPolledTimestampMs: number) {
  const txList = await transactionApi.listTransactionsV2({
    limit: 50,
    createTimeMin: lastPolledTimestampMs,
    transactionStatus: 'COMPLETED',
  });

  for (const tx of txList) {
    if (tx.transactionDirection === 'INFLOW' && tx.transactionStatus === 'COMPLETED') {
      await creditUserIfNotAlready(tx.txKey, tx);
    }
  }
}
```

---

## 4. Asset Consolidation (Sweep) & Gas Top-Up

### Option A -- Auto-Sweep (Recommended, Zero Code)

Configure in Safeheron Console -> Auto-Sweep. Requirements:
- API Co-Signer must be deployed
- Deposit wallets must have `accountTag = "DEPOSIT"` set at creation time
- Configure sweep rules (coin, threshold, destination wallet) in Console

Auto-Sweep handles gas top-up and sweep automatically.

### Option B -- Manual Sweep via API

```typescript
import crypto from 'crypto';

// Internal transfer to hot wallet
await transactionApi.createTransactions({
  customerRefId: crypto.randomUUID(),
  coinKey: 'USDT(ERC20)_ETHEREUM_USDT',
  txAmount: sweepAmount,
  sourceAccountKey: depositWalletAccountKey,
  sourceAccountType: 'VAULT_ACCOUNT',
  destinationAccountType: 'VAULT_ACCOUNT',
  destinationAccountKey: hotWalletAccountKey,
  txFeeLevel: 'MIDDLE',
});
```

---

## 5. User Withdrawal (Outflow) -- Recommended Flow

### Architecture: Async with DB-first pattern

```
Frontend -> [1. Save withdrawal order to DB, status=PENDING]
         -> [Return "request received" to user]

Background Job -> [2. Fetch PENDING orders from DB]
               -> [3. Call Safeheron createTransactions, get txKey]
               -> [4. Update order: txKey, status=SUBMITTED]

Webhook Handler -> [5. Receive TRANSACTION_STATUS_CHANGED]
                -> [6. Update order status by txKey/customerRefId]
                -> [7. Notify user of completion]
```

### Key Implementation

```typescript
import crypto from 'crypto';

// Step 1: save order first
const customerRefId = crypto.randomUUID();
await withdrawalOrderDao.save({
  userId, amount, toAddress, customerRefId, status: 'PENDING',
});

// Step 2: submit to Safeheron
try {
  const resp = await transactionApi.createTransactions({
    customerRefId,
    coinKey,
    txAmount: amount,
    sourceAccountKey: hotWalletAccountKey,
    sourceAccountType: 'VAULT_ACCOUNT',
    destinationAccountType: 'ONE_TIME_ADDRESS',
    destinationAddress: toAddress,
    txFeeLevel: 'MIDDLE',
    failOnAml: true,
    failOnContract: true,
  });

  await withdrawalOrderDao.updateTxKey(customerRefId, resp.txKey, 'SUBMITTED');
} catch (e) {
  if (e instanceof SafeheronError && e.code === 9001) {
    // Already submitted -- query by customerRefId
    const existing = await transactionApi.oneTransactions({ customerRefId });
    await withdrawalOrderDao.updateTxKey(customerRefId, existing.txKey, 'SUBMITTED');
  } else {
    console.error('Failed to submit withdrawal:', e);
  }
}
```

### Withdrawal status sync via Webhook:

```typescript
function handleWithdrawalWebhook(event: any) {
  const { txKey, customerRefId, transactionStatus: newStatus } = event;

  const order = withdrawalOrderDao.findByTxKeyOrCustomerRefId(txKey, customerRefId);
  if (!order) return;  // unknown tx -- ignore

  // Never roll back status
  if (isTerminalStatus(order.status)) return;

  order.status = newStatus;
  withdrawalOrderDao.save(order);

  if (newStatus === 'COMPLETED') {
    notifyUserWithdrawalCompleted(order);
  } else if (['FAILED', 'REJECTED', 'CANCELLED'].includes(newStatus)) {
    refundUserBalance(order);
    notifyUserWithdrawalFailed(order);
  }
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

This requires:
1. API Co-Signer deployed (for auto tier)
2. Policy Engine configured in Web Console
3. Approval Callback Service implemented

See [POLICY_STRATEGY.md](POLICY_STRATEGY.md) for the complete configuration reference.

---

## 7. Transaction Acceleration (Speed-Up)

```typescript
const resp = await transactionApi.recreateTransactions({
  txKey: stuckTxKey,
  txFeeLevel: 'HIGH',
});

const newTxKey = resp.txKey;
// The original tx will eventually reach FAILED status
// Track the speed-up tx via newTxKey
```

**Important:** A user withdrawal order may end up linked to multiple Safeheron `txKey` values (original + one or more speed-ups). Your DB schema must support this 1:many relationship.

Chains that support speed-up: all EVM chains, BTC, BCH, DASH, LTC, DOGE, FIL, Aptos, CFX.
Chains that do NOT support speed-up: NEAR, SUI, TRON, SOL, TON.

---

## 8. AML / Risk Control Integration

### At withdrawal time (before submitting to Safeheron):
- Run destination address through your internal risk engine
- Optionally use the Safeheron Tools API (AML address checker) -- see [TOOLS_API.md](TOOLS_API.md)
- Reject withdrawals to high-risk or blacklisted addresses before they reach Safeheron

### Via Safeheron's built-in AML:
- `failOnAml: true` (default) -- Safeheron blocks the transaction if destination is flagged
- `amlLock` field in webhook/transaction response indicates whether AML blocked a deposit

### In the Approval Callback Service (Co-Signer):

```typescript
function evaluateTransaction(payload: any): string {
  for (const aml of payload.amlList || []) {
    if (aml.riskLevel === 'HIGH') {
      return 'REJECT';
    }
  }
  return 'APPROVE';
}
```
