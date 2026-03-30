# Security Best Practices

> **These rules are mandatory.** When this Skill is active, every piece of generated code and every architecture recommendation must comply without exception.

---

## 1. Credential & Key Management

### Rules for Generated Code

- **Private keys must never be stored in plaintext** -- not in config files, environment variables, or source code.
- Use a secrets management service appropriate for the deployment target:

| Deployment Target | Required Solution |
|---|---|
| Cloud (AWS) | AWS KMS |
| Cloud (GCP) | GCP KMS |
| Self-hosted | HashiCorp Vault |
| Local development only | Read from a file **outside the project directory**; the file must never be committed to git |

### Code Pattern -- Loading Credentials

```python
import os

# CORRECT: Load from environment variable (dev/test)
config = {
    'apiKey': os.environ['SAFEHERON_API_KEY'],
    'privateKey': os.environ['SAFEHERON_RSA_PRIVATE_KEY'],
    'safeheronPublicKey': os.environ['SAFEHERON_PLATFORM_PUBLIC_KEY'],
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}

# CORRECT: Load from AWS Secrets Manager (production)
# import boto3
# client = boto3.client('secretsmanager')
# secret = client.get_secret_value(SecretId='safeheron/rsa-private-key')
# private_key = secret['SecretString']

# WRONG -- never do this:
# config = {
#     'apiKey': 'sk-live-xxxxxxxxxxxx',
#     'privateKey': '-----BEGIN RSA PRIVATE KEY-----\nMIIE...',
# }
```

---

## 2. Transfer & Whitelist Security

### Rules for Generated Code

**2-1. `customerRefId` must be a business-unique ID for idempotency.**
Generate it and persist it to your database **before** calling the Safeheron API. On timeout, retry with the same ID.

```python
import uuid

customer_ref_id = str(uuid.uuid4())
# Save to DB first, then call API
```

**2-2. Whitelist addresses are required for formal transfers.**
`ONE_TIME_ADDRESS` must only be used for genuinely temporary, one-off payment scenarios.

**2-3. AML check is mandatory before every transfer.**

```python
from safeheron_api_sdk_python.api.tools_api import ToolsApi, AmlCheckerRequestRequest

tools_api = ToolsApi(config)
param = AmlCheckerRequestRequest()
param.network = 'Ethereum'
param.address = destination_address
resp = tools_api.aml_checker_request(param)
# Poll for result and block high-risk addresses
```

**2-4. Validate address format before whitelist add or transfer.**

```python
from safeheron_api_sdk_python.api.coin_api import CoinApi, CheckCoinAddressRequest

coin_api = CoinApi(config)
param = CheckCoinAddressRequest()
param.coinKey = "ETHEREUM_ETH"
param.address = destination_address
resp = coin_api.check_coin_address(param)
if not resp.get('addressValid'):
    raise ValueError(f"Invalid address format: {destination_address}")
```

**2-5. Amounts must use `str` in API requests and `Decimal` in application logic.**
Never use `float` for monetary values -- precision loss is guaranteed.

```python
from decimal import Decimal

# CORRECT
param.txAmount = str(Decimal("0.001"))

# WRONG -- never use float for amounts
# param.txAmount = 0.001
# param.txAmount = str(0.1 + 0.2)  # produces "0.30000000000000004"
```

---

## 3. Co-Signer Approval Callback

### Rules for Generated Code

**The Co-Signer callback service must never blindly approve all transactions.**
Every approval callback must implement the following business validations before returning `APPROVE`:

| Validation | What to Check |
|---|---|
| `customerRefId` | Must match a real pending business order in your system |
| Amount | Must exactly match the amount recorded in your business system |
| Destination address | Must exactly match the address recorded in your business system |

```python
from flask import Flask, request, jsonify
from decimal import Decimal

app = Flask(__name__)

@app.route('/cosigner/callback', methods=['POST'])
def handle_callback():
    raw_body = request.get_json(force=True)

    # 1. Decrypt and verify signature (always first)
    biz_content = converter.request_v3_convert(raw_body)
    tx = biz_content.get('customerContent', {})

    # 2. Look up the business order
    order = find_order_by_ref_id(tx.get('customerRefId'))
    if order is None:
        return make_reject_response(biz_content, "Unknown order")

    # 3. Validate amount
    if Decimal(tx.get('txAmount', '0')) != order['expected_amount']:
        return make_reject_response(biz_content, "Amount mismatch")

    # 4. Validate destination address
    if tx.get('destinationAddress', '').lower() != order['destination_address'].lower():
        return make_reject_response(biz_content, "Address mismatch")

    return make_approve_response(biz_content)
```

**Additional Co-Signer deployment rules:**
- The Co-Signer host must be in a **private isolated network**.
- The Approval Callback Service must only accept requests from the Co-Signer host IP.
- Production API Keys **must** have a Callback URL configured.

---

## 4. Webhook Security

### Rules for Generated Code

**4-1. Verify signature before any processing.**
The `WebhookConverter.converter()` method handles this automatically -- it raises an exception if the signature is invalid.

**4-2. Webhook endpoint must use HTTPS** in production.

**4-3. Idempotency is required.**
Safeheron retries delivery up to 7 times. The handler must safely process duplicate events.

**4-4. IP whitelist -- only accept from Safeheron egress IPs.**
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback -- terminal states are final.**

```python
TERMINAL_STATUSES = {'COMPLETED', 'SIGN_COMPLETED', 'FAILED', 'REJECTED', 'CANCELLED'}

def is_terminal_status(status):
    return status in TERMINAL_STATUSES
```

**4-6. Handle out-of-order delivery.**

```python
# In async processor:
def process_webhook_event(event):
    tx = find_transaction_by_tx_key(event['txKey'])
    if tx and is_terminal_status(tx['status']):
        # Ignore late event for terminal transaction
        return
    update_transaction_status(event['txKey'], event['transactionStatus'])
```

**4-7. Implement REST API polling as fallback.**

**4-8. Handle failed Webhook re-delivery.**
Call the transaction query API periodically to catch missed events.

---

## 5. Policy Configuration Security

- Configure tiered approval based on transaction amount.
- Always add a **catch-all blocking rule at the bottom** of the policy stack.
- Subscribe to `NO_MATCHING_TRANSACTION_POLICY` Webhook event as an operational alert.
- After policy configuration, validate by testing with small amounts.

---

## 6. API Key Security

- Scope API Key permissions to the **minimum required** for each use case.
- Register only the minimum necessary server IPs in the IP whitelist.
- Non-essential personnel must not have access to RSA private keys.
- Rotate API Keys periodically and immediately upon suspected compromise.

---

## Quick Reference

| Area | Key Rule |
|---|---|
| Private keys | KMS/Vault only; never plaintext |
| Transfer addresses | Whitelist required; ONE_TIME_ADDRESS only for truly one-off payments |
| AML check | Mandatory before every outbound transfer via ToolsApi |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.check_coin_address() |
| Amounts | str in API, Decimal in code; never float |
| Co-Signer callback | Validate customerRefId + amount + address against business system |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
