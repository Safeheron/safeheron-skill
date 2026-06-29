# Compliance (KYT / KYA) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / KYA (Know Your Address) AML providers to assess transaction and address risk.

Two main integration points:
1. **KYT** — retrieve AML risk reports for completed transactions (`kyt_report`).
2. **KYA** — proactively screen a blockchain address before transacting (`create_kya_screening` / `kya_screening_one` / `kya_screening_order_one`).

---

## Imports

```python
from safeheron_api_sdk_python.api.compliance_api import (
    ComplianceApi,
    KytReportRequest,
    CreateKyaScreeningRequest,
    KyaScreeningOneRequest,
    KyaScreeningOrderOneRequest,
)
```

## Create API Instance

```python
compliance_api = ComplianceApi(config)
```

---

## Retrieve KYT Risk Report for a Transaction

After a transaction completes, retrieve its AML/KYT risk assessment:

```python
param = KytReportRequest()
param.txKey = 'your-tx-key'
# OR: param.customerRefId = 'your-ref-id'

resp = compliance_api.kyt_report(param)

print('Screening state:', resp.get('amlScreeningTriggeredState'))
for aml in resp.get('amlList', []):
    print(f"Provider: {aml['provider']}")
    print(f"Risk Level: {aml['riskLevel']}")
    print(f"Status: {aml['status']}")
```

### KytReportRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKey` | Yes* | str | Safeheron transaction key |
| `customerRefId` | Yes* | str | Your reference ID (max 100 chars) |

> *Either `txKey` or `customerRefId` must be provided. If both are provided, `txKey` takes precedence.

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `customerRefId` | str | Your reference ID |
| `amlScreeningTriggeredState` | str | `IN_PROGRESS` / `TRIGGERED` / `UNTRIGGERED` — see below |
| `amlList` | list | List of AML assessment results (empty when `UNTRIGGERED`) |

**`amlScreeningTriggeredState` values:**

| Value | Meaning |
|-------|---------|
| `IN_PROGRESS` | Evaluating — screening not yet determined; `amlList` unavailable, poll again |
| `TRIGGERED` | Screening initiated; check `amlList` for risk results |
| `UNTRIGGERED` | Screening not triggered; `amlList` is empty |

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | str | AML provider name: `Elliptic`, `Chainalysis`, `MistTrack` |
| `timestamp` | str | Screening creation time (UNIX ms timestamp) |
| `status` | str | `PENDING` / `COMPLETED` / `SKIPPED` / `FAILED` |
| `riskLevel` | str | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `lastUpdateTime` | str | Last update time (UNIX ms timestamp) |
| `payload` | dict | Raw provider-specific report (structure varies by provider) |

---

## Create KYA Screening Request

Initiate an address screening against one or more AML providers in parallel:

```python
param = CreateKyaScreeningRequest()
param.address = '0xRecipientAddress'
param.chainType = 'ETH'
# param.network = 'ethereum'  # required when providers includes MistTrack
param.providers = ['Elliptic', 'Chainalysis']

resp = compliance_api.create_kya_screening(param)

print('Screen ID:', resp.get('screenId'))
for order in resp.get('orders', []):
    print(f"Order ID: {order['screenOrderId']}, Provider: {order['provider']}")
```

### CreateKyaScreeningRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `address` | Yes | str | On-chain address to screen (not limited to Safeheron addresses) |
| `chainType` | Yes | str | Chain type — see `kya_supported_networks()` for valid values |
| `network` | Conditional | str | Network identifier — required when `providers` includes `MistTrack` |
| `providers` | Yes | list | At least one of: `MistTrack`, `Elliptic`, `Chainalysis`; no duplicates |

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | str | Screening request ID (use for polling) |
| `address` | str | Screened address |
| `chainType` | str | Chain type |
| `network` | str | Network identifier |
| `orders` | list | One order per provider; each has `screenOrderId` and `provider` |
| `createTime` | int | Creation time (UNIX ms timestamp) |

---

## Retrieve KYA Screening Summary

Poll until `status` is `FINISHED`:

```python
param = KyaScreeningOneRequest()
param.screenId = 'your-screen-id'

resp = compliance_api.kya_screening_one(param)

print('Overall status:', resp.get('status'))  # PROCESSING / FINISHED
for order in resp.get('orders', []):
    print(f"  {order['provider']}: {order['status']} (risk={order['riskLevel']})")
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | str | Screening request ID |
| `address` | str | Screened address |
| `chainType` | str | Chain type |
| `network` | str | Network identifier |
| `status` | str | `PROCESSING` (some orders pending) / `FINISHED` (all completed) |
| `createTime` | int | Creation time (UNIX ms timestamp) |
| `orders` | list | Per-provider order summaries |

**Order summary fields:**

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | str | Screening order ID |
| `provider` | str | Provider identifier |
| `status` | str | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | str | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | int | Completion time (UNIX ms timestamp); only set in terminal state |

---

## Retrieve KYA Screening Order Details

Get the full provider report for a single order (includes raw `payload`):

```python
param = KyaScreeningOrderOneRequest()
param.screenOrderId = 'your-order-id'

resp = compliance_api.kya_screening_order_one(param)

print('Risk Level:', resp.get('riskLevel'))
print('Address Type:', resp.get('addressType'))
print('Payload:', resp.get('payload'))
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | str | Screening order ID |
| `screenId` | str | Parent screening request ID |
| `provider` | str | Provider identifier |
| `address` | str | Screened address |
| `addressType` | str | `VAULT_ACCOUNT` / `WHITELISTING_ACCOUNT` / `ONE_TIME_ADDRESS` |
| `chainType` | str | Chain type |
| `network` | str | Network identifier |
| `status` | str | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | str | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | int | Completion time (UNIX ms timestamp) |
| `createTime` | int | Screening initiation time (UNIX ms timestamp) |
| `payload` | dict | Provider's raw report (available after completion; structure varies by provider) |

---

## Retrieve Supported Networks & Providers

Query which chain/network combinations are supported by each AML provider:

```python
networks = compliance_api.kya_supported_networks()

for n in networks:
    print(f"Network: {n['network']}, ChainType: {n['chainType']}, Providers: {n['providers']}")
```

### Response Item Fields

| Field | Type | Description |
|-------|------|-------------|
| `network` | str | Network identifier |
| `chainType` | str | Chain type |
| `providers` | list | Providers supporting this network |

---

## AML Lock Behavior

When `failOnAml=True` (default) in `CreateTransactionRequest`, Safeheron will:
1. Automatically run AML checks on the destination address.
2. Block the transaction if the address scores high-risk (`amlLock = "YES"`).

To disable AML blocking (not recommended for production):

```python
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, CreateTransactionRequest

param = CreateTransactionRequest()
param.failOnAml = False  # disable AML lock -- use with caution
```

The transaction response `amlLock` field indicates `"YES"` or `"NO"`.

---

## AML Risk Assessment in Transaction Response

The transaction response (from list/query transaction APIs) includes inline AML data:

```python
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, OneTransactionsRequest

param = OneTransactionsRequest()
param.txKey = tx_key
tx = transaction_api.one_transactions(param)

for aml in tx.get('amlList', []):
    print(f"{aml['provider']} risk: {aml['riskLevel']}")
print(f"AML locked: {tx.get('amlLock', 'NO')}")
```

---

## Full Example: Screen Address Before Sending

Use KYA screening to check a destination address before initiating any outbound transfer:

```python
import time

# Step 1: Create KYA screening
screen_param = CreateKyaScreeningRequest()
screen_param.address = destination_address
screen_param.chainType = 'ETH'
screen_param.providers = ['Chainalysis']
created = compliance_api.create_kya_screening(screen_param)

# Step 2: Poll until FINISHED
poll_param = KyaScreeningOneRequest()
poll_param.screenId = created['screenId']
result = None
for i in range(30):
    result = compliance_api.kya_screening_one(poll_param)
    if result.get('status') == 'FINISHED':
        break
    time.sleep(2)

# Step 3: Handle each riskLevel according to your own business logic
for order in result.get('orders', []):
    risk_level = order.get('riskLevel')
    # Decide what to do based on risk_level (LOW / MEDIUM / HIGH / SEVERE / UNKNOWN).
    # For example, block HIGH/SEVERE, require manual review for MEDIUM, allow LOW, etc.
    # Implement the subsequent business logic (block, alert, proceed with transaction, ...) yourself.
    ...
```

---

## Best Practices

- Use KYA screening (`create_kya_screening`) to check an address **before** initiating a transaction. Poll `kya_screening_one` until `status=FINISHED`.
- Call `kya_supported_networks()` to determine valid `chainType` / `network` / `provider` combinations before submitting a screening.
- `network` is required when `providers` includes `MistTrack`.
- The `failOnAml` flag defaults to `True`. Setting it to `False` bypasses AML blocking entirely.
- Safeheron's server timezone is UTC+0.
