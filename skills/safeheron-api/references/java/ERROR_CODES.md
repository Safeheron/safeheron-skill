# Common Error Codes & Troubleshooting

## API Error Codes

| Code | Message | Root Cause | Resolution |
|------|---------|-----------|------------|
| `1010` | Parameter decryption failed | Wrong **Safeheron platform public key** (`safeheronRsaPublicKey`) in config | Verify and update `safeheronRsaPublicKey` in `SafeheronConfig` — copy the exact value from Safeheron Console |
| `1012` | Signature verification failed | Wrong **your RSA private key** (`rsaPrivateKey`) in config, or private key not in PKCS8 format | Verify `rsaPrivateKey` is PKCS8-encoded (strip BEGIN/END headers); ensure it matches the public key uploaded in Console |
| `9001` | Merchant unique business ID already exists | Duplicate `customerRefId` | Each call must use a unique `customerRefId`; use `UUID.randomUUID().toString()` |
| `9028` | MPC Sign policy not configured | First-time use of MPC Sign without a signing policy | Contact Safeheron Support by email to request MPC Sign policy activation |

---

## Configuration Errors

### Error: `Signature verification failed (1012)`

**Symptom:** Every API call returns error code 1012.

**Checklist:**
1. Is `rsaPrivateKey` in **PKCS8 format**? Generate with:
   ```bash
   openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
   ```
2. Did you strip the PEM header/footer lines? The value should be pure base64 (no `-----BEGIN...` lines).
3. Does your private key match the **public key** uploaded in Safeheron Console?
4. Is the `SafeheronConfig` builder using `.rsaPrivateKey(...)` (not `.safeheronRsaPublicKey(...)` by mistake)?

---

### Error: `Parameter decryption failed (1010)`

**Symptom:** API call returns error code 1010.

**Checklist:**
1. Is `safeheronRsaPublicKey` set to the **Safeheron platform public key** (from Console), not your own public key?
2. Did you copy the full value including all base64 characters (watch for line breaks)?
3. Builder method: `.safeheronRsaPublicKey(...)` — not `.rsaPrivateKey(...)`.

---

### Error: `Illegal IP`

**Symptom:** Co-Signer logs show "Illegal IP", or API returns IP-related error.

**Resolution:** Register your server's IP address in Safeheron Console → API Key → IP Whitelist.
Safeheron supports IPv4 and IPv6 formats. Safeheron's own outbound IPs (for webhooks/callbacks):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

---

### Error: `9001` — Duplicate customerRefId

**Symptom:** API returns 9001 on retry.

**Explanation:** `customerRefId` idempotency is a feature — if you retry with the **same** `customerRefId`, Safeheron returns the original transaction. Error 9001 only occurs if you're intentionally passing a non-unique ID and expecting a new transaction.

**Resolution:**
- For new transactions: always generate a fresh `UUID.randomUUID().toString()`.
- For retries after timeout: reuse the same `customerRefId` to retrieve the original result.

---

### Error: `9028` — MPC Sign Policy Not Found

**Symptom:** `createMPCSignTransactions` returns error 9028.

**Resolution:** MPC Sign requires a special security policy to be configured. Contact Safeheron Support via email using the official template:
https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le

---

## SDK Usage Errors

### Error: Direct interface call without ServiceExecutor

```java
// WRONG — will NOT work
CreateAccountResponse resp = accountApi.createAccount(req);  // ❌

// CORRECT
CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));  // ✓
```

---

### Error: Wrong data type for pagination fields

```java
// WRONG — int
req.setPageSize(10);   // ❌ compile error — expects Long

// CORRECT — Long
req.setPageSize(10L);  // ✓
req.setPageNumber(1L); // ✓
```

---

### Error: Amount as numeric type

```java
// WRONG — float/double causes precision loss
req.setTxAmount(0.1);      // ❌ compile error — expects String
req.setTxAmount(0.1f);     // ❌

// CORRECT — always String
req.setTxAmount("0.1");    // ✓
req.setTxAmount("100.50"); // ✓
```

---

### Error: `requestTimeout` as int

```java
// WRONG
.requestTimeout(20000)    // ❌ compile error — expects Long

// CORRECT
.requestTimeout(20000L)   // ✓
```

---

### Error: Web3 API with Vault Account key

**Symptom:** Web3 API returns "account not found".

**Cause:** Web3 APIs require a **Web3 wallet**. Using a regular vault account key causes this error.

**Resolution:** Create a Web3 wallet in Safeheron Console, then copy its `accountKey`.

---

### Error: MPC Sign hash with "0x" prefix

```java
// WRONG — includes "0x" prefix
hashItem.setData("0xa1b2c3d4...");   // ❌

// CORRECT — strip "0x"
hashItem.setData(hash.substring(2)); // ✓ (when hash is "0x...")
// or ensure hash is already without prefix
```

---

## Fee-Related Issues

### Transaction fails with fee error

- If using `feeRateDto`, both `gasLimit` AND `feeRate` are required — providing only `gasLimit` is insufficient.
- `txFeeLevel` and `feeRateDto` cannot both be set; `txFeeLevel` takes precedence.

### TRON fee estimation returns 0

- For TRON fee estimation, `sourceAddress` is required to check on-chain staking/energy.
- If the address has no TRX staked, bandwidth/energy is 0 and estimated fee will be based on TRX burn.

---

## Webhook / Callback Issues

| Symptom | Cause | Resolution |
|---------|-------|------------|
| Webhook not received | URL not accessible from public internet | Test with `curl` from Safeheron IP range; avoid ngrok/webhook.site domains |
| Webhook stops after URL change | Old URL deleted | Only deletion stops webhooks; URL modification redirects them |
| Duplicate webhook events | Safeheron retries on non-200 | Implement idempotency by checking `txKey` |
| Callback `Timestamp out of range` | Server clock drift > 5s | Sync server time with NTP |
| Co-Signer callback not triggered | Transaction not in approval flow | Callback fires only when a transaction needs signing approval |

---

## Nonce Issues (EVM Chains)

Nonce is assigned when a transaction is **approved** (enters SIGNING):
1. If `maxUseNonce` is null, only `chainNonce` is used.
2. `nonce < chainNonce` → error "Nonce too low".
3. `nonce > maxUseNonce + 1` → error "Nonce cannot exceed maxUseNonce+1".
4. `chainNonce <= nonce <= maxUseNonce` → confirmation dialog shown.

---

## Safeheron Server Timezone

Safeheron's servers operate in **UTC+0**. All timestamps in API responses/webhooks are Unix milliseconds (timezone-agnostic), but any log-based debugging should account for UTC.
