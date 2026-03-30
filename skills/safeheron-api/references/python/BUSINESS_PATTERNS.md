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

```python
from safeheron_api_sdk_python.api.account_api import AccountApi, CreateAccountRequest

account_api = AccountApi(config)

param = CreateAccountRequest()
param.accountName = f"user-{user_id}"
param.hiddenOnUI = True           # hide from App/Console UI
param.accountTag = "DEPOSIT"      # required for Auto-Sweep

resp = account_api.create_account(param)
account_key = resp['accountKey']  # store: bind this accountKey <-> userId in your DB
```

### Step 2 -- Add coin, get deposit address

```python
from safeheron_api_sdk_python.api.account_api import CreateAccountCoinV2Request

param = CreateAccountCoinV2Request()
param.accountKey = account_key
param.coinKeyList = ["ETHEREUM_ETH", "USDT(ERC20)_ETHEREUM_USDT"]

resp = account_api.create_account_coin_v2(param)
deposit_address = resp['coinAddressList'][0]['addressList'][0]['address']
# Show this to the user as their deposit address
```

---

## 3. Deposit Detection (User Credit)

Use **Webhook as primary + REST API polling as fallback**. Never rely on only one.

### 3-1. Webhook -- Primary Path

Subscribe to `TRANSACTION_STATUS_CHANGED`. Filter for deposit (inflow) transactions:

```python
# In your webhook handler:
event_type = biz_content.get('eventType')
if event_type != 'TRANSACTION_STATUS_CHANGED':
    return

tx_type = biz_content.get('transactionType')       # "NORMAL"
tx_direction = biz_content.get('transactionDirection')  # "INFLOW"
status = biz_content.get('transactionStatus')

if tx_type == 'NORMAL' and tx_direction == 'INFLOW':
    handle_deposit_status_change(biz_content)
```

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | On-chain, 0 confirmations | Optionally show "pending" to user |
| `CONFIRMING` | >=1 confirmation | Optionally show confirmation count |
| `COMPLETED` | >=N confirmations (Safeheron default) | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules:**
- Return HTTP 200 **immediately** before any processing
- **Never roll back status** -- if DB already has `COMPLETED`, ignore a late `CONFIRMING` event
- Implement **idempotency by `txKey`**

**Anti-dust attack:** Configure a minimum deposit amount and ignore events below that threshold.

### 3-2. REST API Polling -- Fallback Path

```python
import time
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, ListTransactionsV2Request

transaction_api = TransactionApi(config)

# Run periodically as a background job
param = ListTransactionsV2Request()
param.limit = 50
param.createTimeMin = last_polled_timestamp_ms
param.transactionStatus = "COMPLETED"

tx_list = transaction_api.list_transactions_v2(param)
for tx in tx_list:
    if tx.get('transactionDirection') == 'INFLOW' and tx.get('transactionStatus') == 'COMPLETED':
        credit_user_if_not_already(tx['txKey'], tx)
```

---

## 4. Asset Consolidation (Sweep) & Gas Top-Up

### Option A -- Auto-Sweep (Recommended, Zero Code)

Configure in Safeheron Console -> Auto-Sweep. Requirements:
- API Co-Signer must be deployed
- Deposit wallets must have `accountTag = "DEPOSIT"`
- Configure sweep rules in Console

### Option B -- Manual Sweep via API

```python
import uuid
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, CreateTransactionRequest

param = CreateTransactionRequest()
param.customerRefId = str(uuid.uuid4())
param.coinKey = "USDT(ERC20)_ETHEREUM_USDT"
param.txAmount = sweep_amount
param.sourceAccountKey = deposit_wallet_account_key
param.sourceAccountType = "VAULT_ACCOUNT"
param.destinationAccountType = "VAULT_ACCOUNT"
param.destinationAccountKey = hot_wallet_account_key
param.txFeeLevel = "MIDDLE"

resp = transaction_api.create_transactions(param)
```

---

## 5. User Withdrawal (Outflow) -- Recommended Flow

### Architecture: Async with DB-first pattern

```
Frontend -> [1. Save withdrawal order to DB, status=PENDING]
         -> [Return "request received" to user]

Background Job -> [2. Fetch PENDING orders from DB]
               -> [3. Call Safeheron createTransaction, get txKey]
               -> [4. Update order: txKey, status=SUBMITTED]

Webhook Handler -> [5. Receive TRANSACTION_STATUS_CHANGED]
                -> [6. Update order status by txKey/customerRefId]
                -> [7. Notify user of completion]
```

### Key Implementation

```python
import uuid
from decimal import Decimal
from safeheron_api_sdk_python.api.transaction_api import (
    TransactionApi, CreateTransactionRequest, OneTransactionsRequest,
)

transaction_api = TransactionApi(config)

# Step 1: save order first
customer_ref_id = str(uuid.uuid4())
save_withdrawal_order(user_id, amount, to_address, customer_ref_id, "PENDING")

# Step 2: submit to Safeheron
param = CreateTransactionRequest()
param.customerRefId = customer_ref_id
param.coinKey = coin_key
param.txAmount = str(amount)  # always string
param.sourceAccountKey = hot_wallet_account_key
param.sourceAccountType = "VAULT_ACCOUNT"
param.destinationAccountType = "ONE_TIME_ADDRESS"
param.destinationAddress = to_address
param.txFeeLevel = "MIDDLE"
param.failOnAml = True
param.failOnContract = True

try:
    resp = transaction_api.create_transactions(param)
    update_order_tx_key(customer_ref_id, resp['txKey'], "SUBMITTED")
except Exception as e:
    if is_duplicate_ref_id_error(e):
        # Error 9001: already submitted -- query by customerRefId
        query_param = OneTransactionsRequest()
        query_param.customerRefId = customer_ref_id
        existing = transaction_api.one_transactions(query_param)
        update_order_tx_key(customer_ref_id, existing['txKey'], "SUBMITTED")
    else:
        raise
```

### Withdrawal status sync via Webhook

```python
# In webhook handler for TRANSACTION_STATUS_CHANGED
tx_key = biz_content.get('txKey')
customer_ref_id = biz_content.get('customerRefId')
new_status = biz_content.get('transactionStatus')

order = find_order_by_tx_key_or_ref_id(tx_key, customer_ref_id)
if order is None:
    return  # unknown tx -- ignore

# Never roll back status
if is_terminal_status(order['status']):
    return

update_order_status(order['id'], new_status)

if new_status == 'COMPLETED':
    notify_user_withdrawal_completed(order)
elif new_status in ('FAILED', 'REJECTED', 'CANCELLED'):
    refund_user_balance(order)
    notify_user_withdrawal_failed(order)
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

```python
from safeheron_api_sdk_python.api.transaction_api import RecreateTransactionRequest

param = RecreateTransactionRequest()
param.txKey = stuck_tx_key
param.txFeeLevel = "HIGH"

resp = transaction_api.recreate_transactions(param)
new_tx_key = resp['txKey']
```

Chains that support speed-up: all EVM chains, BTC, BCH, DASH, LTC, DOGE, FIL, Aptos, CFX.
Chains that do NOT support speed-up: NEAR, SUI, TRON, SOL, TON.

---

## 8. AML / Risk Control Integration

### At withdrawal time:
- Run destination address through your internal risk engine
- Optionally use the Safeheron Tools API (AML address checker) -- see [TOOLS_API.md](TOOLS_API.md)

### In the Approval Callback Service (Co-Signer):

```python
# Check AML risk in callback
for aml in customer_content.get('amlList', []):
    if aml.get('riskLevel', '').upper() == 'HIGH':
        return "REJECT"
```
