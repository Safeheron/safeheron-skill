# Common Error Codes & Troubleshooting

## API Error Codes

| Code | Message | Root Cause | Resolution |
|------|---------|-----------|------------|
| `1010` | Parameter decryption failed | Wrong **Safeheron platform public key** (`safeheronPublicKey`) in config | Verify and update `safeheronPublicKey` in your config dict -- copy the exact value from Safeheron Console |
| `1012` | Signature verification failed | Wrong **your RSA private key** (`privateKey`) in config, or private key not in PKCS8 format | Verify `privateKey` is PKCS8-encoded (strip BEGIN/END headers); ensure it matches the public key uploaded in Console |
| `9001` | Merchant unique business ID already exists | Duplicate `customerRefId` | Each call must use a unique `customerRefId`; use `str(uuid.uuid4())` |
| `9028` | MPC Sign policy not configured | First-time use of MPC Sign without a signing policy | Contact Safeheron Support by email to request MPC Sign policy activation |

---

## Configuration Errors

### Error: `Signature verification failed (1012)`

**Symptom:** Every API call returns error code 1012.

**Checklist:**
1. Is `privateKey` in **PKCS8 format**? Generate with:
   ```bash
   openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
   ```
2. Did you strip the PEM header/footer lines? The value should be pure base64 (no `-----BEGIN...` lines).
   Alternatively, use `privateKeyPemFile` to point to the PEM file directly.
3. Does your private key match the **public key** uploaded in Safeheron Console?
4. Is the config using `privateKey` (not `safeheronPublicKey` by mistake)?

---

### Error: `Parameter decryption failed (1010)`

**Symptom:** API call returns error code 1010.

**Checklist:**
1. Is `safeheronPublicKey` set to the **Safeheron platform public key** (from Console), not your own public key?
2. Did you copy the full value including all base64 characters (watch for line breaks)?
3. Note: Python SDK uses `safeheronPublicKey` (NOT `safeheronRsaPublicKey`).

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

**Explanation:** `customerRefId` idempotency is a feature -- if you retry with the **same** `customerRefId`, Safeheron returns the original transaction. Error 9001 only occurs if you're intentionally passing a non-unique ID and expecting a new transaction.

**Resolution:**
- For new transactions: always generate a fresh `str(uuid.uuid4())`.
- For retries after timeout: reuse the same `customerRefId` to retrieve the original result.

---

### Error: `9028` -- MPC Sign Policy Not Found

**Symptom:** `create_mpc_sign_transactions` returns error 9028.

**Resolution:** MPC Sign requires a special security policy to be configured. Contact Safeheron Support via email:
https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le

---

## Python-Specific Issues

### Error: `KeyError` on config dict

**Symptom:** `KeyError: 'apiKey'` or similar when creating an API instance.

**Resolution:** Ensure your config dict contains all required keys:
```python
config = {
    'apiKey': '...',
    'privateKey': '...',       # OR 'privateKeyPemFile': '/path/to/key.pem'
    'safeheronPublicKey': '...',  # NOT 'safeheronRsaPublicKey'
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}
```

---

### Error: Amount precision issues

```python
# WRONG -- float precision loss
amount = 0.1 + 0.2  # = 0.30000000000000004
param.txAmount = str(amount)  # "0.30000000000000004"

# CORRECT -- use Decimal
from decimal import Decimal
amount = Decimal("0.1") + Decimal("0.2")  # = Decimal("0.3")
param.txAmount = str(amount)  # "0.3"
```

---

### Error: Web3 API with Vault Account key

**Symptom:** Web3 API returns "account not found".

**Cause:** Web3 APIs require a **Web3 wallet**. Using a regular vault account key causes this error.

**Resolution:** Create a Web3 wallet in Safeheron Console, then copy its `accountKey`.

---

### Error: MPC Sign hash with "0x" prefix

```python
# WRONG -- includes "0x" prefix
param.dataList[0].data = "0xa1b2c3d4..."

# CORRECT -- strip "0x"
param.dataList[0].data = tx_hash.hex()  # already without "0x" if using bytes.hex()
# or if hash starts with "0x":
param.dataList[0].data = hash_str[2:]  # strip "0x"
```

---

## Fee-Related Issues

### Transaction fails with fee error

- If using `feeRateDto`, both `gasLimit` AND `feeRate` are required -- providing only `gasLimit` is insufficient.
- `txFeeLevel` and `feeRateDto` cannot both be set; `txFeeLevel` takes precedence.

### TRON fee estimation returns 0

- For TRON fee estimation, `sourceAddress` is required to check on-chain staking/energy.
- If the address has no TRX staked, bandwidth/energy is 0 and estimated fee will be based on TRX burn.

---

## Webhook / Callback Issues

| Symptom | Cause | Resolution |
|---------|-------|------------|
| Webhook not received | URL not accessible from public internet | Test with `curl` from Safeheron IP range |
| Webhook stops after URL change | Old URL deleted | Only deletion stops webhooks; URL modification redirects them |
| Duplicate webhook events | Safeheron retries on non-200 | Implement idempotency by checking `txKey` |
| Callback `Timestamp out of range` | Server clock drift > 5s | Sync server time with NTP |
| Co-Signer callback not triggered | Transaction not in approval flow | Callback fires only when a transaction needs signing approval |

---

## Nonce Issues (EVM Chains)

Nonce is assigned when a transaction is **approved** (enters SIGNING):
1. If `maxUseNonce` is null, only `chainNonce` is used.
2. `nonce < chainNonce` -> error "Nonce too low".
3. `nonce > maxUseNonce + 1` -> error "Nonce cannot exceed maxUseNonce+1".

---

## Safeheron Server Timezone

Safeheron's servers operate in **UTC+0**. All timestamps in API responses/webhooks are Unix milliseconds (timezone-agnostic), but any log-based debugging should account for UTC.
