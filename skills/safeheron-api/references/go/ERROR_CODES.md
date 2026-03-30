# Common Error Codes & Troubleshooting

## API Error Codes

| Code | Message | Root Cause | Resolution |
|------|---------|-----------|------------|
| `1010` | Parameter decryption failed | Wrong **Safeheron platform public key** PEM file | Verify `SafeheronRsaPublicKey` file path points to the correct Safeheron platform public key PEM file |
| `1012` | Signature verification failed | Wrong **your RSA private key** PEM file | Verify `RsaPrivateKey` file path and ensure the PEM file matches the public key uploaded in Console |
| `9001` | Merchant unique business ID already exists | Duplicate `customerRefId` | Each call must use a unique `customerRefId`; use `uuid.New().String()` |
| `9028` | MPC Sign policy not configured | First-time use of MPC Sign without a signing policy | Contact Safeheron Support by email to request MPC Sign policy activation |

---

## Configuration Errors

### Error: `Signature verification failed (1012)`

**Symptom:** Every API call returns error code 1012.

**Checklist:**
1. Does `RsaPrivateKey` point to a valid PEM file? The Go SDK reads PEM files by path (unlike Java which uses base64 strings).
2. Does the PEM file contain a valid RSA private key with `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----` header?
3. Does the private key match the **public key** uploaded in Safeheron Console?
4. Is the file readable by the running process? (`chmod 600`)

---

### Error: `Parameter decryption failed (1010)`

**Symptom:** API call returns error code 1010.

**Checklist:**
1. Is `SafeheronRsaPublicKey` pointing to the **Safeheron platform public key** PEM file (from Console), not your own public key?
2. Is the PEM file complete and not corrupted?
3. Are the `RsaPrivateKey` and `SafeheronRsaPublicKey` paths not swapped?

---

### Error: `Illegal IP`

**Symptom:** Co-Signer logs show "Illegal IP", or API returns IP-related error.

**Resolution:** Register your server's IP address in Safeheron Console -> API Key -> IP Whitelist.
Safeheron supports IPv4 and IPv6 formats. Safeheron's own outbound IPs (for webhooks/callbacks):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

---

### Error: `9001` -- Duplicate customerRefId

**Symptom:** API returns 9001 on retry.

**Explanation:** `customerRefId` idempotency is a feature -- if you retry with the **same** `customerRefId`, Safeheron returns the original transaction. Error 9001 only occurs if you are intentionally passing a non-unique ID and expecting a new transaction.

**Resolution:**
- For new transactions: always generate a fresh `uuid.New().String()`.
- For retries after timeout: reuse the same `customerRefId` to retrieve the original result.

---

### Error: `9028` -- MPC Sign Policy Not Found

**Symptom:** `CreateMpcSign` returns error 9028.

**Resolution:** MPC Sign requires a special security policy to be configured. Contact Safeheron Support via email.

---

## Go SDK-Specific Errors

### Error: `open pems/xxx.pem: no such file or directory`

**Symptom:** Panic at startup when creating the client.

**Cause:** The PEM file path in `ApiConfig` is incorrect or the file does not exist.

**Resolution:** Verify the file path is correct relative to the working directory, or use an absolute path.

---

### Error: `if err != nil` -- Error Handling Pattern

All Safeheron Go SDK methods return `error`. Always check:

```go
// CORRECT -- always check error
var resp api.ListAccountResponse
err := accountApi.ListAccounts(req, &resp)
if err != nil {
    log.Printf("API call failed: %v", err)
    return
}

// WRONG -- ignoring error
// accountApi.ListAccounts(req, &resp) // never do this
```

---

### Pointer vs Value Parameters

All response parameters must be passed as **pointers**:

```go
// CORRECT -- pass pointer
var resp api.CreateAccountResponse
err := accountApi.CreateAccount(req, &resp)  // &resp

// WRONG -- passing value (will not work)
// err := accountApi.CreateAccount(req, resp)  // missing &
```

---

### Error: Amount as numeric type

```go
// WRONG -- float causes precision loss
// req.TxAmount = fmt.Sprintf("%f", 0.1)  // "0.100000" -- wrong precision

// CORRECT -- always string literal
req.TxAmount = "0.1"

// CORRECT -- using decimal library
import "github.com/shopspring/decimal"
amount, _ := decimal.NewFromString("0.1")
req.TxAmount = amount.String()
```

---

### Error: Web3 API with Vault Account key

**Symptom:** Web3 API returns "account not found".

**Cause:** Web3 APIs require a **Web3 wallet**. Using a regular vault account key causes this error.

**Resolution:** Create a Web3 wallet in Safeheron Console, then copy its `accountKey`.

---

### Error: MPC Sign hash with "0x" prefix

```go
// WRONG -- includes "0x" prefix
data := "0xa1b2c3d4..."

// CORRECT -- strip "0x"
data := hash[2:] // when hash starts with "0x"
```

---

## Fee-Related Issues

### Transaction fails with fee error

- If using custom fee rate, both `gasLimit` AND `feeRate` are required.
- `TxFeeLevel` and custom fee rate cannot both be set; `TxFeeLevel` takes precedence.

### TRON fee estimation returns 0

- For TRON fee estimation, `sourceAddress` is required to check on-chain staking/energy.

---

## Webhook / Callback Issues

| Symptom | Cause | Resolution |
|---------|-------|------------|
| Webhook not received | URL not accessible from public internet | Test with `curl` from Safeheron IP range |
| Duplicate webhook events | Safeheron retries on non-200 | Implement idempotency by checking `txKey` |
| Callback `Timestamp out of range` | Server clock drift > 5s | Sync server time with NTP |

---

## Nonce Issues (EVM Chains)

Nonce is assigned when a transaction is **approved** (enters SIGNING):
1. If `maxUseNonce` is null, only `chainNonce` is used.
2. `nonce < chainNonce` -> error "Nonce too low".
3. `nonce > maxUseNonce + 1` -> error "Nonce cannot exceed maxUseNonce+1".

---

## Safeheron Server Timezone

Safeheron's servers operate in **UTC+0**. All timestamps in API responses/webhooks are Unix milliseconds (timezone-agnostic).
