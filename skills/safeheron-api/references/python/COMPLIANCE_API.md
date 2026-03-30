# Compliance (KYT / AML) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / AML (Anti-Money Laundering) providers to assess transaction risk.

Two main integration points:
1. **ComplianceApi** -- retrieve KYT risk reports for completed transactions.
2. **ToolsApi (AML Checker)** -- proactively assess a blockchain address before transacting. See [TOOLS_API.md](TOOLS_API.md).

---

## Imports

```python
from safeheron_api_sdk_python.api.compliance_api import ComplianceApi, KytReportRequest
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
# Optional: param.customerRefId = 'your-ref-id'

resp = compliance_api.kyt_report(param)

for aml in resp.get('amlList', []):
    print(f"Provider: {aml['provider']}")
    print(f"Risk Level: {aml['riskLevel']}")
    print(f"Status: {aml['status']}")
```

### Request Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKey` | Yes* | str | Safeheron transaction key |
| `customerRefId` | Yes* | str | Your reference ID (max 100 chars) |

> *Either `txKey` or `customerRefId` must be provided.

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `customerRefId` | str | Your reference ID |
| `amlList` | list | List of AML assessment results |

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | str | AML provider name (e.g. "Elliptic", "Chainalysis") |
| `timestamp` | str | Assessment timestamp (milliseconds) |
| `status` | str | Assessment status |
| `riskLevel` | str | Risk level: `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `lastUpdateTime` | str | Last update timestamp |
| `payload` | dict | Raw provider-specific data |

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

## Best Practices

- For address-level risk checks *before* initiating a transaction, use the Tools AML Checker -- see [TOOLS_API.md](TOOLS_API.md).
- Safeheron's server timezone is UTC+0.
- The `failOnAml` flag defaults to `True`. Setting it to `False` bypasses AML blocking entirely.
