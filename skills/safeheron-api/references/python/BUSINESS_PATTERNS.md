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
| `INTERNAL_TRANSFER` | Team wallet -> team wallet (source and destination are both `VAULT_ACCOUNT`) | Consolidation, gas top-up (`AUTO_FUEL`), and a user withdrawing to another user's deposit address on your platform. **Only a `NORMAL` internal transfer that matches one of your withdrawal orders is ever credited** — see `classify_transaction` below |

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
from safeheron_api_sdk_python.api.account_api import CreateAccountCoinRequestV2

param = CreateAccountCoinRequestV2()
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

Subscribe to `TRANSACTION_STATUS_CHANGED`. Decide what each transaction does with ONE classification function shared by the webhook handler and the polling job (3-2):

```python
# Module level: one classification rule shared by the webhook handler AND the polling job below.
IGNORE = 'IGNORE'
CREDIT_DEPOSIT = 'CREDIT_DEPOSIT'
SETTLE_WITHDRAWAL_AND_CREDIT_RECEIVER = 'SETTLE_WITHDRAWAL_AND_CREDIT_RECEIVER'


def classify_transaction(tx):
    # Anything but NORMAL is a system / ops movement, never a user deposit: AUTO_FUEL (Auto Sweep gas top-up of a
    # deposit address, vault -> vault = INTERNAL_TRANSFER), AUTO_SWEEP, UTXO_COLLECTION, GAS_SERVICE_DEPOSIT,
    # GAS_SERVICE_FUEL, BOUNCED_TRANSFER, ABNORMAL, UTXO_BATCH_TRANSFER, ...
    if tx.get('transactionType') != 'NORMAL':
        return IGNORE
    direction = tx.get('transactionDirection')
    if direction == 'INFLOW':
        return CREDIT_DEPOSIT
    if direction == 'INTERNAL_TRANSFER':
        # Only a transfer created by one of YOUR withdrawal orders (user A -> user B's deposit address) is credited;
        # any other internal transfer (hot -> cold, manual consolidation, ...) is not a deposit.
        order = find_withdrawal_order(tx.get('txKey'), tx.get('customerRefId'))  # placeholder: your order store
        return SETTLE_WITHDRAWAL_AND_CREDIT_RECEIVER if order is not None else IGNORE
    return IGNORE  # OUTFLOW: handled by the withdrawal flow (section 5), never credited


# In your webhook handler:
event_type = biz_content.get('eventType')
if event_type != 'TRANSACTION_STATUS_CHANGED':
    return

event_detail = biz_content.get('eventDetail') or {}  # business fields live under eventDetail
action = classify_transaction(event_detail)
if action == CREDIT_DEPOSIT:
    handle_deposit_status_change(event_detail)        # credit only on COMPLETED, idempotent by txKey
elif action == SETTLE_WITHDRAWAL_AND_CREDIT_RECEIVER:
    handle_internal_withdrawal(event_detail)          # settle the withdrawal order + credit receiver on COMPLETED
# IGNORE: system / ops movements (AUTO_FUEL, AUTO_SWEEP, UTXO_COLLECTION, ...) -- never credited


def handle_internal_withdrawal(event_detail):
    """Settle the matching withdrawal order (same rules as section 5) and, on COMPLETED, credit each output whose
    address is a user deposit address on your platform -- in ONE DB transaction, idempotent by txKey."""
    ...  # placeholder: settle_withdrawal(event_detail) + credit outputs via find_user_by_deposit_address(...)


def deposit_outputs(event_detail):
    """(address, amount) pairs credited by this transaction. UTXO coins (BTC, LTC, DOGE, ...) put them in
    destinationAddressList; destinationAddress is not populated for UTXO coins."""
    outputs = event_detail.get('destinationAddressList') or []
    if outputs:
        return [(o.get('address'), o.get('amount')) for o in outputs]
    return [(event_detail.get('destinationAddress'), event_detail.get('txAmount'))]
```

**Status progression for inflow transactions:**

| Status | Meaning | Action |
|--------|---------|--------|
| `BROADCASTING` | On-chain, 0 confirmations (incoming: **Bitcoin only** — other chains skip this state) | Optionally show "pending" to user |
| `CONFIRMING` | >=1 confirmation | Optionally show confirmation count |
| `COMPLETED` | >=N confirmations (Safeheron default) | **Credit user account** |
| `FAILED` | Failed on-chain | No credit |

**Critical rules:**
- Verify the signature and **durably store** the event (DB inbox / durable MQ), then ack `{"code":"200","message":"SUCCESS"}` -- do the business processing asynchronously from that store (see [WEBHOOK.md](WEBHOOK.md))
- **Never roll back status** -- if DB already has `COMPLETED`, ignore a late `CONFIRMING` event
- Implement **idempotency by `txKey`** and commit it in the **same DB transaction** as the credit -- never commit a dedup row first and credit afterwards

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
    if tx.get('transactionStatus') != 'COMPLETED':
        continue
    action = classify_transaction(tx)                  # same rule as the webhook path
    if action == CREDIT_DEPOSIT:
        credit_user_if_not_already(tx['txKey'], tx)    # idempotent by txKey (webhook may have credited it already)
    elif action == SETTLE_WITHDRAWAL_AND_CREDIT_RECEIVER:
        handle_internal_withdrawal(tx)                 # also idempotent by txKey
    # IGNORE: AUTO_FUEL / AUTO_SWEEP / UTXO_COLLECTION / ops transfers -- never credited
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
event_detail = biz_content.get('eventDetail') or {}  # business fields live under eventDetail
tx_key = event_detail.get('txKey')
customer_ref_id = event_detail.get('customerRefId')
new_status = event_detail.get('transactionStatus')

order = find_order_by_tx_key_or_ref_id(tx_key, customer_ref_id)
if order is None:
    return  # unknown tx -- ignore

# Never roll back status
if is_terminal_status(order['status']):
    return

# Speed-up: the original tx ends FAILED while its replacement may still complete — keep the order open and let the replacement decide
# (check BEFORE moving the order to a terminal state, otherwise the replacement's COMPLETED event is discarded)
if new_status in ('FAILED', 'REJECTED', 'CANCELLED') and has_active_replacement(order, tx_key):
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
param.txHash = stuck_tx_hash   # on-chain hash of the stuck transaction
param.coinKey = coin_key
param.txFeeLevel = "HIGH"

resp = transaction_api.recreate_transactions(param)
new_tx_key = resp['txKey']
```

Chains that support speed-up: EVM chains (ETH, ETC, BSC, MATIC, etc.), Bitcoin and Dogecoin (RBF and CPFP) and other UTXO chains supporting CPFP, Filecoin, Aptos, XRP Ledger.
Chains that do NOT support speed-up (not listed in the published docs as supported): NEAR, SUI, TRON, SOL, TON.

---

## 8. AML / Risk Control Integration

### At withdrawal time:
- Run destination address through your internal risk engine
- Screen the destination address with **KYA** (`ComplianceApi.create_kya_screening` -> poll `kya_screening_one`) -- see [COMPLIANCE_API.md](COMPLIANCE_API.md). Do **not** use the deprecated AML Checker (`/v1/tools/aml-checker/*`, `ToolsApi.aml_checker_request` / `aml_checker_retrieves`), and `checkAml` in `check_coin_address` is not a substitute for KYA
- Reject withdrawals to `HIGH` / `SEVERE` (and, by default, `UNKNOWN`) risk addresses before they reach Safeheron

### In the Approval Callback Service (Co-Signer):

Use `evaluate_aml(approval_id, tx)` in the Flask callback service ([COSIGNER.md](COSIGNER.md#aml--kyt-handling-in-the-callback)):
- Only `riskLevel` `LOW` / `MEDIUM` pass by default -- `HIGH`, `SEVERE`, `UNKNOWN` and unexpected values are rejected
- While `amlScreeningTriggeredState = IN_PROGRESS` or any `status = PENDING`, return neither APPROVE nor REJECT (Co-Signer retries every 5s); fall back to reject / manual review ~5 min after the first callback for that approval
- `status = FAILED`, a missing / unknown status, or `TRIGGERED` with an empty `amlList` -> reject and alert
