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

### Decryption Steps

1. Build signature string: sort all fields by key ascending (exclude `rsaType`, `aesType`):
   `bizContent=...&code=...&key=...&timestamp=...`
2. Verify `sig` using **Safeheron's RSA public key** -- reject if invalid.
3. Decrypt `key` field using **your RSA private key** -> 48 bytes (AES key + IV).
4. Decrypt `bizContent` using AES/GCM/NoPadding -> plaintext JSON event payload.

---

## Webhook Handler Example (Using SDK WebHookConverter)

### Step 1 -- Configure WebHookConverter

```typescript
import { WebHookConverter } from '@safeheron/api-sdk';
import { SafeheronWebHookConfig } from '@safeheron/api-sdk';
import { readFileSync } from 'fs';
import path from 'path';

const webhookConfig: SafeheronWebHookConfig = {
  webHookRsaPrivateKey: readFileSync(path.resolve('./keys/webhook_private.pem'), 'utf8'),
  safeheronWebHookRsaPublicKey: readFileSync(path.resolve('./keys/safeheron_webhook_public.pem'), 'utf8'),
};

const converter = new WebHookConverter(webhookConfig);
```

### Step 2 -- Express.js Webhook Handler

```typescript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/safeheron/webhook', (req, res) => {
  try {
    // convertWebHook() handles everything internally:
    //   1. Verifies RSA signature -- throws SafeheronError if invalid
    //   2. Decrypts AES key using your RSA private key
    //   3. Decrypts bizContent
    const decryptedContent = converter.convertWebHook(req.body);
    const event = JSON.parse(decryptedContent);

    // Route by eventType -- offload to async queue in production
    switch (event.eventType) {
      case 'TRANSACTION_CREATED':
      case 'TRANSACTION_CUSTOMIZED_CONFIRMING':
      case 'TRANSACTION_STATUS_CHANGED':
        handleTransactionEvent(event);
        break;
      case 'MPC_SIGN_CREATED':
      case 'MPC_SIGN_STATUS_CHANGED':
        handleMpcSignEvent(event);
        break;
      case 'WEB3_SIGN_CREATED':
      case 'WEB3_SIGN_STATUS_CHANGED':
        handleWeb3SignEvent(event);
        break;
      case 'ILLEGAL_IP_REQUEST':
      case 'NO_MATCHING_TRANSACTION_POLICY':
      case 'GAS_BALANCE_WARNING':
      case 'AML_KYT_ALERT':
        handleSecurityEvent(event);
        break;
    }
  } catch (e) {
    console.error('Webhook processing error:', e);
  }

  // Always respond HTTP 200 immediately -- never block
  res.json({ code: 200, message: 'SUCCESS' });
});
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
| `txKey` | string | Safeheron transaction key |
| `customerRefId` | string | Your reference ID |
| `txHash` | string | On-chain transaction hash |
| `transactionStatus` | string | Current status |
| `transactionSubStatus` | string | Sub-status |
| `coinKey` | string | Coin identifier |
| `txAmount` | string | Amount (string) |
| `sourceAccountKey` | string | Sender wallet |
| `sourceAddress` | string | Sender address |
| `destinationAddress` | string | Recipient address |
| `txFee` | string | Transaction fee paid |
| `createTime` | number | Unix timestamp (ms) |
| `completedTime` | number | Unix timestamp (ms) of completion |
| `amlLock` | string | AML status: `YES` / `NO` |

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

## Re-push API

Two methods for recovering missed webhook events:

### `resendWebhook` — Re-push latest event for one transaction

Re-pushes only the **most recent** webhook event for a given transaction (not full history).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | string | No | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `txKey` | string | No | Transaction key |

```typescript
import { WebhookApi } from '@safeheron/api-sdk';

const webhookApi = new WebhookApi(config);

await webhookApi.resendWebhook({
  category: 'TRANSACTION',
  txKey: 'your-tx-key',
});
```

### `resendFailed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes.

> **Warning:** Your handler must be idempotent. `resendFailed` may re-deliver intermediate-status events (e.g. `CONFIRMING`) even if you already received a terminal status (`COMPLETED`). Never roll back a terminal state.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `startTime` | number (ms) | Yes | Start of time range |
| `endTime` | number (ms) | Yes | End of time range (max interval: 1 hour) |

```typescript
const result = await webhookApi.resendFailed({
  startTime: Date.now() - 3600000,
  endTime: Date.now(),
});
// result.messagesCount: number of events triggered
```

Safeheron automatic retry schedule on non-200 response:
`30s -> 1m -> 5m -> 1h -> 12h -> 24h` (7 total attempts, then stops)

---

## Custom Block Confirmation Count

Configure custom confirmation thresholds per blockchain in Safeheron Console. Once enabled, it applies to **both incoming and outgoing** transactions on that chain. A `TRANSACTION_CUSTOMIZED_CONFIRMING` event fires when the threshold is reached.

---

## Webhook Filtering

- **No amount filtering** -- any successful deposit, regardless of amount, triggers a webhook (including dust attacks / small transfers).
- All transaction types trigger webhooks: ordinary transfers, Auto Sweep, gas refill, speed-up, batch transfers.

---

## Security Requirements for Webhook Endpoints

### 1. Verify Signature Before Processing (Mandatory)

Every webhook payload **must be signature-verified** before any business logic runs. The SDK's `WebHookConverter.convertWebHook()` handles this automatically -- it throws `SafeheronError` on invalid signatures.

### 2. IP Whitelist -- Only Accept Safeheron's Egress IPs

Configure your webhook server to **only accept connections from**:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

### 3. Avoid Status Rollback

Webhook events may arrive out of order. Your handler **must never downgrade a status**:

```typescript
function shouldUpdateStatus(currentStatus: string, newStatus: string): boolean {
  const terminalStatuses = new Set(['COMPLETED', 'SIGN_COMPLETED', 'FAILED', 'REJECTED', 'CANCELLED']);
  if (terminalStatuses.has(currentStatus)) {
    return false;  // already in terminal state -- ignore late events
  }
  return true;
}
```

### 4. Idempotency by `txKey`

Safeheron may deliver the same event multiple times (retry on non-200). Always check if you've already processed a given `txKey` + `transactionStatus` combination before acting.

### 5. Anti-Dust Attack Filtering for Deposits

```typescript
const minDeposit = getMinimumDepositAmount(coinKey);
const txAmount = parseFloat(event.txAmount);

if (txAmount < minDeposit) {
  console.log(`Ignoring dust deposit: ${txAmount} ${coinKey} (below minimum ${minDeposit})`);
  return;
}
```

### 6. Subscribe to Security Events

```typescript
switch (event.eventType) {
  case 'ILLEGAL_IP_REQUEST':
    alertSecurityTeam('Safeheron API called from illegal IP', event);
    break;
  case 'NO_MATCHING_TRANSACTION_POLICY':
    alertOpsTeam('Transaction blocked: no matching policy', event);
    break;
  case 'GAS_BALANCE_WARNING':
    alertOpsTeam('Gas balance warning', event);
    break;
  case 'AML_KYT_ALERT':
    alertOpsTeam('KYT alert notification', event);
    break;
}
```

---

## Best Practices

- Return HTTP 200 as fast as possible -- offload all processing to a queue/async worker.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- Schedule a periodic call to `oneTransactions()` as a safety net.
- Auto Sweep, Gas refill, speed-up, and batch transfers all generate standard webhook events.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
