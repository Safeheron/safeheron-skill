# Security Best Practices

> **These rules are mandatory.** When this Skill is active, every piece of generated code and every architecture recommendation must comply without exception.

---

## 1. Credential & Key Management

### Rules for Generated Code

- **Private keys must never be stored in plaintext** -- not in config files committed to version control, environment variables with hardcoded values, or source code.
- PEM key files must be stored outside the project directory with restricted permissions (`chmod 600`).

| Deployment Target | Required Solution |
|---|---|
| Cloud (AWS) | AWS KMS |
| Cloud (GCP) | GCP KMS |
| Self-hosted | HashiCorp Vault |
| Local development only | Read from a file **outside the project directory**; never committed to git |

### Code Pattern -- Loading Credentials

```go
// CORRECT: Load from environment variable (dev/test)
sc := safeheron.Client{Config: safeheron.ApiConfig{
    BaseUrl:               os.Getenv("SAFEHERON_BASE_URL"),
    ApiKey:                os.Getenv("SAFEHERON_API_KEY"),
    RsaPrivateKey:         os.Getenv("SAFEHERON_PRIVATE_KEY_PATH"),
    SafeheronRsaPublicKey: os.Getenv("SAFEHERON_PUBLIC_KEY_PATH"),
    RequestTimeout:        20000,
}}

// CORRECT: PEM file path from secrets manager (production)
// privateKeyPath := awsSecretsManager.GetSecret("safeheron/private-key-path")

// WRONG -- never do this:
// ApiKey: "sk-live-xxxxxxxxxxxx",
// RsaPrivateKey: "/hardcoded/path/in/source/code.pem",
```

---

## 2. Transfer & Whitelist Security

### Rules for Generated Code

**2-1. `customerRefId` must be a business-unique ID for idempotency.**
Generate it and persist it to your database **before** calling the Safeheron API. On timeout, retry with the same ID.

```go
customerRefId := uuid.New().String()
saveOrderToDB(customerRefId, amount, toAddress) // persist FIRST
// then call Safeheron API
```

**2-2. Whitelist addresses are required for formal transfers.**
`ONE_TIME_ADDRESS` must only be used for genuinely temporary, one-off payment scenarios.

**2-3. AML check is mandatory before every transfer.**

```go
// Required before creating any outbound transaction
submitReq := api.AmlCheckerRequestRequest{
    Network: "Ethereum",
    Address: destinationAddress,
}
var submitResp api.AmlCheckerRequestResponse
if err := toolsApi.AmlCheckerRequest(submitReq, &submitResp); err != nil {
    panic(err)
}
// ... poll and check risk level
```

**2-4. Validate address format before whitelist add or transfer.**

```go
checkReq := api.CheckCoinAddressRequest{
    CoinKey: "ETHEREUM_ETH",
    Address: destinationAddress,
}
var checkResp api.CheckCoinAddressResponse
if err := coinApi.CheckCoinAddress(checkReq, &checkResp); err != nil {
    panic(err)
}
if !checkResp.AddressValid {
    panic("invalid address format")
}
```

**2-5. Amounts must use `string` in API requests and `decimal.Decimal` in application logic.**
Never use `float32` or `float64` for monetary values.

```go
import "github.com/shopspring/decimal"

// CORRECT
amount := decimal.NewFromString("0.001")
req.TxAmount = amount.String()

// WRONG -- never use float for amounts
// amount := 0.001
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

```go
// REQUIRED pattern for Co-Signer approval callback
func evaluateTransaction(content map[string]interface{}) string {
    customerRefId, _ := content["customerRefId"].(string)

    // 1. Look up the business order
    order := findOrderByCustomerRefId(customerRefId)
    if order == nil {
        return "REJECT" // unknown order
    }

    // 2. Validate amount matches
    txAmount, _ := decimal.NewFromString(content["txAmount"].(string))
    if !txAmount.Equal(order.Amount) {
        return "REJECT" // amount mismatch
    }

    // 3. Validate destination address matches
    destAddr, _ := content["destinationAddress"].(string)
    if destAddr != order.DestinationAddress {
        return "REJECT" // address mismatch
    }

    return "APPROVE"
}
```

**Additional Co-Signer deployment rules:**
- The Co-Signer host must be in a **private isolated network**.
- The Approval Callback Service must only accept requests from the Co-Signer host IP.
- Production API Keys **must** have a Callback URL configured.

---

## 4. Webhook Security

### Rules for Generated Code

**4-1. Verify signature before any processing.**
The SDK's `WebhookConverter.Convert()` handles verification. Reject events with invalid signatures immediately.

**4-2. Webhook endpoint should use HTTPS in production.**

**4-3. Idempotency is required.**
Safeheron retries delivery up to 7 times. Never double-credit a deposit or double-process a withdrawal.

**4-4. IP whitelist -- only accept from Safeheron egress IPs.**
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback -- terminal states are final.**

```go
func isTerminalStatus(status string) bool {
    switch status {
    case "COMPLETED", "SIGN_COMPLETED", "FAILED", "REJECTED", "CANCELLED":
        return true
    }
    return false
}

// In webhook handler:
if isTerminalStatus(currentDBStatus) {
    return // ignore late events for terminal transactions
}
```

**4-6. Handle out-of-order delivery.**
If a terminal-state event was already processed, discard subsequent intermediate-state events.

**4-7. Implement REST API polling as fallback.**
Do not rely solely on Webhooks. Run a periodic goroutine to poll the transaction list API.

---

## 5. Policy Configuration Security

- Configure tiered approval based on transaction amount.
- Always add a **catch-all blocking rule** at the bottom of the policy stack.
- Subscribe to `NO_MATCHING_TRANSACTION_POLICY` Webhook event.
- Perform regular security audits of policy rules.

---

## 6. API Key Security

- Scope API Key permissions to the **minimum required**.
- Register only the minimum necessary server IPs in the IP whitelist.
- Non-essential personnel must not have access to RSA private keys.
- Rotate API Keys periodically and immediately upon suspected compromise.

---

## Quick Reference

| Area | Key Rule |
|---|---|
| Private keys | PEM files outside project; never committed to git |
| Transfer addresses | Whitelist required; ONE_TIME_ADDRESS only for truly one-off payments |
| AML check | Mandatory before every outbound transfer via ToolsApi |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.CheckCoinAddress() |
| Amounts | `string` in API, `decimal.Decimal` in code; never float |
| Co-Signer callback | Validate customerRefId + amount + address against business system |
| Webhook | HTTPS preferred; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling as backup |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
