# Tools API Reference (AML Address Checker)

## Overview

The Tools API provides utilities for address-level AML risk assessment *before* initiating a transaction. This is distinct from transaction-level KYT reports (see [COMPLIANCE_API.md](COMPLIANCE_API.md)).

Use Case: Check whether a recipient address is flagged as high-risk **before** sending funds.

---

## Imports

```python
from safeheron_api_sdk_python.api.tools_api import ToolsApi, AmlCheckerRequestRequest, AmlCheckerRetrievesRequest
```

## Create API Instance

```python
tools_api = ToolsApi(config)
```

---

## AML Address Risk Assessment (Two-Step Async Flow)

Assessment is asynchronous: submit the address, then poll for results.

### Step 1: Submit Address for AML Assessment

```python
param = AmlCheckerRequestRequest()
param.network = 'Ethereum'  # "Bitcoin", "Ethereum", or "Tron"
param.address = '0xAbCd1234...'
resp = tools_api.aml_checker_request(param)
request_id = resp['requestId']  # save this for polling
```

### Request Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `network` | Yes | str | Blockchain network: `Bitcoin`, `Ethereum`, `Tron` |
| `address` | Yes | str | Blockchain address to assess |

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | str | Async assessment request ID -- use to poll for results |

---

### Step 2: Retrieve AML Assessment Result

```python
import time

for i in range(30):
    retrieve_param = AmlCheckerRetrievesRequest()
    retrieve_param.requestId = request_id
    resp = tools_api.aml_checker_retrieves(retrieve_param)
    mist_track = resp.get('mistTrack', {})
    if mist_track.get('status', '').upper() == 'SUCCESS':
        break
    time.sleep(2)

print(f"Risk Level: {mist_track.get('riskLevel')}")
print(f"Status:     {mist_track.get('status')}")
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | str | The assessment request ID |
| `network` | str | Blockchain network |
| `address` | str | Assessed address |
| `mistTrack` | dict | MistTrack risk assessment result |
| `mistTrack.status` | str | MistTrack risk assessment status (EVALUATING / SUCCESS) |
| `mistTrack.riskLevel` | str | `Low`, `Moderate`, `High`, `Severe` |

---

## Full Example: Check Address Before Sending

```python
import uuid
import time
from safeheron_api_sdk_python.api.tools_api import ToolsApi, AmlCheckerRequestRequest, AmlCheckerRetrievesRequest
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, CreateTransactionRequest

tools_api = ToolsApi(config)
transaction_api = TransactionApi(config)

destination_address = "0xAbCd1234..."

# 1. Submit address for assessment
submit_param = AmlCheckerRequestRequest()
submit_param.network = 'Ethereum'
submit_param.address = destination_address
submit_resp = tools_api.aml_checker_request(submit_param)
request_id = submit_resp['requestId']

# 2. Poll until result is ready
result = None
for i in range(30):
    retrieve_param = AmlCheckerRetrievesRequest()
    retrieve_param.requestId = request_id
    result = tools_api.aml_checker_retrieves(retrieve_param)
    if result.get('mistTrack', {}).get('status', '').upper() == 'SUCCESS':
        break
    time.sleep(2)

# 3. Block high-risk addresses
risk_level = result.get('mistTrack', {}).get('riskLevel', '').upper()
if risk_level in ('HIGH', 'SEVERE'):
    raise RuntimeError(f"AML check failed: destination address is {risk_level}-risk")

# 4. Proceed with transaction
param = CreateTransactionRequest()
param.customerRefId = str(uuid.uuid4())
param.destinationAddress = destination_address
param.destinationAccountType = "ONE_TIME_ADDRESS"
param.coinKey = "ETHEREUM_ETH"
param.txAmount = "0.05"
param.sourceAccountKey = account_key
param.sourceAccountType = "VAULT_ACCOUNT"
param.txFeeLevel = "MIDDLE"

resp = transaction_api.create_transactions(param)
print(f"Transaction created: {resp['txKey']}")
```

---

## Supported Networks

| Network Value | Supported Chains |
|---------------|-----------------|
| `Bitcoin` | Bitcoin (BTC) |
| `Ethereum` | Ethereum, ERC-20 tokens, and EVM-compatible chains |
| `Tron` | TRON (TRX), TRC-20 tokens |

---

## Best Practices

- AML checker uses external providers -- results may take several seconds.
- For transaction-level KYT reports (after a transaction completes), use `compliance_api.kyt_report()` -- see [COMPLIANCE_API.md](COMPLIANCE_API.md).
- The `failOnAml=True` flag in `CreateTransactionRequest` (default) enables automatic server-side AML blocking -- the Tools API is for manual pre-screening before sending.
- API endpoint: `POST /v1/tools/aml-checker/request` and `POST /v1/tools/aml-checker/retrieves`.
