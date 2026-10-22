# Webhook Integration Reference

## Overview

Safeheron pushes real-time event notifications to your configured webhook URL when transactions, MPC sign requests, Web3 sign requests, whitelist changes, or security events occur.

Configure the webhook URL in: **Safeheron Web Console → Settings → API → Webhook**

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
2. Verify `sig` using **Safeheron's RSA public key** — reject if invalid.
3. Decrypt `key` field using **your RSA private key** → 48 bytes (AES key + IV).
4. Decrypt `bizContent` using AES/GCM/NoPadding → plaintext JSON event payload.

---

## Webhook Handler Example (Spring Boot)

### Step 1 — Register WebhookConverter as a Bean

```java
@Configuration
public class SafeheronWebhookConfig {

    @Bean
    public WebhookConverter webhookConverter() {
        return new WebhookConverter("${SAFEHERON_WEBHOOK_RSA_PUBLIC_KEY}", "${WEBHOOK_RSA_PRIVATE_KEY}");//todo Replace with the WebHook RSA private key you read from Vault/KMS and replace with the Safeheron WebHook RSA public key from Safeheron Console
    }
}
```

### Step 2 — Webhook Controller (durable inbox, then ack)

> Ack **only after the verified event is durably stored** (a DB inbox table with a unique key, or a durable MQ).
> In-memory queues and `@Async` hand-offs lose events on a crash, and `TOKEN_BLACKLIST_ADDRESS_HIT`
> **cannot be re-pushed** by `resendFailed` (see "Re-push API" below).
> `WebhookInbox` is a placeholder for your own repository: `saveIfAbsent` inserts a row keyed by `eventKey`
> (a duplicate delivery hits the unique key and is simply acked again).

```java
import com.safeheron.client.exception.SafeheronException;
import com.safeheron.client.webhook.WebHook;
import com.safeheron.client.webhook.WebHookBizContent;
import com.safeheron.client.webhook.WebHookResponse;
import com.safeheron.client.webhook.WebhookConverter;
import org.springframework.http.ResponseEntity;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.HexFormat;

@Slf4j
@RestController
public class WebhookController {

    @Resource
    private ObjectMapper objectMapper;

    @Resource
    private WebhookConverter webhookConverter;

    @Resource
    private WebhookInbox webhookInbox;   // your DB inbox repository (placeholder)

    @PostMapping("/safeheron/webhook")
    public ResponseEntity<WebHookResponse> handleWebhook(@RequestBody String rawBody) {
        try {
            WebHook param = objectMapper.readValue(rawBody, WebHook.class);

            // convert() handles everything internally:
            //   1. Verifies RSA signature — throws SafeheronException if invalid
            //   2. Decrypts AES key using your RSA private key
            //   3. Decrypts bizContent
            //   4. Deserializes eventDetail to a concrete type ONLY for TRANSACTION_*, MPC_SIGN_* and WEB3_SIGN_* events.
            //      For other events (WHITELIST_*, AML_KYT_ALERT, GAS_BALANCE_WARNING, ...) eventDetail stays a raw Map
            WebHookBizContent content = webhookConverter.convert(param);

            // Durably store the verified event BEFORE acking; processing happens in WebhookInboxWorker
            webhookInbox.saveIfAbsent(eventKey(content), content.getEventType(), objectMapper.writeValueAsString(content));
        } catch (SafeheronException e) {
            // Signature verification failed — never ack; the event is not trusted
            log.error("Webhook signature verification FAILED — possible tampering", e);
            return ResponseEntity.status(401).build();
        } catch (Exception e) {
            // Internal error (decrypt / parse / store) — a non-SUCCESS response makes Safeheron retry
            // (30s, 1m, 5m, 1h, 12h, 24h); after that, recover via re-push or REST polling
            log.error("Webhook processing error", e);
            return ResponseEntity.status(500).build();
        }

        // Body per docs: {"code":"200","message":"SUCCESS"}; anything else is treated as a failed delivery and retried
        WebHookResponse response = new WebHookResponse();
        response.setCode("200");
        response.setMessage("SUCCESS");
        return ResponseEntity.ok(response);
    }

    // The same event re-delivered has the same eventType + eventDetail, hence the same key
    private String eventKey(WebHookBizContent content) throws Exception {
        String material = content.getEventType() + objectMapper.writeValueAsString(content.getEventDetail());
        byte[] hash = MessageDigest.getInstance("SHA-256").digest(material.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hash);
    }
}
```

### Step 3 — Inbox Worker (business processing)

Run it from a scheduler or MQ consumer in a **separate bean** (so Spring's `@Transactional` proxy applies).
Business side effects and marking the inbox row `DONE` are committed in **one** DB transaction; if processing
throws, the row stays `PENDING` and is retried. Never "mark processed and commit, then do the work".

```java
import com.safeheron.client.webhook.MPCSignParam;
import com.safeheron.client.webhook.TransactionParam;
import com.safeheron.client.webhook.WebHookBizContent;
import com.safeheron.client.webhook.Web3SignParam;
import com.fasterxml.jackson.core.type.TypeReference;

import java.util.Map;

@Slf4j
@Component
public class WebhookInboxWorker {

    @Resource
    private ObjectMapper objectMapper;

    @Resource
    private WebhookInbox webhookInbox;

    @Transactional
    public void process(WebhookInboxRow row) throws Exception {
        WebHookBizContent content = objectMapper.readValue(row.getPayload(), WebHookBizContent.class);
        // After the JSON round-trip eventDetail is a Map — convert it to the typed class
        switch (content.getEventType()) {
            case "TRANSACTION_CREATED":
            case "TRANSACTION_CUSTOMIZED_CONFIRMING":
            case "TRANSACTION_STATUS_CHANGED":
                TransactionParam tx = objectMapper.convertValue(content.getEventDetail(), TransactionParam.class);
                // handle transaction (UTXO coins: read tx.getDestinationAddressList(), not only destinationAddress)
                break;
            case "MPC_SIGN_CREATED":
            case "MPC_SIGN_STATUS_CHANGED":
                MPCSignParam mpc = objectMapper.convertValue(content.getEventDetail(), MPCSignParam.class);
                // handle MPC sign
                break;
            case "WEB3_SIGN_CREATED":
            case "WEB3_SIGN_STATUS_CHANGED":
                Web3SignParam web3 = objectMapper.convertValue(content.getEventDetail(), Web3SignParam.class);
                // handle Web3 sign
                break;
            case "TOKEN_BLACKLIST_ADDRESS_HIT":
                // SDK 1.0.15 has no typed class for this event — keep it as a Map
                Map<String, Object> frozen = objectMapper.convertValue(
                        content.getEventDetail(), new TypeReference<Map<String, Object>>() {});
                // e.g. alert ops and stop using frozen.get("address") for frozen.get("coinKey")
                break;
            default:
                log.info("Unhandled webhook event type {}", content.getEventType());
        }
        webhookInbox.markDone(row.getId());   // same transaction as the business writes above
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

**Transaction Event Payload (decrypted bizContent) Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `txHash` | String | On-chain transaction hash |
| `transactionStatus` | String | Current status |
| `transactionSubStatus` | String | Sub-status |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Amount (string) |
| `sourceAccountKey` | String | Sender wallet |
| `sourceAddress` | String | Sender address |
| `destinationAddress` | String | Recipient address |
| `txFee` | String | Transaction fee paid |
| `blockHeight` | Long | Confirmed block height |
| `createTime` | Long | Unix timestamp (ms) |
| `completedTime` | Long | Unix timestamp (ms) of completion |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |

---

### MPC Sign Events

| Event Type | Trigger |
|------------|---------|
| `MPC_SIGN_CREATED` | New MPC sign request created |
| `MPC_SIGN_STATUS_CHANGED` | Status transition |

**MPC Sign Event Payload Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | MPC sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Wallet used for signing |
| `signAlg` | String | `Secp256k1` or `Ed25519` |
| `dataList` | Array | List of `{data, sig}` items (sig available on SUCCESS) |

---

### Web3 Sign Events

| Event Type | Trigger |
|------------|---------|
| `WEB3_SIGN_CREATED` | New Web3 sign request |
| `WEB3_SIGN_STATUS_CHANGED` | Status transition |

**Web3 Sign Event Payload Key Fields:**

| Field | Type   | Description |
|-------|--------|-------------|
| `txKey` | String | Web3 sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGN_TYPED_DATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | String | Web3 wallet key |
| `sourceAddress` | String | Signing address |
| `message` / `messageHash` / `transaction` | Object | Signed content (type-dependent) |

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
| `TOKEN_BLACKLIST_ADDRESS_HIT` | An address in your team was added to a token issuer's blacklist (e.g. Tether) — verify and handle the affected assets |

### TOKEN_BLACKLIST_ADDRESS_HIT eventDetail (TokenBlackListAddressParam)

> Not modelled in SDK 1.0.15 — map `eventDetail` yourself (e.g. `objectMapper.convertValue(...)` into your own class or a `Map`).

| Parameters | Type | Description |
|------------|------|-------------|
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
| `lastUpdateTime` | int64 | Last update time (UNIX millisecond timestamp) |

---

## Re-push API

Two methods for recovering missed webhook events:

### `resendWebhook` — Re-push latest event for one transaction

Re-pushes only the **most recent** webhook event for a given transaction (not full history).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | String | Yes | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `txKey` | String | Yes | Transaction key |

```java
WebhookApiService webhookApi = ServiceCreator.create(WebhookApiService.class, config);

ResendWebhookRequest resendReq = new ResendWebhookRequest();
resendReq.setCategory("TRANSACTION");
resendReq.setTxKey("your-tx-key");
ServiceExecutor.execute(webhookApi.resendWebhook(resendReq));
```

### `resendFailed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes. Only webhooks from the **past 7 days** can be resent — requests with timestamps older than 7 days will silently return empty results.

> **Warning:** Your handler must be idempotent. `resendFailed` may re-deliver intermediate-status events (e.g. `CONFIRMING`) even if you already received a terminal status (`COMPLETED`). Never roll back a terminal state.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `startTime` | Long (ms) | Yes | Start of time range |
| `endTime` | Long (ms) | Yes | End of time range (max interval: 1 hour) |

```java
ResendFailedRequest failedReq = new ResendFailedRequest();
failedReq.setStartTime(System.currentTimeMillis() - 3600000L);
failedReq.setEndTime(System.currentTimeMillis());
MessagesCountResponse result = ServiceExecutor.execute(webhookApi.resendFailed(failedReq));
```

> `resendFailed` supports these event types (per the published docs): `TRANSACTION_CREATED`, `TRANSACTION_STATUS_CHANGED`, `MPC_SIGN_CREATED`, `MPC_SIGN_STATUS_CHANGED`, `WEB3_SIGN_CREATED`, `WEB3_SIGN_STATUS_CHANGED`, `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, `WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`, `GAS_BALANCE_WARNING`, `AML_KYT_ALERT`. `TOKEN_BLACKLIST_ADDRESS_HIT` is **not** supported and cannot be re-pushed — store every event durably before acking.

Safeheron automatic retry schedule on a non-200 / malformed response: failed deliveries are retried 6 times at
`30s → 1m → 5m → 1h → 12h → 24h`, then stop.

---

## Custom Block Confirmation Count

Configure custom confirmation thresholds per blockchain in Safeheron Console. Once enabled, it applies to **both incoming and outgoing** transactions on that chain. A `TRANSACTION_CUSTOMIZED_CONFIRMING` event fires when the threshold is reached.

---

## Webhook Filtering

- **No amount filtering** — any successful deposit, regardless of amount, triggers a webhook (including dust attacks / small transfers).
- All transaction types trigger webhooks: ordinary transfers, Auto Sweep (归集), gas refill (加油), speed-up, batch transfers.

---

## Configuration Change Behavior

- **Modifying** a webhook URL: events immediately route to the new URL. No approval required. Takes effect instantly.
- **Deleting** a webhook URL: webhook delivery stops completely.
- Adding/modifying/deleting Webhook URLs and RSA public keys **require no admin approval** — changes are immediate.
- Manual webhook re-trigger in Web Console requires **"Initiate Transaction" permission** and is rate-limited to **10 times/minute**.

---

## Security Requirements for Webhook Endpoints

### 1. Verify Signature Before Processing (Mandatory)

Every webhook payload **must be signature-verified** before any business logic runs. Never trust the decrypted content if the signature check fails.

```java
@PostMapping("/safeheron/webhook")
public ResponseEntity<WebHookResponse> handleWebhook(@RequestBody String rawBody,
                                                      HttpServletRequest httpReq) {
    // Step 0: IP whitelist check — only accept from Safeheron egress IPs
    String clientIp = httpReq.getRemoteAddr();
    if (!SAFEHERON_EGRESS_IPS.contains(clientIp)) {
        log.warn("Rejected webhook from unknown IP: {}", clientIp);
        return ResponseEntity.status(403).build();
    }

    try {
        WebHook param = mapper.readValue(rawBody, WebHook.class);

        // convert() atomically verifies RSA signature + decrypts bizContent.
        // Throws SafeheronException if signature verification fails.
        WebHookBizContent resp = webhookConverter.convert(param);

        // Signature verified — store durably (DB inbox with a unique key, or a durable MQ) BEFORE acking.
        // In-memory queues lose events on a crash; TOKEN_BLACKLIST_ADDRESS_HIT cannot be re-pushed by resendFailed.
        webhookInbox.saveIfAbsent(eventKey(resp), resp.getEventType(), mapper.writeValueAsString(resp));  // eventKey(): see Step 2

    } catch (SafeheronException e) {
        // Signature verification failed — never ack; the event is not trusted
        log.error("Webhook signature verification FAILED — possible tampering", e);
        return ResponseEntity.status(401).build();
    } catch (Exception e) {
        // Internal error (decrypt / parse / store) — a non-SUCCESS response makes Safeheron retry
        // (30s, 1m, 5m, 1h, 12h, 24h); after that, recover via re-push or REST polling
        log.error("Webhook error", e);
        return ResponseEntity.status(500).build();
    }

    // Ack ONLY after the event is verified and durably stored.
    // Body per docs: {"code":"200","message":"SUCCESS"}; anything else is treated as a failed delivery and retried
    WebHookResponse ack = new WebHookResponse();
    ack.setCode("200");
    ack.setMessage("SUCCESS");
    return ResponseEntity.ok(ack);
}
```

### 2. IP Whitelist — Only Accept Safeheron's Egress IPs

Configure your webhook server to **only accept connections from** (not listed in the published API reference — confirm the current list with Safeheron Support):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

Reject all other sources at the network/firewall level, not just in application code.

### 3. Avoid Status Rollback

Webhook events may arrive out of order due to Safeheron's async retry mechanism. Your handler **must never downgrade a status**:

```java
private boolean shouldUpdateStatus(String currentStatus, String newStatus) {
    // Terminal statuses are final — never overwrite them
    Set<String> terminalStatuses = Set.of("COMPLETED", "SIGN_COMPLETED", "FAILED", "REJECTED", "CANCELLED");
    if (terminalStatuses.contains(currentStatus)) {
        return false;  // already in terminal state — ignore late events
    }
    return true;
}
```

**Example:** If your DB shows `COMPLETED` and you receive a late `CONFIRMING` event — keep `COMPLETED`.

### 4. Idempotency by `txKey`

Safeheron may deliver the same event multiple times (retry on non-200). Always check if you've already processed a given `txKey` + `transactionStatus` combination before acting, and commit the "processed" marker in the **same DB transaction** as the business side effects (see Step 3) — never insert a dedup row and commit before doing the work, or a failure after that commit loses the event for good.

### 5. Anti-Dust Attack Filtering for Deposits

External actors can send tiny amounts to your users' deposit addresses to pollute your webhook stream. Protect against this:

```java
// Filter dust/address-pollution transactions
BigDecimal minDeposit = getMinimumDepositAmount(coinKey);
BigDecimal txAmount   = new BigDecimal(event.get("txAmount").asText());

if (txAmount.compareTo(minDeposit) < 0) {
    log.info("Ignoring dust deposit: {} {} (below minimum {})", txAmount, coinKey, minDeposit);
    return;
}
```

### 6. Subscribe to Security Events

Always handle these non-transaction events:

```java
switch (eventType) {
    case "ILLEGAL_IP_REQUEST":
        // API called from non-whitelisted IP — investigate immediately
        alertSecurityTeam("Safeheron API called from illegal IP", event);
        break;
    case "NO_MATCHING_TRANSACTION_POLICY":
        // Transaction has no matching policy — possible misconfiguration or attack
        alertOpsTeam("Transaction blocked: no matching policy", event);
        break;
    case "GAS_BALANCE_WARNING":
        // Gas station balance low — may block future transactions
        alertOpsTeam("Gas balance warning", event);
        break;
    case "AML_KYT_ALERT":
        // KYT alert notification — transactions may be marked by aml
        alertOpsTeam("KYT alert notification", event);
        break;
    case "TOKEN_BLACKLIST_ADDRESS_HIT":
        // An address in your team was added to a token issuer's blacklist (e.g. Tether) — verify and handle the affected assets
        alertOpsTeam("Address frozen by token issuer", event);
        break;
}
```

---

## Best Practices

- Return HTTP 200 as fast as possible — but only after the verified event is **durably** stored (DB inbox or durable MQ); never ack after handing off to an in-memory queue or `@Async`.
- Store the raw encrypted payload in a database before processing, for auditability.
- Deposits on UTXO coins: credit from `destinationAddressList[]` (address / amount), not only `destinationAddress`.
- `transactionDirection = INTERNAL_TRANSFER` also covers withdrawals to another address inside your Safeheron team — handle it in both the withdrawal and the deposit flows.
- Implement idempotency by checking `txKey` before acting — Safeheron may deliver duplicates on retry.
- Schedule a periodic call to `/v1/transactions/one` as a safety net.
- Auto Sweep (归集), Gas refill, speed-up, and batch transfers all generate standard webhook events.
- Verify the webhook signature on **every single request** — never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
- Poll the transaction list API periodically as a fallback to catch events missed due to webhook downtime.
