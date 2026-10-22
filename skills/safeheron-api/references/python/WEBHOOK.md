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
   `bizContent=...&key=...&timestamp=...`
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
    # Base64 body ONLY -- no '-----BEGIN ... KEY-----' / '-----END ... KEY-----' lines: SDK 1.1.25 adds them itself,
    # so passing a full .pem file's text here fails every verification (see GETTING_STARTED.md / ERROR_CODES.md)
    'webHookRsaPrivateKey': '${WEBHOOK_RSA_PRIVATE_KEY}',           # todo Replace with the WebHook RSA private key you read from Vault/KMS
    # OR point to a PEM file (headers included) -- the SDK loads it as-is:
    # 'webHookRsaPrivateKeyPemFile': '/path/to/webhook_private.pem',
}

webhook_converter = WebhookConverter(webhook_config)
```

### Step 2 -- Flask Webhook Handler

```python
from flask import Flask, request, jsonify
import hashlib
import json

# Placeholders you must implement: `inbox` (durable store: save_if_absent / mark_processed, unique on event_id),
# `db.transaction()` (your DB transaction context manager), and a worker that calls process_inbox_event()
# for unprocessed rows.

app = Flask(__name__)

@app.route('/safeheron/webhook', methods=['POST'])
def handle_webhook():
    try:
        raw_body = request.get_json(force=True)
        # SDK 1.1.25 converter() pops 'sig', 'key', 'rsaType', 'aesType' from the dict you pass in --
        # pass a copy if you also want to keep the original envelope (e.g. for audit)
        envelope = dict(raw_body)

        # converter() handles everything internally:
        #   1. Verifies RSA signature -- raises Exception("rsa verify: false") if invalid
        #   2. Decrypts AES key using your RSA private key
        #   3. Decrypts bizContent
        #   4. Returns the decrypted bizContent as a dict: {"eventType": ..., "eventDetail": {...}}
        # The SDK raises a plain Exception for every failure, so treat any error here as "not trusted"
        biz_content = webhook_converter.converter(envelope)
    except Exception:
        app.logger.exception("Webhook signature verification / decryption FAILED -- possible tampering or wrong key")
        return jsonify({'code': '401', 'message': 'UNAUTHORIZED'}), 401

    try:
        event_type = biz_content.get('eventType', '')
        # Business fields (txKey, transactionStatus, ...) live under eventDetail, NOT at the top level
        event_detail = biz_content.get('eventDetail') or {}

        # Persist the verified event DURABLY before acking (DB inbox table / durable MQ).
        # Do NOT ack after handing it to an in-memory queue or a background thread: a crash loses the event,
        # and TOKEN_BLACKLIST_ADDRESS_HIT cannot be re-pushed by Safeheron (not supported by resend_failed).
        # Derive the dedup key from the DECRYPTED event (the docs do not guarantee the encrypted bizContent
        # is byte-identical across retries); a business key such as txKey + transactionStatus also works.
        canonical = json.dumps({'eventType': event_type, 'eventDetail': event_detail},
                               sort_keys=True, separators=(',', ':'), ensure_ascii=False)
        event_id = hashlib.sha256(canonical.encode('utf-8')).hexdigest()
        inbox.save_if_absent(event_id, event_type, event_detail)   # INSERT ... ON CONFLICT DO NOTHING
    except Exception:
        # Internal error (parse / persist) -- a non-SUCCESS response makes Safeheron retry 6 times
        # (30s, 1m, 5m, 1h, 12h, 24h); after that use resend_failed (every event type except
        # TOKEN_BLACKLIST_ADDRESS_HIT) or REST polling
        app.logger.exception("Webhook processing error")
        return jsonify({'code': '500', 'message': 'INTERNAL_ERROR'}), 500

    # Ack ONLY after the event is verified and durably stored.
    # Body per docs: {"code": "200", "message": "SUCCESS"}; anything else is treated as a failed delivery and retried
    return jsonify({'code': '200', 'message': 'SUCCESS'})


def process_inbox_event(row):
    """Run by a worker over unprocessed inbox rows. Business side effects and the 'processed' mark
    are committed in ONE database transaction; on failure the row stays unprocessed and is retried.
    Never commit a dedup row first and process afterwards -- a failure would then lose the event."""
    event_type, event_detail = row.event_type, row.event_detail
    with db.transaction():
        if event_type in ('TRANSACTION_CREATED',
                          'TRANSACTION_CUSTOMIZED_CONFIRMING',
                          'TRANSACTION_STATUS_CHANGED'):
            handle_transaction_event(event_detail)

        elif event_type in ('MPC_SIGN_CREATED', 'MPC_SIGN_STATUS_CHANGED'):
            handle_mpc_sign_event(event_detail)

        elif event_type in ('WEB3_SIGN_CREATED', 'WEB3_SIGN_STATUS_CHANGED'):
            handle_web3_sign_event(event_detail)

        elif event_type == 'ILLEGAL_IP_REQUEST':
            alert_security_team("API called from illegal IP", event_detail)

        elif event_type == 'NO_MATCHING_TRANSACTION_POLICY':
            alert_ops_team("Transaction blocked: no matching policy", event_detail)

        elif event_type == 'GAS_BALANCE_WARNING':
            alert_ops_team("Gas balance warning", event_detail)

        elif event_type == 'AML_KYT_ALERT':
            alert_ops_team("KYT alert notification", event_detail)

        elif event_type == 'TOKEN_BLACKLIST_ADDRESS_HIT':
            # A team address was frozen by the token issuer (e.g. Tether) -- stop using it and check affected assets
            alert_security_team("Address frozen by token issuer", event_detail)

        elif event_type in ('WHITELIST_ADDED', 'WHITELIST_UPDATED', 'WHITELIST_REMOVED'):
            sync_whitelist(event_type, event_detail)

        inbox.mark_processed(row.event_id)


def handle_transaction_event(event):
    tx_key = event.get('txKey')
    status = event.get('transactionStatus')
    app.logger.info(f"Transaction {tx_key}: {status}")
    # Deposits: for UTXO coins read event['destinationAddressList'] ([{address, amount}, ...]) --
    # `destinationAddress` is not populated for UTXO coins.
    # transactionDirection: INFLOW / OUTFLOW / INTERNAL_TRANSFER (both sides are your vault accounts) --
    # a withdrawal to another user's deposit address on your platform arrives as INTERNAL_TRANSFER.


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


def sync_whitelist(event_type, event):
    # Keep your local whitelist copy in sync (fields: see "Whitelist Events" below)
    app.logger.info(f"{event_type}: {event.get('whitelistKey')} {event.get('address')}")
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
| `destinationAddress` | str | Recipient address (for UTXO coins use `destinationAddressList`) |
| `destinationAddressList` | list | Destination list `[{address, amount, ...}]` -- use this for UTXO coins |
| `transactionDirection` | str | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER` (source and destination are both `VAULT_ACCOUNT`) |
| `txFee` | str | Transaction fee paid |
| `blockHeight` | int | Confirmed block height |
| `completedTime` | int | Unix timestamp (ms) of completion |
| `customerExt1` | str | Custom field 1 |
| `customerExt2` | str | Custom field 2 |
| `amlLock` | str | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |

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
| `WHITELIST_ADDED` | A newly added whitelist entry was approved and became effective |
| `WHITELIST_UPDATED` | An update to an effective whitelist entry was approved |
| `WHITELIST_REMOVED` | An effective whitelist entry was removed (deleted) |

`eventDetail` fields (dict):

| Field | ADDED | UPDATED | REMOVED | Description |
|-------|:-----:|:-------:|:-------:|-------------|
| `whitelistKey` | ✓ | ✓ | ✓ | Whitelist unique identifier |
| `whitelistName` | ✓ | ✓ | ✓ | Whitelist name |
| `chainType` | ✓ | ✓ | ✓ | The public blockchain of the whitelisted address |
| `address` | ✓ | ✓ | ✓ | Whitelisted address |
| `memo` | ✓ | ✓ (updated memo) | ✓ | Memo of the address |
| `previousWhitelistName` | | ✓ | | Original whitelist name |
| `previousAddress` | | ✓ | | Original whitelisted address |
| `previousMemo` | | ✓ | | Original address memo |
| `approvalTime` | ✓ | ✓ | | Time the whitelist change was approved (int, ms) |

---

### Security & System Events

| Event Type | Trigger |
|------------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching approval policy |
| `GAS_BALANCE_WARNING` | Gas station balance below threshold |
| `AML_KYT_ALERT` | KYT Alert Notification |
| `TOKEN_BLACKLIST_ADDRESS_HIT` | An address was added to a token issuer's blacklist (e.g. Tether) — the token on that address is frozen |

`TOKEN_BLACKLIST_ADDRESS_HIT` `eventDetail` fields (dict):

| Field | Type | Description |
|-------|------|-------------|
| `blockChain` | str | Blockchain |
| `contract` | str | Contract address |
| `address` | str | Address |
| `workspaceKey` | str | Workspace Key, the only workspace identifier |
| `workspaceKeyName` | str | Workspace Name |
| `accountKey` | str | Account Key, the only account identifier |
| `accountName` | str | Account name |
| `coinKey` | str | Coin key |
| `issuer` | str | Issuer |
| `lastTxHash` | str | Transaction hash |
| `lastUpdateTime` | int | Last update time (UNIX millisecond timestamp) |

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

> Supported event types (published docs): `TRANSACTION_CREATED`, `TRANSACTION_STATUS_CHANGED`, `MPC_SIGN_CREATED`, `MPC_SIGN_STATUS_CHANGED`, `WEB3_SIGN_CREATED`, `WEB3_SIGN_STATUS_CHANGED`, `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, `WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`, `GAS_BALANCE_WARNING`, `AML_KYT_ALERT` — more types may be added later. **`TOKEN_BLACKLIST_ADDRESS_HIT` cannot be re-pushed.** Re-push is a recovery aid, not a substitute: still persist every event durably before acking.

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes. Only webhooks from the **past 7 days** can be resent — requests with timestamps older than 7 days will silently return empty results.

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
`30s -> 1m -> 5m -> 1h -> 12h -> 24h` -- 6 retries after the first delivery, then it stops.

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

Safeheron may deliver the same event multiple times (retries and `resend_*` re-pushes). Always check if you've already processed a given `txKey` + `transactionStatus` combination before acting, and commit that check-and-mark **in the same database transaction** as the business side effect (credit, status change). Never insert-and-commit a dedup row first and process afterwards — if processing fails, the retry is then treated as a duplicate and the event is lost.

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

- Verify, persist the event durably (DB inbox / durable MQ), then return `{"code": "200", "message": "SUCCESS"}` -- process asynchronously from that durable store. In-memory queues or background threads lose events on a crash.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- Schedule a periodic call to the transaction query API as a safety net.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
