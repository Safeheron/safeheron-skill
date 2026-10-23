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
sc := safeheron.Client{Config: safeheron.ApiConfig{
    BaseUrl:               "https://api.safeheron.vip",
    ApiKey:                "${SAFEHERON_API_KEY}",
    RsaPrivateKey:         "${RSA_PRIVATE_KEY_PATH}",
    SafeheronRsaPublicKey: "${SAFEHERON_PUBLIC_KEY_PATH}",
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
Use KYA screening via `ComplianceApi` to screen the destination address before creating the transaction.

```go
// Required before creating any outbound transaction
createReq := api.KyaScreeningRequest{
    Address:   destinationAddress,
    ChainType: "EVM", // published example value; valid values via KyaSupportedNetworks()
    Providers: []string{"Chainalysis"},
}
var created api.KyaScreeningCreateResponse
if err := complianceApi.KyaScreeningCreate(createReq, &created); err != nil {
    panic(err)
}

pollReq := api.KyaScreeningOneRequest{ScreenId: created.ScreenId}
var result api.KyaScreeningOneResponse
for i := 0; i < 30; i++ {
    if err := complianceApi.KyaScreeningOne(pollReq, &result); err != nil {
        panic(err)
    }
    if result.Status == "FINISHED" {
        break
    }
    time.Sleep(2 * time.Second)
}
// Fail closed: an unfinished screening is NOT a pass
if result.Status != "FINISHED" {
    panic(fmt.Sprintf("KYA screening not finished for %s, route to manual review", destinationAddress))
}
if len(result.Orders) == 0 {
    panic(fmt.Sprintf("KYA screening returned no provider result for %s", destinationAddress))
}
// Whitelist: only LOW / MEDIUM pass (adjust per your policy); HIGH, SEVERE, UNKNOWN, a missing level or FAILED are blocked
allowedRiskLevels := map[string]bool{"LOW": true, "MEDIUM": true}
for _, order := range result.Orders {
    if order.Status == "SKIPPED" {
        continue // manually skipped in Web Console
    }
    if order.Status != "COMPLETED" {
        panic(fmt.Sprintf("KYA screening %s (%s), route to manual review", order.Status, order.Provider))
    }
    if !allowedRiskLevels[order.RiskLevel] {
        panic(fmt.Sprintf("AML check failed: %s is %q risk", destinationAddress, order.RiskLevel))
    }
}
```

**2-4. Validate address format before whitelist add or transfer.**

```go
checkReq := api.CheckCoinAddressRequest{
    CoinKey:           "ETHEREUM_ETH",
    Address:           destinationAddress,
    CheckAddressValid: true, // required — without it AddressValid is not verified and the check below is meaningless
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
amount := decimal.RequireFromString("0.001") // or decimal.NewFromString(s) and handle the error
req.TxAmount = amount.String()

// WRONG -- never use float for amounts
// amount := 0.001
```

---

## 3. Co-Signer Approval Callback

### Rules for Generated Code

**The Co-Signer callback service must never blindly approve anything.** Use the handler (`decide()`) in [COSIGNER.md](COSIGNER.md) as the REQUIRED pattern — do not write a reduced version. It enforces:

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
The SDK's `WebhookConverter.Convert()` handles verification. Respond `401` on an invalid signature and `500` on an internal error (decrypt / parse / durable store); respond `{"code":"200","message":"SUCCESS"}` **only** after the event is verified and **durably** stored (DB inbox / durable MQ) — any non-SUCCESS response makes Safeheron retry (6 times: 30s, 1m, 5m, 1h, 12h, 24h). An in-memory channel or fire-and-forget goroutine loses events acked just before a crash, and `ResendFailed` cannot re-push `TOKEN_BLACKLIST_ADDRESS_HIT` (the other published event types can be re-pushed, within 7 days). See [WEBHOOK.md](WEBHOOK.md).

**4-2. Webhook endpoint should use HTTPS in production.**

**4-3. Idempotency is required.**
Failed deliveries are retried 6 times (30s, 1m, 5m, 1h, 12h, 24h) and duplicates can arrive anyway. Never double-credit a deposit or double-process a withdrawal: write the idempotency marker and the business side-effects in the **same DB transaction** (never commit the marker first and process afterwards).

**4-4. IP whitelist -- only accept from Safeheron egress IPs** (not listed in the published API reference — confirm the current list with Safeheron Support).
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
| AML check | Mandatory before every outbound transfer via ComplianceApi KYA screening |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.CheckCoinAddress() |
| Amounts | `string` in API, `decimal.Decimal` in code; never float |
| Co-Signer callback | Use the COSIGNER.md handler (`decide()`): rules per transactionType (order + coin + address + amount + memo for NORMAL), AML/KYT whitelist, default REJECT |
| Webhook | HTTPS preferred; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling as backup |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
