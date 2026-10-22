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

## Webhook Handler Example (Go SDK)

### Using the SDK's WebhookConverter

```go
import (
    "log"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/webhook"
)

webhookConverter := webhook.WebhookConverter{Config: webhook.WebHookConfig{
    SafeheronWebHookRsaPublicKey: "${SAFEHERON_WEBHOOK_RSA_PUBLIC_KEY_PATH}",
    WebHookRsaPrivateKey:         "${WEBHOOK_RSA_PRIVATE_PATH}",
}}

// Decrypt and verify webhook payload
webHookBizContent, err := webhookConverter.Convert(webHookPayload)
if err != nil {
    log.Printf("Webhook verification failed: %v", err)
    return
}

// Return success response
var response webhook.WebHookResponse
response.Code = "200"
response.Message = "SUCCESS"
```

### Full net/http Handler

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "time"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/webhook"
)

var webhookConverter webhook.WebhookConverter

func init() {
    webhookConverter = webhook.WebhookConverter{Config: webhook.WebHookConfig{
		SafeheronWebHookRsaPublicKey: "${SAFEHERON_WEBHOOK_RSA_PUBLIC_KEY_PATH}",
		WebHookRsaPrivateKey:         "${WEBHOOK_RSA_PRIVATE_PATH}",
    }}
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    // Read the raw request body
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    // Parse the webhook payload
    var webHook webhook.WebHook
    if err := json.Unmarshal(body, &webHook); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Decrypt and verify signature
    bizContent, err := webhookConverter.Convert(webHook)
    if err != nil {
        // The SDK returns plain errors; "webhook signature verification failed" means the event is not trusted
        if err.Error() == "webhook signature verification failed" {
            log.Printf("Webhook signature verification FAILED -- possible tampering: %v", err)
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        // Decrypt error (check your webhook RSA private key) -- a non-SUCCESS response makes Safeheron retry
        log.Printf("Webhook decrypt error: %v", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    // Persist the verified event DURABLY (DB inbox table or durable MQ) before acking.
    // In-memory channels / fire-and-forget goroutines lose events acked just before a crash, and
    // TOKEN_BLACKLIST_ADDRESS_HIT cannot be re-pushed by ResendFailed.
    if err := eventInbox.Save(bizContent); err != nil {
        log.Printf("Webhook inbox write failed -- asking Safeheron to retry: %v", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    // Ack ONLY after the event is verified and durably stored.
    // Body per docs: {"code":"200","message":"SUCCESS"}; anything else is treated as a failed delivery and retried
    // (6 times: 30s, 1m, 5m, 1h, 12h, 24h).
    resp := webhook.WebHookResponse{Code: "200", Message: "SUCCESS"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

// EventInbox is your durable store (placeholder — implement with your DB / MQ).
//   Save:    insert the raw bizContent; dedupe on a unique key (e.g. sha256 of bizContent) so duplicates are ignored.
//   Pending: return events not yet processed.
//   Process: run the business side-effects AND mark the event done in the SAME DB transaction;
//            on error roll back so the event stays pending and is retried. Never "mark done, then process".
type EventInbox interface {
    Save(bizContent string) error
    Pending() ([]InboxEvent, error)
    Process(ev InboxEvent, handle func(bizContent string) error) error
}

type InboxEvent struct {
    ID         int64
    BizContent string
}

var eventInbox EventInbox // inject your implementation at startup

// Background worker: drains the inbox. A failing / panicking event stays pending and is retried later.
func eventWorker() {
    for {
        events, err := eventInbox.Pending()
        if err != nil {
            log.Printf("ERROR: read webhook inbox: %v", err)
        }
        for _, ev := range events {
            if err := eventInbox.Process(ev, processSafely); err != nil {
                log.Printf("ERROR: webhook event %d not processed, will retry: %v", ev.ID, err)
            }
        }
        time.Sleep(time.Second)
    }
}

// A panic while handling one event must not kill the worker; it is turned into an error (event stays pending)
func processSafely(bizContent string) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("webhook event processing panicked: %v", r)
        }
    }()
    return processWebhookEvent(bizContent)
}

func processWebhookEvent(bizContent string) error {
    // Parse the decrypted bizContent JSON
    var event map[string]interface{}
    if err := json.Unmarshal([]byte(bizContent), &event); err != nil {
        return fmt.Errorf("parse bizContent: %w", err)
    }

    eventType, _ := event["eventType"].(string)
    switch eventType {
    case "TRANSACTION_CREATED", "TRANSACTION_STATUS_CHANGED", "TRANSACTION_CUSTOMIZED_CONFIRMING":
        // Handle transaction event
        log.Printf("Transaction event: %s", eventType)
    case "MPC_SIGN_CREATED", "MPC_SIGN_STATUS_CHANGED":
        // Handle MPC sign event
        log.Printf("MPC sign event: %s", eventType)
    case "WEB3_SIGN_CREATED", "WEB3_SIGN_STATUS_CHANGED":
        // Handle Web3 sign event
        log.Printf("Web3 sign event: %s", eventType)
    case "ILLEGAL_IP_REQUEST":
        log.Printf("ALERT: API called from non-whitelisted IP")
    case "NO_MATCHING_TRANSACTION_POLICY":
        log.Printf("ALERT: Transaction blocked -- no matching policy")
    case "GAS_BALANCE_WARNING":
        log.Printf("ALERT: Gas balance warning")
    case "AML_KYT_ALERT":
        log.Printf("ALERT: KYT alert notification")
    case "TOKEN_BLACKLIST_ADDRESS_HIT":
        // An address in your team was frozen by a token issuer (e.g. Tether) — the Go SDK has no
        // typed struct for this event, so decode eventDetail yourself
        detailJSON, _ := json.Marshal(event["eventDetail"])
        var hit TokenBlackListAddressParam
        if err := json.Unmarshal(detailJSON, &hit); err != nil {
            return fmt.Errorf("decode TOKEN_BLACKLIST_ADDRESS_HIT: %w", err)
        }
        log.Printf("ALERT: %s frozen by %s on %s (coin %s, account %s)",
            hit.Address, hit.Issuer, hit.BlockChain, hit.CoinKey, hit.AccountKey)
    case "WHITELIST_ADDED", "WHITELIST_UPDATED", "WHITELIST_REMOVED":
        log.Printf("Whitelist event: %s", eventType)
    }
    return nil
}

// eventDetail of TOKEN_BLACKLIST_ADDRESS_HIT (field names per Safeheron docs)
type TokenBlackListAddressParam struct {
    BlockChain       string `json:"blockChain"`
    Contract         string `json:"contract"`
    Address          string `json:"address"`
    WorkspaceKey     string `json:"workspaceKey"`
    WorkspaceKeyName string `json:"workspaceKeyName"`
    AccountKey       string `json:"accountKey"`
    AccountName      string `json:"accountName"`
    CoinKey          string `json:"coinKey"`
    Issuer           string `json:"issuer"`
    LastTxHash       string `json:"lastTxHash"`
    LastUpdateTime   int64  `json:"lastUpdateTime"`
}

func main() {
    go eventWorker()
    http.HandleFunc("/safeheron/webhook", webhookHandler)
    fmt.Println("Webhook server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
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
| `txKey` | `string` | Safeheron transaction key |
| `customerRefId` | `string` | Your reference ID |
| `txHash` | `string` | On-chain transaction hash |
| `transactionStatus` | `string` | Current status |
| `transactionSubStatus` | `string` | Sub-status |
| `coinKey` | `string` | Coin identifier |
| `txAmount` | `string` | Amount (string) |
| `sourceAccountKey` | `string` | Sender wallet |
| `sourceAddress` | `string` | Sender address |
| `destinationAddress` | `string` | Recipient address (for UTXO coins use `destinationAddressList`) |
| `destinationAddressList` | `[]object` | UTXO coins: destinations as `{address, amount, memo}` — credit each output that belongs to you |
| `transactionDirection` | `string` | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER` |
| `txFee` | `string` | Transaction fee paid |
| `blockHeight` | `int64` | Confirmed block height |
| `createTime` | `int64` | Unix timestamp (ms) |
| `completedTime` | `int64` | Unix timestamp (ms) of completion |
| `customerExt1` | `string` | Custom field 1 |
| `customerExt2` | `string` | Custom field 2 |
| `amlLock` | `string` | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |

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
| `TOKEN_BLACKLIST_ADDRESS_HIT` | An address was added to a token issuer's blacklist (e.g. Tether) — verify and handle the affected assets |

### TOKEN_BLACKLIST_ADDRESS_HIT — `eventDetail` (TokenBlackListAddressParam)

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

### `ResendWebhook` — Re-push latest event for one transaction

Re-pushes only the **most recent** webhook event for a given transaction (not full history).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `Category` | string | Yes | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `TxKey` | string | Yes | Transaction key |

```go
webhookApi := api.WebhookApi{Client: sc}

resendReq := api.ResendWebhookRequest{
    Category: "TRANSACTION",
    TxKey:    "your-tx-key",
}
var resendResp api.ResultResponse
if err := webhookApi.ResendWebhook(resendReq, &resendResp); err != nil {
    log.Printf("ResendWebhook failed: %v", err)
}
```

### `ResendFailed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes. Only webhooks from the **past 7 days** can be resent — requests with timestamps older than 7 days will silently return empty results.

> **Supported event types:** `TRANSACTION_CREATED`, `TRANSACTION_STATUS_CHANGED`, `MPC_SIGN_CREATED`, `MPC_SIGN_STATUS_CHANGED`, `WEB3_SIGN_CREATED`, `WEB3_SIGN_STATUS_CHANGED`, `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, `WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`, `GAS_BALANCE_WARNING`, `AML_KYT_ALERT` (more types may be added later). `TOKEN_BLACKLIST_ADDRESS_HIT` is **not** supported and cannot be re-pushed — store every event durably before acking.

> **Warning:** Your handler must be idempotent. `ResendFailed` may re-deliver intermediate-status events (e.g. `CONFIRMING`) even if you already received a terminal status (`COMPLETED`). Never roll back a terminal state.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `StartTime` | int64 (ms) | Yes | Start of time range |
| `EndTime` | int64 (ms) | Yes | End of time range (max interval: 1 hour) |

```go
now := time.Now().UnixMilli()
failedReq := api.ResendFailedRequest{
    StartTime: now - 3600000,
    EndTime:   now,
}
var failedResp api.MessagesCountResponse
if err := webhookApi.ResendFailed(failedReq, &failedResp); err != nil {
    log.Printf("ResendFailed failed: %v", err)
}
// failedResp.MessagesCount: number of events triggered
```

Safeheron automatic retry schedule on non-200 response:
`30s -> 1m -> 5m -> 1h -> 12h -> 24h` (7 total attempts, then stops)

---

## Security Requirements for Webhook Endpoints

### 1. Verify Signature Before Processing (Mandatory)

The SDK's `WebhookConverter.Convert()` handles signature verification automatically. Never process the decrypted content if verification fails.

### 2. IP Whitelist -- Only Accept Safeheron's Egress IPs

Configure your webhook server to **only accept connections from**:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

### 3. Avoid Status Rollback

```go
func shouldUpdateStatus(currentStatus, newStatus string) bool {
    terminalStatuses := map[string]bool{
        "COMPLETED": true, "SIGN_COMPLETED": true, "FAILED": true,
        "REJECTED": true, "CANCELLED": true,
    }
    if terminalStatuses[currentStatus] {
        return false // already in terminal state -- ignore late events
    }
    return true
}
```

### 4. Idempotency by txKey

Safeheron may deliver the same event multiple times. Always check if you have already processed a given `txKey` + `transactionStatus` combination before acting. Write the "processed" marker and the business side-effects (credit, status update) in the **same DB transaction** — never commit the marker first and process afterwards, or a failure after the commit makes the retried event look like a duplicate and it is lost.

### 5. Anti-Dust Attack Filtering

```go
import (
    "log"

    "github.com/shopspring/decimal"
)

minDeposit := getMinimumDepositAmount(coinKey)
txAmount, _ := decimal.NewFromString(event["txAmount"].(string))
if txAmount.LessThan(minDeposit) {
    log.Printf("Ignoring dust deposit: %s %s (below minimum %s)",
        txAmount, coinKey, minDeposit)
    return
}
```

---

## Best Practices

- Return HTTP 200 as fast as possible -- but only after the verified event is **durably** stored (DB inbox / durable MQ); process it in a background worker. An in-memory channel or fire-and-forget goroutine loses events on crash.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- **UTXO deposits:** read `destinationAddressList[]` (`address`, `amount`) -- `destinationAddress` / `txAmount` alone miss multi-output and UTXO deposits.
- **`transactionDirection = INTERNAL_TRANSFER`:** a withdrawal to another address in your own team is reported as an internal transfer, not OUTFLOW -- it must still settle the withdrawal order (and credit the receiving side if that address belongs to a user).
- Schedule a periodic call to `/v1/transactions/one` as a safety net.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
