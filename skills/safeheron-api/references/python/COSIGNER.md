# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a service you deploy that **automatically approves or rejects transactions** based on your business logic. It replaces manual approval workflows for programmatic use cases and is required for unattended operation in production environments.

When a transaction requires approval, Safeheron calls your **Approval Callback Service** (an HTTP endpoint you implement) with an encrypted callback payload. Your service evaluates the transaction and responds `APPROVE` or `REJECT`.

---

## How It Works

```
1. Client creates transaction / MPC sign / Web3 sign
2. Safeheron: transaction enters SUBMITTED status
3. API Co-Signer polls Safeheron (every 5s in v2.x, every 1s in v1.x)
4. API Co-Signer calls your Approval Callback Service (HTTP POST)
5. Your service returns APPROVE or REJECT
6. Transaction proceeds to SIGNING or is REJECTED
```

**Approval Callback timeout:** Response timestamp must be within **5 seconds** of the API Co-Signer server's current time.

---

## Approval Callback -- Python SDK Integration

### Configure CoSignerConverter

```python
from safeheron_api_sdk_python.cosigner.co_signer_converter import (
    CoSignerConverter,
    CoSignerResponseV3,
)

# Config for Co-Signer callback
cosigner_config = {
    'coSignerPubKey': 'MIICIjANBgkqhk...',  # Co-Signer identity public key (from export-public-key)
    'approvalCallbackServicePrivateKey': 'MIIJQgIBADANBgk...',  # Your callback private key (base64)
    # OR use PEM file:
    # 'approvalCallbackServicePrivateKeyPemFile': '/path/to/callback_private.pem',
}

converter = CoSignerConverter(cosigner_config)
```

### Flask Approval Callback Service (V3 -- Recommended)

```python
from flask import Flask, request, jsonify
import json
from decimal import Decimal

app = Flask(__name__)

@app.route('/cosigner/callback', methods=['POST'])
def handle_cosigner_callback():
    try:
        raw_body = request.get_json(force=True)

        # Decrypt and verify signature (V3 protocol)
        biz_content = converter.request_v3_convert(raw_body)

        callback_type = biz_content.get('type', '')
        customer_content = biz_content.get('customerContent', {})

        # Determine approval based on business logic
        action = evaluate_approval(callback_type, customer_content)

        # Build encrypted response
        response = CoSignerResponseV3()
        response.action = action  # "APPROVE" or "REJECT"
        response.approvalId = biz_content.get('approvalId', '')

        encrypted_response = converter.response_v3_converter(response)
        return jsonify(encrypted_response)

    except Exception as e:
        app.logger.error(f"Callback processing error: {e}")
        # On error, reject by default for safety
        response = CoSignerResponseV3()
        response.action = "REJECT"
        response.approvalId = ""
        encrypted_response = converter.response_v3_converter(response)
        return jsonify(encrypted_response)


def evaluate_approval(callback_type, content):
    """Implement your business validation logic here."""

    if callback_type == 'TRANSACTION':
        return evaluate_transaction_approval(content)
    elif callback_type == 'MPC_SIGN':
        return evaluate_mpc_sign_approval(content)
    elif callback_type == 'WEB3_SIGN':
        return evaluate_web3_sign_approval(content)

    # Unknown type -- reject
    return "REJECT"


def evaluate_transaction_approval(tx):
    customer_ref_id = tx.get('customerRefId', '')

    # 1. Verify customerRefId exists in your DB
    order = find_order_by_ref_id(customer_ref_id)
    if order is None:
        app.logger.warning(f"REJECT: unknown customerRefId {customer_ref_id}")
        return "REJECT"

    # 2. Verify amount matches
    if Decimal(tx.get('txAmount', '0')) != order['expected_amount']:
        app.logger.warning(f"REJECT: amount mismatch for {customer_ref_id}")
        return "REJECT"

    # 3. Verify destination address matches
    if tx.get('destinationAddress', '').lower() != order['destination_address'].lower():
        app.logger.warning(f"REJECT: destination mismatch for {customer_ref_id}")
        return "REJECT"

    # 4. Check AML risk
    for aml in tx.get('amlList', []):
        if aml.get('riskLevel', '').upper() == 'HIGH':
            return "REJECT"

    return "APPROVE"
```

### Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Regular transaction approval |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval |

---

## TRANSACTION_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `txHash` | String | Transaction hash (if available) |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `transactionType` | String | Transaction type |
| `transactionStatus` | String | Current status |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Sender wallet key |
| `sourceAccountType` | String | e.g. `VAULT_ACCOUNT` |
| `sourceAddress` | String | Sender address |
| `destinationAccountKey` | String | Recipient wallet key |
| `destinationAccountType` | String | e.g. `ONE_TIME_ADDRESS` |
| `destinationAddress` | String | Recipient address |
| `destinationAddressList` | List | Multi-dest address list |
| `estimateFee` | String | Estimated fee |
| `feeCoinKey` | String | Fee coin |
| `note` | String | Transaction note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | AML status: `YES` / `NO` |
| `amlList` | List\<Aml\> | AML assessments (provider, riskLevel, etc.) |
| `createTime` | Long | Unix timestamp (ms) |

---

## WEB3_SIGN_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Web3 sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | String | Web3 wallet account key |
| `sourceAddress` | String | Signing address |
| `createTime` | Long | Unix timestamp (ms) |
| `note` | String | Note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `message` | Object | For personalSign/ethSignTypedData: `{chainId, data}` |
| `messageHash` | Object | For ethSign: `{chainId, data:[hashes]}` |
| `transaction` | Object | For ethSignTransaction: `{chainId, to, value, gasLimit, gasPrice, maxFeePerGas, maxPriorityFeePerGas, nonce, data}` |

---

## MPC_SIGN_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | MPC sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `sourceAccountKey` | String | Wallet key |
| `createTime` | Long | Unix timestamp (ms) |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `signAlg` | String | `Secp256k1` or `Ed25519` |
| `hashs` | List | Hashes to sign: `[{note, hash}]` |
| `dataList` | List | Raw data list: `[{note, data}]` |

---

## Approval Callback Response

Your callback service must return an **encrypted response** using the same AES+RSA scheme, signed with your Callback Private Key:

```json
{
  "action": "APPROVE",
  "approvalId": "tx*******"
}
```

| Field | Type | Values |
|-------|------|--------|
| `action` | String | `APPROVE` or `REJECT` |
| `approvalId` | String | Echo back the incoming `txKey` |

---

## API Co-Signer Deployment Notes

| Topic | Detail                                                                  |
|-------|-------------------------------------------------------------------------|
| Polling interval | v2.x: every 5s; v1.x: every 1s                                          |
| Callback timeout | Response must arrive within 5s of Co-Signer server time                 |
| Production | Callback URL is **required** for production teams                       |
| Test environment | Callback URL is optional                                                |
| KMS support | AWS KMS, GCP KMS, Alibaba Cloud KMS                                    |
| CLI commands | `sudo ./cosigner start`, `sudo ./cosigner stop`, `sudo ./cosigner logs` |
| Export Co-Signer public key | `sudo ./cosigner export-public-key`                                     |
| Minimum server specs | Refer to Safeheron Console deployment guide                             |
| Config changes | Modify `.env` then run `stop` + `start` to reload                       |
| Update Callback URL | Web Console update takes effect within 5 min — no Co-Signer restart needed |
| IP whitelist | Add Co-Signer host IP to the API Key IP whitelist in Console            |

---

## New vs. Old Co-Signer Version Differences

| Aspect | New Version (v2.x+) | Old Version (v1.x) |
|--------|--------------------|--------------------|
| RSA key pairs required | **1 pair** | 2 pairs |
| Polling interval | 5 seconds | 1 second |
| Cancel transaction in DB | Automatic | Manual SQL required |
| Config management | Secrets Manager (recommended) + .env | Config file |

**Old version** requires configuring the public keys `biz_pubkey` and `api_privkey` in two pairs of public and private keys in the configuration file. The new version only requires registering `biz_pubkey` in the Web console.

---

## CLI Commands Reference

```bash
sudo ./cosigner start               # Start Co-Signer
sudo ./cosigner stop                # Stop Co-Signer
sudo ./cosigner setup               # Initial setup (must run before start)
sudo ./cosigner export-public-key   # Export Co-Signer identity public key
./cosigner logs -f                  # Stream live logs
./cosigner logs -s                  # Export logs to file (for Support)
```

---

## Key Pair Configuration Summary

| Key | Owner | Purpose |
|-----|-------|---------|
| **Co-Signer Identity Private Key** | Safeheron (Co-Signer internal) | Signs callback requests sent to your service |
| **Co-Signer Identity Public Key** | Exported via `export-public-key` | You use this to verify callback request signatures |
| **Callback Public Key** | You generate & upload to Console | Safeheron Co-Signer needs to use this public key to verify the signature in the Callback's response data |
| **Callback Private Key** | You hold securely | Your callback service needs to use this private key to sign the Callback response body |

---

## Common Co-Signer Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Illegal IP` in logs | Co-Signer host IP not in API Key whitelist | Add IP to whitelist in Console, restart Co-Signer |
| `The final private key fragment is not exists` | Co-Signer not yet activated | Normal — proceed with activation workflow |
| Transaction stays in `SUBMITTED` | Co-Signer not running or not polling | Check `./cosigner logs -f` for errors |
| `Timestamp out of range` | Server clock skew > 5s | Sync Co-Signer server time with NTP |
| Docker login failure | Token expired or firewall blocks `registry.gitlab.com` | Re-download CLI from Console; check firewall rules |

---

## Official SDK Demo

See the SDK repository for a working callback demo:
https://github.com/Safeheron/safeheron-api-sdk-python

---

## Security Deployment Requirements (API Co-Signer Host)

These requirements apply to the server where API Co-Signer is deployed. Non-compliance can result in asset loss.

### Mandatory Security Principles

| Principle | Requirement |
|-----------|------------|
| Strong passwords | All accounts (DB, cloud, OS) must use randomly generated strong passwords — no weak/default credentials |
| MFA everywhere | Enable 2FA/MFA on every account and service that supports it |
| Minimum privilege | Each account/role has only the permissions needed for its specific function |
| Minimum exposure | Close all unnecessary ports. Only port `9999` (business) needs to be open |
| Secure secret storage | Store all secrets (passwords, private keys, Secret Keys) in a dedicated secrets manager (e.g., 1Password, AWS Secrets Manager). Never transmit secrets via chat/email/documents |

### Host Security Controls

| Control | Description |
|---------|-------------|
| Network isolation | Deploy Co-Signer in a **private isolated network**. No public internet inbound access |
| IP whitelist | Only your internal systems can reach the Co-Signer host |
| MFA (host login) | All host access requires MFA |
| CWPP | Cloud Workload Protection Platform — real-time threat monitoring on the Co-Signer instance |
| EDR/XDR | Endpoint/Extended Detection & Response — continuous threat detection and automated response |
| SIEM | Collect host login and system activity logs; configure real-time alerts |
| PAM | Privileged Access Management — audit all privileged operations on the host |

### Required Network Connectivity (Outbound Only)

API Co-Signer has **zero inbound** from the internet. Configure firewall to allow only these outbound connections:

**Always required (runtime):**
- `https://api.safeheron.vip:443`
- `wss://gm-gateway.safeheron.vip:443`
- MySQL (your DB host)
- Approval Callback Service (your internal service)

**If using AWS KMS:**
- `https://*.amazonaws.com:443`
- `https://*.amazonaws.com.cn:443`

**If using Alibaba Cloud KMS:**
- `https://*.cryptoservice.kms.aliyuncs.com:443`

**If using GCP Cloud KMS:**
- `https://*.cloudkms.googleapis.com:443`
- `https://cloudkms.googleapis.com:443`
- `http://metadata.google.internal:80`

**Only during install/update:**
- `registry.gitlab.com:443`
- `https://prod-safeheron-openapi.s3.ap-east-1.amazonaws.com:443`

**Only on first install (Docker bootstrap):**
- `https://get.docker.com:443`
- `https://download.docker.com:443`
- `https://api.github.com:443`
- `https://github.com:443`

### Callback Service Security

The Approval Callback Service must:

1. **Verify the Co-Signer identity signature** on every request before processing
2. **Only accept connections from the Co-Signer host IP** — configure firewall/IP whitelist on the callback service
3. **Validate `customerRefId`** — reject any callback for a `customerRefId` not found in your DB
4. **Validate amount and destination** — cross-check against the original withdrawal order
5. **Respond within 5 seconds** — Co-Signer will time out otherwise (sync server time via NTP)
