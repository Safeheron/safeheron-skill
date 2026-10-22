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

config = {
    'apiKey': '${SAFEHERON_API_KEY}',#todo Replace with the API Key you read from Safeheron Console
    'privateKey': '${RSA_PRIVATE_KEY}', #todo Replace with the RSA private key you read from Vault/KMS
    'safeheronPublicKey': '${SAFEHERON_PLATFORM_PUBLIC_KEY}',#todo Replace with the Safeheron platform public key from Safeheron Console
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
Use KYA screening via `ComplianceApi` to screen the destination address before creating the transaction. Block or alert on HIGH / SEVERE (and, by default, UNKNOWN) risk results.

```python
from safeheron_api_sdk_python.api.compliance_api import ComplianceApi, CreateKyaScreeningRequest, KyaScreeningOneRequest
import time

screen_param = CreateKyaScreeningRequest()
screen_param.address = destination_address
screen_param.chainType = 'EVM'  # valid values via kya_supported_networks()
screen_param.providers = ['Chainalysis']
created = compliance_api.create_kya_screening(screen_param)

poll_param = KyaScreeningOneRequest()
poll_param.screenId = created['screenId']
for i in range(30):
    result = compliance_api.kya_screening_one(poll_param)
    if result.get('status') == 'FINISHED':
        break
    time.sleep(2)
# Fail closed: an unfinished screening is NOT a pass
if result.get('status') != 'FINISHED':
    raise RuntimeError(f"KYA screening not finished for {destination_address}, route to manual review")
orders = result.get('orders') or []
if not orders:
    raise RuntimeError(f"KYA screening returned no provider result for {destination_address}")
# Whitelist: only LOW / MEDIUM pass (adjust per your policy); HIGH, SEVERE, UNKNOWN, a missing level or FAILED are blocked
ALLOWED_RISK_LEVELS = {'LOW', 'MEDIUM'}
for order in orders:
    if order.get('status') == 'SKIPPED':
        continue  # manually skipped in Web Console
    if order.get('status') != 'COMPLETED':
        raise RuntimeError(f"KYA screening {order.get('status')} ({order.get('provider')}), route to manual review")
    if order.get('riskLevel') not in ALLOWED_RISK_LEVELS:
        raise RuntimeError(f"AML check failed: {destination_address} is {order.get('riskLevel')} risk")
```

**2-4. Validate address format before whitelist add or transfer.**

```python
from safeheron_api_sdk_python.api.coin_api import CoinApi, CheckCoinAddressRequest

coin_api = CoinApi(config)
param = CheckCoinAddressRequest()
param.coinKey = "ETHEREUM_ETH"
param.address = destination_address
param.checkAddressValid = True   # required — without it addressValid is not checked
param.checkAml = True            # required — without it amlValid defaults to True
resp = coin_api.check_coin_address(param)
if not resp.get('addressValid'):
    raise ValueError(f"Invalid address format: {destination_address}")
if not resp.get('amlValid'):
    raise ValueError(f"Address is on the AML blacklist: {destination_address}")
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

**The Co-Signer callback service must never blindly approve anything.** Use the Flask callback service in [COSIGNER.md](COSIGNER.md) (`decide()` + `passes_business_rules()` + `evaluate_aml()`) as the REQUIRED pattern — do not write a reduced version. It enforces:

| Rule | What it checks |
|---|---|
| Default REJECT | Unknown callback types, unknown `transactionType`, parse errors → `REJECT` |
| `MPC_SIGN` / `WEB3_SIGN` | Verified against your pending sign request (`dataList`, `messageHash` / `message` / `transaction`) — never auto-approved |
| `NORMAL` / `UTXO_BATCH_TRANSFER` | `customerRefId` matches a real pending order; coin (`coinKey`) and **every** destination address + amount + memo match it (UTXO coins use `destinationAddressList`) |
| `UTXO_COLLECTION` / `AUTO_FUEL` | Destination account type is `VAULT_ACCOUNT` (+ fuel policy) |
| `AUTO_SWEEP` | Destination account and USD amount match your Auto Sweep policy |
| `GAS_SERVICE_DEPOSIT` | Deposit exists in your records and destination account type is `PROFILE_ACCOUNT` |
| AML / KYT | Only `LOW` / `MEDIUM` pass (`HIGH` / `SEVERE` / `UNKNOWN` / unexpected values rejected); `FAILED` or unknown status rejected; wait (neither APPROVE nor REJECT) while `IN_PROGRESS` / `PENDING`, then reject after ~5 min — see [AML / KYT Handling](COSIGNER.md#aml--kyt-handling-in-the-callback) |

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

**4-3. Durable ack + idempotency are required.**
Failed deliveries are retried 6 times (30s, 1m, 5m, 1h, 12h, 24h) and duplicates can arrive anyway. Ack `{'code': '200', 'message': 'SUCCESS'}` only after the verified event is **durably stored** (DB inbox / durable MQ) -- never after handing it to an in-memory queue or a background thread. Commit the idempotency mark and the business side effect in the **same DB transaction**; never commit a dedup row first and process afterwards. `resend_failed` re-pushes failed TRANSACTION_*, MPC_SIGN_*, WEB3_SIGN_*, ILLEGAL_IP_REQUEST, NO_MATCHING_TRANSACTION_POLICY, WHITELIST_*, GAS_BALANCE_WARNING and AML_KYT_ALERT events -- but NOT TOKEN_BLACKLIST_ADDRESS_HIT, so durable storage before ack is still mandatory. See [WEBHOOK.md](WEBHOOK.md).

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
def process_webhook_event(event):  # event = eventDetail of a TRANSACTION_* webhook
    tx = find_transaction_by_tx_key(event['txKey'])
    if tx and is_terminal_status(tx['status']):
        # Ignore late event for terminal transaction
        return
    update_transaction_status(event['txKey'], event['transactionStatus'])
```

**4-7. Implement REST API polling as fallback.**

**4-8. Handle failed Webhook re-delivery.**
Call the transaction query API periodically to catch missed events (set `createTimeMin` explicitly -- List Transactions V2 defaults to the last 24 hours).

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
| Private keys | Vault/KMS only; never plaintext |
| Transfer addresses | Whitelist required; ONE_TIME_ADDRESS only for truly one-off payments |
| AML check | Mandatory before every outbound transfer via ComplianceApi KYA screening |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.check_coin_address() |
| Amounts | str in API, Decimal in code; never float |
| Co-Signer callback | Use the Flask callback service (COSIGNER.md): rules per transactionType (order + coin + address + amount + memo for NORMAL), AML/KYT whitelist, default REJECT |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
