# Webhook Integration Reference

## Overview

Safeheron pushes real-time event notifications to your configured webhook URL when transactions, MPC sign requests, Web3 sign requests, whitelist changes, or security events occur.

Configure the webhook URL in: **Safeheron Web Console -> Settings -> API -> Webhook**

- Supported protocols: HTTP or HTTPS
- URL must start with `http://` or `https://`
- Note: Some public domains (ngrok, webhook.site) may be blocked by Safeheron's network filters

---

## Webhook Payload Structure

Webhook payloads use the **same AES+RSA encryption scheme** as API responses. The encrypted payload is a JSON body with these fields:

```json
{
  "timestamp": "1623038312088",
  "sig":        "<RSA-signature-base64>",
  "key":        "<RSA-encrypted-AES-key-base64>",
  "bizContent": "<AES-encrypted-payload-base64>",
  "rsaType":    "ECB_OAEP",
  "aesType":    "GCM_NOPADDING"
}
```

### Decryption Steps (same as API response decryption)

1. Build signature string: sort all fields by key ascending (exclude `rsaType`, `aesType`):
   `bizContent=...&code=...&key=...&timestamp=...`
2. Verify `sig` using **Safeheron's RSA public key** -- reject if invalid.
3. Decrypt `key` field using **your RSA private key** -> 48 bytes (AES key + IV).
4. Decrypt `bizContent` using AES/GCM/NoPadding -> plaintext JSON event payload.

---

## Webhook Handler Example (Flask)

### Step 1 -- Configure WebhookConverter

```python
from safeheron_api_sdk_python.webhook.webhook_converter import WebhookConverter

# Config for webhook decryption
webhook_config = {
    'safeheronWebHookRsaPublicKey': '${SAFEHERON_WEBHOOK_RSA_PUBLIC_KEY}',  # todo Replace with the Safeheron WebHook RSA public key from Safeheron Console
    'webHookRsaPrivateKey': '${WEBHOOK_RSA_PRIVATE_KEY}',           # todo Replace with the WebHook RSA private key you read from Vault/KMS
    # OR use PEM file:
    # 'webHookRsaPrivateKeyPemFile': '/path/to/webhook_private.pem',
}

webhook_converter = WebhookConverter(webhook_config)
```

### Step 2 -- Flask Webhook Handler

```python
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/safeheron/webhook', methods=['POST'])
def handle_webhook():
    try:
        raw_body = request.get_json(force=True)

        # converter() handles everything internally:
        #   1. Verifies RSA signature -- raises Exception if invalid
        #   2. Decrypts AES key using your RSA private key
        #   3. Decrypts bizContent
        #   4. Returns parsed JSON dict
        biz_content = webhook_converter.converter(raw_body)

        event_type = biz_content.get('eventType', '')

        # Route by eventType -- offload to async queue in production
        if event_type in ('TRANSACTION_CREATED',
                          'TRANSACTION_CUSTOMIZED_CONFIRMING',
                          'TRANSACTION_STATUS_CHANGED'):
            handle_transaction_event(biz_content)

        elif event_type in ('MPC_SIGN_CREATED', 'MPC_SIGN_STATUS_CHANGED'):
            handle_mpc_sign_event(biz_content)

        elif event_type in ('WEB3_SIGN_CREATED', 'WEB3_SIGN_STATUS_CHANGED'):
            handle_web3_sign_event(biz_content)

        elif event_type == 'ILLEGAL_IP_REQUEST':
            alert_security_team("API called from illegal IP", biz_content)

        elif event_type == 'NO_MATCHING_TRANSACTION_POLICY':
            alert_ops_team("Transaction blocked: no matching policy", biz_content)

        elif event_type == 'GAS_BALANCE_WARNING':
            alert_ops_team("Gas balance warning", biz_content)

    except Exception as e:
        app.logger.error(f"Webhook processing error: {e}")

    # Always respond HTTP 200 immediately -- never block
    return jsonify({'code': '200', 'message': 'SUCCESS'})


def handle_transaction_event(event):
    tx_key = event.get('txKey')
    status = event.get('transactionStatus')
    app.logger.info(f"Transaction {tx_key}: {status}")
    # Process asynchronously in production


def handle_mpc_sign_event(event):
    tx_key = event.get('txKey')
    status = event.get('transactionStatus')
    app.logger.info(f"MPC Sign {tx_key}: {status}")


def handle_web3_sign_event(event):
    tx_key = event.get('txKey')
    status = event.get('transactionStatus')
    app.logger.info(f"Web3 Sign {tx_key}: {status}")


def alert_security_team(msg, event):
    app.logger.critical(f"SECURITY: {msg} - {event}")


def alert_ops_team(msg, event):
    app.logger.warning(f"OPS: {msg} - {event}")
```

---

## Event Types

### Transaction Events

| Event Type | Trigger |
|------------|---------|
| `TRANSACTION_CREATED` | New transaction submitted |
| `TRANSACTION_STATUS_CHANGED` | Any status transition |
| `TRANSACTION_CUSTOMIZED_CONFIRMING` | Custom block confirmation count reached |

**Transaction Event Payload Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `customerRefId` | str | Your reference ID |
| `txHash` | str | On-chain transaction hash |
| `transactionStatus` | str | Current status |
| `transactionSubStatus` | str | Sub-status |
| `coinKey` | str | Coin identifier |
| `txAmount` | str | Amount (string) |
| `sourceAccountKey` | str | Sender wallet |
| `sourceAddress` | str | Sender address |
| `destinationAddress` | str | Recipient address |
| `amlLock` | str | AML status: `YES` / `NO` |

---

### MPC Sign Events

| Event Type | Trigger |
|------------|---------|
| `MPC_SIGN_CREATED` | New MPC sign request created |
| `MPC_SIGN_STATUS_CHANGED` | Status transition |

---

### Web3 Sign Events

| Event Type | Trigger |
|------------|---------|
| `WEB3_SIGN_CREATED` | New Web3 sign request |
| `WEB3_SIGN_STATUS_CHANGED` | Status transition |

---

### Whitelist Events

| Event Type | Trigger |
|------------|---------|
| `WHITELIST_ADDED` | New whitelist entry created |
| `WHITELIST_UPDATED` | Whitelist entry modified |
| `WHITELIST_REMOVED` | Whitelist entry deleted |

---

### Security & System Events

| Event Type | Trigger |
|------------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching approval policy |
| `GAS_BALANCE_WARNING` | Gas station balance below threshold |
| `AML_KYT_ALERT` | KYT Alert Notification |

---

## Safeheron Egress IPs (Webhook Source)

Configure your webhook server to **only accept connections from**:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

Reject all other sources at the network/firewall level, not just in application code.

---

## Re-push API

Two methods for recovering missed webhook events:

### `resend_webhook` — Re-push latest event for one transaction

Re-pushes only the **most recent** webhook event for a given transaction (not full history).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | str | Yes | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `txKey` | str | Yes | Transaction key |

```python
from safeheron_api_sdk_python.api.webhook_api import WebhookApi, ResendWebhookRequest

webhook_api = WebhookApi(config)

req = ResendWebhookRequest()
req.category = 'TRANSACTION'
req.txKey = 'your-tx-key'
webhook_api.resend_webhook(req)
```

### `resend_failed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes.

> **Warning:** Your handler must be idempotent. `resend_failed` may re-deliver intermediate-status events (e.g. `CONFIRMING`) even if you already received a terminal status (`COMPLETED`). Never roll back a terminal state.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `startTime` | int (ms) | Yes | Start of time range |
| `endTime` | int (ms) | Yes | End of time range (max interval: 1 hour) |

```python
from safeheron_api_sdk_python.api.webhook_api import ResendFailedRequest
import time

failed_req = ResendFailedRequest()
failed_req.startTime = int(time.time() * 1000) - 3600000
failed_req.endTime = int(time.time() * 1000)
result = webhook_api.resend_failed(failed_req)
# result['messagesCount']: number of events triggered
```

## Retry Schedule

Safeheron automatic retry schedule on non-200 response:
`30s -> 1m -> 5m -> 1h -> 12h -> 24h` (7 total attempts, then stops)

---

## Security Requirements for Webhook Endpoints

### 1. Verify Signature Before Processing (Mandatory)

The `WebhookConverter.converter()` method handles signature verification automatically. If the signature is invalid, it raises an `Exception`. Never process the payload if this fails.

### 2. Avoid Status Rollback

Webhook events may arrive out of order. Your handler **must never downgrade a status**:

```python
TERMINAL_STATUSES = {'COMPLETED', 'SIGN_COMPLETED', 'FAILED', 'REJECTED', 'CANCELLED'}

def should_update_status(current_status, new_status):
    """Terminal statuses are final -- never overwrite them."""
    if current_status in TERMINAL_STATUSES:
        return False
    return True
```

### 3. Idempotency by `txKey`

Safeheron may deliver the same event multiple times. Always check if you've already processed a given `txKey` + `transactionStatus` combination before acting.

### 4. Anti-Dust Attack Filtering for Deposits

```python
from decimal import Decimal

min_deposit = get_minimum_deposit_amount(coin_key)
tx_amount = Decimal(event.get('txAmount', '0'))

if tx_amount < min_deposit:
    app.logger.info(f"Ignoring dust deposit: {tx_amount} {coin_key} (below minimum {min_deposit})")
    return
```

---

## Best Practices

- Return HTTP 200 as fast as possible -- offload all processing to a queue/async worker.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- Schedule a periodic call to the transaction query API as a safety net.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
