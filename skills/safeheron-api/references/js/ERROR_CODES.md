# Common Error Codes & Troubleshooting

## API Error Codes

| Code | Message | Root Cause | Resolution |
|------|---------|-----------|------------|
| `1010` | Parameter decryption failed | Wrong **Safeheron platform public key** (`safeheronRsaPublicKey`) in config | Verify and update `safeheronRsaPublicKey` -- copy the exact value from Safeheron Console |
| `1012` | Signature verification failed | Wrong **your RSA private key** (`rsaPrivateKey`) in config, or private key not in PKCS8 format | Verify `rsaPrivateKey` is PKCS8-encoded PEM; ensure it matches the public key uploaded in Console |
| `9001` | Merchant unique business ID already exists | Duplicate `customerRefId` | Each call must use a unique `customerRefId`; use `crypto.randomUUID()` |
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
2. Is the full PEM string provided (including `-----BEGIN...` headers)?
3. Does your private key match the **public key** uploaded in Safeheron Console?
4. Is the config using `rsaPrivateKey` (not `safeheronRsaPublicKey` by mistake)?

---

### Error: `Parameter decryption failed (1010)`

**Symptom:** API call returns error code 1010.

**Checklist:**
1. Is `safeheronRsaPublicKey` set to the **Safeheron platform public key** (from Console), not your own public key?
2. Did you copy the full value?
3. Config field: `safeheronRsaPublicKey` -- not `rsaPrivateKey`.

---

### Error: `Illegal IP`

**Symptom:** Co-Signer logs show "Illegal IP", or API returns IP-related error.

**Resolution:** Register your server's IP address in Safeheron Console -> API Key -> IP Whitelist.
Safeheron's own outbound IPs (for webhooks/callbacks):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

---

### Error: `9001` -- Duplicate customerRefId

**Symptom:** API returns 9001 on retry.

**Explanation:** `customerRefId` idempotency is a feature -- if you retry with the **same** `customerRefId`, Safeheron returns the original transaction. Error 9001 only occurs if you're intentionally passing a non-unique ID and expecting a new transaction.

**Resolution:**
- For new transactions: always generate a fresh `crypto.randomUUID()`.
- For retries after timeout: reuse the same `customerRefId` to retrieve the original result.

---

### Error: `9028` -- MPC Sign Policy Not Found

**Symptom:** `createMPCSignTransactions` returns error 9028.

**Resolution:** MPC Sign requires a special security policy. Contact Safeheron Support via email:
https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le

---

## JS/TS SDK-Specific Errors

### Error: SafeheronError Type Check

Always use `instanceof` to check for SDK errors:

```typescript
import { SafeheronError } from '@safeheron/api-sdk';

try {
  const result = await accountApi.createAccount({ accountName: 'test' });
} catch (e) {
  if (e instanceof SafeheronError) {
    // SDK error with code and message
    console.error(`Error code: ${e.code}, message: ${e.message}`);
  } else if (e instanceof Error) {
    // Network error, timeout, etc.
    console.error(`Network error: ${e.message}`);
  } else {
    console.error('Unknown error:', e);
  }
}
```

### Error: Async/Await Not Used

All SDK methods return Promises. Forgetting `await` causes silent failures:

```typescript
// WRONG -- missing await
const result = transactionApi.createTransactions(req);  // returns Promise, not result
console.log(result.txKey);  // undefined!

// CORRECT
const result = await transactionApi.createTransactions(req);
console.log(result.txKey);  // works
```

### Error: Unhandled Promise Rejection

Always wrap async SDK calls in try/catch or attach `.catch()`:

```typescript
// CORRECT -- try/catch
async function createWallet() {
  try {
    const result = await accountApi.createAccount({ accountName: 'test' });
    return result;
  } catch (e) {
    if (e instanceof SafeheronError) {
      console.error(`API error: ${e.code} ${e.message}`);
    }
    throw e;
  }
}

// CORRECT -- .catch()
accountApi.createAccount({ accountName: 'test' })
  .then(result => console.log(result.accountKey))
  .catch(e => console.error(e));
```

---

### Error: Web3 API with Vault Account key

**Symptom:** Web3 API returns "account not found".

**Cause:** Web3 APIs require a **Web3 wallet**. Using a regular vault account key causes this error.

**Resolution:** Create a Web3 wallet in Safeheron Console, then copy its `accountKey`.

---

### Error: MPC Sign hash with "0x" prefix

```typescript
// WRONG -- includes "0x" prefix
dataList: [{ data: '0xa1b2c3d4...' }]   // will fail

// CORRECT -- strip "0x"
dataList: [{ data: hash.substring(2) }]  // strip "0x" prefix
```

---

### Error: Amount as number type

```typescript
// WRONG -- number causes precision loss
txAmount: 0.1,      // may not work or cause precision issues

// CORRECT -- always string
txAmount: '0.1',    // correct
txAmount: '100.50', // correct
```

---

## Fee-Related Issues

### Transaction fails with fee error

- If using `feeRateDto`, both `gasLimit` AND `feeRate` are required -- providing only `gasLimit` is insufficient.
- `txFeeLevel` and `feeRateDto` cannot both be set; `txFeeLevel` takes precedence.

### TRON fee estimation returns 0

- For TRON fee estimation, `sourceAddress` is required to check on-chain staking/energy.

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
4. `chainNonce <= nonce <= maxUseNonce` -> confirmation dialog shown.

---

## Safeheron Server Timezone

Safeheron's servers operate in **UTC+0**. All timestamps in API responses/webhooks are Unix milliseconds (timezone-agnostic).
