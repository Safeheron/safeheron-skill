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
   `bizContent=...&key=...&timestamp=...`
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
  webHookRsaPrivateKey: '${WEBHOOK_RSA_PRIVATE_KEY}',//todo Replace with the WebHook RSA private key you read from Vault/KMS
  safeheronWebHookRsaPublicKey: '${SAFEHERON_WEBHOOK_RSA_PUBLIC_KEY}',//todo Replace with the Safeheron WebHook RSA public key from Safeheron Console
};

const converter = new WebHookConverter(webhookConfig);
```

### Step 2 -- Express.js Webhook Handler

> **Durability rule:** ack `SUCCESS` only after the verified event is **durably stored** (a DB inbox table or a durable MQ). An in-memory queue or a fire-and-forget `setImmediate`/`Promise` loses the event if the process crashes after the ack — and `TOKEN_BLACKLIST_ADDRESS_HIT` **cannot be re-pushed** by Safeheron at all (see [Re-push API](#re-push-api)); other event types can be re-pushed with `resendFailed`, but only within 7 days and 1-hour windows.

```typescript
import express from 'express';
import { createHash } from 'crypto';

const app = express();
app.use(express.json());

app.post('/safeheron/webhook', async (req, res) => {
  try {
    // convertWebHook() handles everything internally:
    //   1. Verifies RSA signature -- throws Error if invalid
    //   2. Decrypts AES key using your RSA private key
    //   3. Decrypts bizContent
    const decryptedContent = converter.convertWebHook(req.body);
    const event = JSON.parse(decryptedContent);

    // The payload has no event id: use a hash of the decrypted content as the inbox key, so exact
    // duplicates (retries / re-push) are stored once while each status change is a new row
    const eventId = createHash('sha256').update(decryptedContent).digest('hex');

    // Durable write BEFORE the ack. inbox.saveIfAbsent = INSERT ... ON CONFLICT (event_id) DO NOTHING
    await inbox.saveIfAbsent(eventId, event.eventType, decryptedContent);
  } catch (e) {
    if (e instanceof Error && e.message === 'webhook signature verification failed') {
      // Signature verification failed -- never ack; the event is not trusted
      console.error('Webhook signature verification FAILED -- possible tampering:', e);
      return res.status(401).end();
    }
    // Internal error (decrypt / parse / inbox write) -- a non-SUCCESS response makes Safeheron retry
    // (30s, 1m, 5m, 1h, 12h, 24h); after that use resendFailed (every event type except TOKEN_BLACKLIST_ADDRESS_HIT)
    console.error('Webhook processing error:', e);
    return res.status(500).end();
  }

  // Ack ONLY after the event is durably stored. Body per docs: code is the string '200'
  res.json({ code: '200', message: 'SUCCESS' });
});
```

### Step 3 -- Process the Inbox (separate worker)

Process each stored event **in one DB transaction together with its business side-effects**, then mark it done. If processing fails, roll back and leave the row pending so the worker retries it. Never "insert a dedup row and commit, then process" — a failure after that commit loses the event, because the retry is treated as a duplicate.

```typescript
// Runs periodically / on notify. db.transaction() is your DB client's transaction helper.
async function processInbox() {
  for (const row of await inbox.fetchPending(100)) {
    try {
      await db.transaction(async (trx) => {
        const event = JSON.parse(row.payload);
        switch (event.eventType) {
          case 'TRANSACTION_CREATED':
          case 'TRANSACTION_CUSTOMIZED_CONFIRMING':
          case 'TRANSACTION_STATUS_CHANGED':
            await handleTransactionEvent(trx, event.eventDetail);
            break;
          case 'MPC_SIGN_CREATED':
          case 'MPC_SIGN_STATUS_CHANGED':
            await handleMpcSignEvent(trx, event.eventDetail);
            break;
          case 'WEB3_SIGN_CREATED':
          case 'WEB3_SIGN_STATUS_CHANGED':
            await handleWeb3SignEvent(trx, event.eventDetail);
            break;
          case 'WHITELIST_ADDED':
          case 'WHITELIST_UPDATED':
          case 'WHITELIST_REMOVED':
            await handleWhitelistEvent(trx, event.eventDetail);
            break;
          case 'ILLEGAL_IP_REQUEST':
          case 'NO_MATCHING_TRANSACTION_POLICY':
          case 'GAS_BALANCE_WARNING':
          case 'AML_KYT_ALERT':
            await handleSecurityEvent(trx, event);
            break;
          case 'TOKEN_BLACKLIST_ADDRESS_HIT':
            // One of your addresses was added to a token issuer's blacklist (e.g. Tether) — alert ops and stop using it
            await handleTokenFrozenEvent(trx, event.eventDetail);
            break;
          default:
            // Unknown / newly added event types: keep the row for later handling, just log
            console.warn('Unhandled webhook eventType:', event.eventType);
        }
        await inbox.markDone(trx, row.eventId);  // same transaction as the side-effects
      });
    } catch (e) {
      console.error('Inbox processing failed, will retry:', row.eventId, e);  // row stays pending
    }
  }
}
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
| `destinationAddress` | string | Recipient address. **UTXO coins:** use `destinationAddressList[]` (`address` / `amount` / `memo`) instead — `destinationAddress` may be empty |
| `destinationAddressList` | array | Multi-destination list (UTXO coins / batch transfers): `address`, `amount`, `memo` |
| `transactionDirection` | string | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER` — a withdrawal to another address inside your own Safeheron team arrives as `INTERNAL_TRANSFER`, handle it as the withdrawal's completion and (if applicable) the receiver's deposit **only** when `transactionType = NORMAL` and it matches one of your withdrawal orders; `AUTO_FUEL` / `AUTO_SWEEP` / `UTXO_COLLECTION` internal transfers are never deposits (see `classifyTransaction` in BUSINESS_PATTERNS.md) |
| `txFee` | string | Transaction fee paid |
| `blockHeight` | number | Confirmed block height |
| `createTime` | number | Unix timestamp (ms) |
| `completedTime` | number | Unix timestamp (ms) of completion |
| `customerExt1` | string | Custom field 1 |
| `customerExt2` | string | Custom field 2 |
| `amlLock` | string | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |

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
| `WHITELIST_ADDED` | A newly added whitelist entry has been approved and becomes effective |
| `WHITELIST_UPDATED` | The update of an existing, effective whitelist entry has been approved |
| `WHITELIST_REMOVED` | An effective whitelist entry has been removed (deleted) |

**`eventDetail` fields:**

| Field | WHITELIST_ADDED | WHITELIST_UPDATED | WHITELIST_REMOVED | Description |
|-------|:-:|:-:|:-:|-------------|
| `whitelistKey` | ✓ | ✓ | ✓ | Whitelist unique identifier |
| `whitelistName` | ✓ | ✓ | ✓ | Whitelist name |
| `chainType` | ✓ | ✓ | ✓ | The public blockchain of the whitelisted address |
| `address` | ✓ | ✓ | ✓ | Whitelisted address |
| `memo` | ✓ | ✓ (updated) | ✓ | Address memo |
| `previousWhitelistName` | | ✓ | | Original whitelist name |
| `previousAddress` | | ✓ | | Original whitelisted address |
| `previousMemo` | | ✓ | | Original address memo |
| `approvalTime` | ✓ | ✓ | | The time when the whitelist is approved (UNIX ms) |

---

### Security & System Events

| Event Type | Trigger |
|------------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching approval policy |
| `GAS_BALANCE_WARNING` | Gas Service balance below the alert threshold (at most once every 30 minutes) |
| `AML_KYT_ALERT` | KYT Alert Notification |
| `TOKEN_BLACKLIST_ADDRESS_HIT` | An address was added to a token issuer's blacklist (e.g. Tether) — token frozen by issuer |

**`TOKEN_BLACKLIST_ADDRESS_HIT` `eventDetail` fields** (the JS SDK has no type for it — read the parsed JSON):

| Field | Type | Description |
|-------|------|-------------|
| `blockChain` | string | Blockchain |
| `contract` | string | Contract address |
| `address` | string | Address |
| `workspaceKey` | string | Workspace Key, the only workspace identifier |
| `workspaceKeyName` | string | Workspace Name |
| `accountKey` | string | Account Key, the only account identifier |
| `accountName` | string | Account name |
| `coinKey` | string | Coin key |
| `issuer` | string | Issuer |
| `lastTxHash` | string | Transaction hash |
| `lastUpdateTime` | number | Last update time (UNIX millisecond timestamp) |

---

## Re-push API

Two methods for recovering missed webhook events:

### `resendWebhook` — Re-push latest event for one transaction

Re-pushes only the **most recent** webhook event for a given transaction (not full history).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | string | Yes | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `txKey` | string | Yes | Transaction key |

```typescript
import { WebhookApi } from '@safeheron/api-sdk';

const webhookApi = new WebhookApi(config);

await webhookApi.resendWebhook({
  category: 'TRANSACTION',
  txKey: 'your-tx-key',
});
```

### `resendFailed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes. Only webhooks from the **past 7 days** can be resent — requests with timestamps older than 7 days will silently return empty results.

> **Supported events:** `TRANSACTION_CREATED`, `TRANSACTION_STATUS_CHANGED`, `MPC_SIGN_CREATED`, `MPC_SIGN_STATUS_CHANGED`, `WEB3_SIGN_CREATED`, `WEB3_SIGN_STATUS_CHANGED`, `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, `WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`, `GAS_BALANCE_WARNING`, `AML_KYT_ALERT` (more types may be added later). `TOKEN_BLACKLIST_ADDRESS_HIT` **cannot be re-pushed** — persist every event durably before acking.

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

Every webhook payload **must be signature-verified** before any business logic runs. The SDK's `WebHookConverter.convertWebHook()` handles this automatically -- it throws `Error` on invalid signatures.

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

Safeheron may deliver the same event multiple times (failed deliveries are retried 6 times: 30s, 1m, 5m, 1h, 12h, 24h). Always check if you've already processed a given `txKey` + `transactionStatus` combination before acting, and commit that check together with the business side-effects in **one DB transaction** (see [Step 3](#step-3----process-the-inbox-separate-worker)).

### 5. Anti-Dust Attack Filtering for Deposits

```typescript
const minDeposit = getMinimumDepositAmount(coinKey);
const txAmount = new Decimal(event.txAmount);  // use Decimal to avoid float precision loss

if (txAmount.lessThan(minDeposit)) {
  console.log(`Ignoring dust deposit: ${event.txAmount} ${coinKey} (below minimum ${minDeposit})`);
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
  case 'TOKEN_BLACKLIST_ADDRESS_HIT':
    alertOpsTeam('Address frozen by token issuer', event);
    break;
}
```

---

## Best Practices

- Ack fast, but only after the verified event is **durably** stored (DB inbox or durable MQ) — never ack after only handing it to an in-memory queue or a fire-and-forget async task.
- Commit idempotency markers and business side-effects in the same DB transaction.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- Schedule a periodic call to `oneTransactions()` as a safety net.
- Auto Sweep, Gas refill, speed-up, and batch transfers all generate standard webhook events.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
