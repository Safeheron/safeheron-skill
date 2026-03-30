# Transaction API Reference

## Imports

```python
import uuid
from safeheron_api_sdk_python.api.transaction_api import (
    TransactionApi,
    CreateTransactionRequest,
    OneTransactionsRequest,
    ListTransactionsV2Request,
    ListTransactionsV1Request,
    TransactionsFeeRateRequest,
    CancelTransactionRequest,
    RecreateTransactionRequest,
)
```

## Create API Instance

```python
transaction_api = TransactionApi(config)
```

---

## Create a Transaction

```python
param = CreateTransactionRequest()
param.customerRefId = str(uuid.uuid4())   # your unique ID
param.coinKey = "ETHEREUM_ETH"
param.txAmount = "0.01"                    # string, never float
param.sourceAccountKey = account_key
param.sourceAccountType = "VAULT_ACCOUNT"
param.destinationAccountType = "ONE_TIME_ADDRESS"
param.destinationAddress = "0xRecipientAddress"
param.txFeeLevel = "MIDDLE"               # "LOW" | "MIDDLE" | "HIGH"
# Optional: param.note = "Payment for invoice #123"
# Optional: param.customerExt1 = "order-id-456"

resp = transaction_api.create_transactions(param)
tx_key = resp['txKey']  # save this -- Safeheron transaction identifier
```

### CreateTransactionRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `customerRefId` | Yes | str | Your unique ID for idempotency (max 100 chars) |
| `coinKey` | Yes | str | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `txAmount` | Yes | str | Amount as string -- never use float |
| `sourceAccountKey` | Yes | str | Sender wallet key |
| `sourceAccountType` | Yes | str | `VAULT_ACCOUNT` |
| `destinationAccountType` | Yes | str | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `destinationAddress` | Cond. | str | Required when `destinationAccountType=ONE_TIME_ADDRESS` |
| `destinationAccountKey` | Cond. | str | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `txFeeLevel` | No | str | `LOW` / `MIDDLE` / `HIGH` (priority over feeRateDto) |
| `feeRateDto` | No | dict | Custom fee rate (requires both `gasLimit` AND `feeRate`) |
| `maxTxFeeRate` | No | str | Max acceptable fee rate |
| `note` | No | str | Transaction note (max 180 chars) |
| `customerExt1` | No | str | Custom field 1 (max 255 chars) |
| `customerExt2` | No | str | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | bool | If True, fee is deducted from `txAmount` |
| `memo` | No | str | Memo/tag (for TON, XRP, Stellar -- max 100 chars) |
| `failOnAml` | No | bool | Default True -- block high-risk addresses via AML |
| `failOnContract` | No | bool | Block contract addresses as destination |
| `nonce` | No | int | Custom nonce for EVM chains |
| `isRbf` | No | bool | Enable Replace-by-Fee for Bitcoin |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` -- providing only `gasLimit` will cause the transaction to fail.

---

## Retrieve a Single Transaction

```python
param = OneTransactionsRequest()
param.txKey = tx_key
# OR: param.customerRefId = customer_ref_id

resp = transaction_api.one_transactions(param)
print(f"Status: {resp['transactionStatus']}")
print(f"Sub-status: {resp['transactionSubStatus']}")
print(f"TxHash: {resp.get('txHash', 'N/A')}")
```

**Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `txHash` | str | On-chain transaction hash |
| `coinKey` | str | Coin identifier |
| `txAmount` | str | Transaction amount |
| `txFee` | str | Actual fee paid |
| `transactionStatus` | str | See status table below |
| `transactionSubStatus` | str | Sub-status |
| `sourceAccountKey` | str | Sender wallet key |
| `sourceAddress` | str | Sender address |
| `destinationAddress` | str | Recipient address |
| `createTime` | int | Unix timestamp (ms) |
| `completedTime` | int | Unix timestamp (ms) of completion |
| `customerRefId` | str | Your reference ID |
| `amlLock` | str | `YES` / `NO` -- AML blocked |

---

## List Transactions

### V2 (Recommended -- cursor-based)

```python
param = ListTransactionsV2Request()
param.limit = 20
param.transactionStatus = "COMPLETED"  # optional filter
param.coinKey = "ETHEREUM_ETH"        # optional filter
# param.createTimeMin = start_ms      # optional timestamp filter
# param.createTimeMax = end_ms

tx_list = transaction_api.list_transactions_v2(param)
for tx in tx_list:
    print(f"{tx['txKey']} {tx['transactionStatus']}")
```

**ListTransactionsV2Request Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `direct` | str | Page direction: `NEXT` (default) |
| `limit` | int | Items per page, max 500 |
| `fromId` | str | txKey of last record from previous page; omit for first page |
| `sourceAccountKey` | str | Source account key |
| `coinKey` | str | Coin key, multiple separated by commas |
| `transactionStatus` | str | Transaction status |
| `createTimeMin` | int | Start time, UNIX timestamp (ms) |
| `createTimeMax` | int | End time, UNIX timestamp (ms) |
| `transactionDirection` | str | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER`; default: all |

### V1 (legacy -- page-based)

```python
param = ListTransactionsV1Request()
param.pageSize = 10
param.pageNumber = 1

result = transaction_api.list_transactions_v1(param)
tx_list = result['content']
```

---

## Estimate Transaction Fee

```python
param = TransactionsFeeRateRequest()
param.coinKey = "ETHEREUM_ETH"
param.value = "0.01"
param.sourceAccountKey = account_key
param.destinationAddress = "0xRecipient"

resp = transaction_api.transaction_fee_rate(param)
print(f"Low:    {resp['lowFeeRate']}")
print(f"Middle: {resp['middleFeeRate']}")
print(f"High:   {resp['highFeeRate']}")
```

> **BTC fee** = feeRate x bytesSize. **ETH fee** = feeRate x gasLimit.
> **TRON fee**: `sourceAddress` must be provided to check on-chain staking/energy.

---

## Cancel a Transaction

```python
param = CancelTransactionRequest()
param.txKey = tx_key

transaction_api.cancel_transactions(param)
```

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

Replaces a stuck transaction with a higher fee version:

```python
param = RecreateTransactionRequest()
param.txKey = original_tx_key
param.txFeeLevel = "HIGH"  # use higher fee

resp = transaction_api.recreate_transactions(param)
new_tx_key = resp['txKey']
```

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
| `REJECTED` | Rejected by approver |

### Status Flow

```
SUBMITTED
    -> SIGNING
    -> BROADCASTING
    -> CONFIRMING
    -> COMPLETED
    +-- FAILED

    At SUBMITTED or SIGNING:
    -> REJECTED (denied by approver)
    -> CANCELLED (cancelled by team member)
```

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
- **TRON**: Fee has no LOW/MID/HIGH tiers -- pre-executed on-chain; all tiers return the same fee. `sourceAddress` required for fee estimation.
- **Contract addresses**: API transfers to contract addresses require `failOnContract = False` (default `True` blocks them).
- **destinationAccountKey rules**:
  - `destinationAccountType = VAULT_ACCOUNT` -> pass `accountKey`
  - `destinationAccountType = WHITELISTING_ACCOUNT` -> pass `whitelistKey`
  - `destinationAccountType = ONE_TIME_ADDRESS` -> do NOT pass `destinationAccountKey`; use `destinationAddress`

---

## Best Practices

1. Always set `customerRefId` from your own DB record ID before calling create.
2. Store the returned `txKey` -- it's Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call one_transactions periodically as fallback.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) -- never use float.
6. On timeout, **retry with the same `customerRefId`** -- Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
