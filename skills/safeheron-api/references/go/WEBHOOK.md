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

## Webhook Handler Example (Go SDK)

### Using the SDK's WebhookConverter

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/webhook"
)

webhookConverter := webhook.WebhookConverter{Config: webhook.WebHookConfig{
    SafeheronWebHookRsaPublicKey: "pems/safeheron_webhook_public.pem",
    WebHookRsaPrivateKey:         "pems/webhook_private.pem",
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

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/webhook"
)

var webhookConverter webhook.WebhookConverter

func init() {
    webhookConverter = webhook.WebhookConverter{Config: webhook.WebHookConfig{
        SafeheronWebHookRsaPublicKey: "pems/safeheron_webhook_public.pem",
        WebHookRsaPrivateKey:         "pems/webhook_private.pem",
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
        log.Printf("Webhook verification failed: %v", err)
        // Still return 200 to avoid retry storms
    } else {
        // Process the event asynchronously
        go processWebhookEvent(bizContent)
    }

    // Always respond 200 immediately -- never block
    resp := webhook.WebHookResponse{Code: "200", Message: "SUCCESS"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func processWebhookEvent(bizContent string) {
    // Parse the decrypted bizContent JSON
    var event map[string]interface{}
    json.Unmarshal([]byte(bizContent), &event)

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
    }
}

func main() {
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
| `destinationAddress` | `string` | Recipient address |
| `createTime` | `int64` | Unix timestamp (ms) |

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
err := webhookApi.ResendWebhook(resendReq, &resendResp)
```

### `ResendFailed` — Re-push all failed events in a time range

Re-pushes every failed webhook event within a time window (max 1 hour). Rate-limited to once every 10 minutes.

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
err = webhookApi.ResendFailed(failedReq, &failedResp)
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

Safeheron may deliver the same event multiple times. Always check if you have already processed a given `txKey` + `transactionStatus` combination before acting.

### 5. Anti-Dust Attack Filtering

```go
import "github.com/shopspring/decimal"

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

- Return HTTP 200 as fast as possible -- offload all processing to a goroutine or channel-based worker.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting -- Safeheron may deliver duplicates on retry.
- Schedule a periodic call to `/v1/transactions/one` as a safety net.
- Verify the webhook signature on **every single request** -- never skip this step in production.
- Apply IP whitelist at the firewall level, not just application code.
