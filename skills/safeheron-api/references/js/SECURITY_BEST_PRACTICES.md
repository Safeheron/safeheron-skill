# Security Best Practices

> **These rules are mandatory.** When this Skill is active, every piece of generated code and every architecture recommendation must comply without exception.

---

## 1. Credential & Key Management

### Rules for Generated Code

- **Private keys must never be stored in plaintext** -- not in config files, environment variables, or source code.
- Use a secrets management service appropriate for the deployment target:

| Deployment Target | Required Solution |
|---|---|
| Cloud (AWS) | AWS KMS |
| Cloud (GCP) | GCP KMS |
| Self-hosted | HashiCorp Vault |
| Local development only | Read from a file **outside the project directory**; the file must never be committed to git |

### Code Pattern -- Loading Credentials

```typescript
const config: SafeheronConfig = {
  baseUrl: 'https://api.safeheron.vip',
  apiKey: '${SAFEHERON_API_KEY}',//todo Replace with the API Key you read from Safeheron Console
  rsaPrivateKey: '${RSA_PRIVATE_KEY}', //todo Replace with the RSA private key you read from Vault/KMS
  safeheronRsaPublicKey: '${SAFEHERON_PLATFORM_PUBLIC_KEY}',//todo Replace with the Safeheron platform public key from Safeheron Console
  requestTimeout: 20000,
};

// CORRECT: Load from file (development only)
// import { readFileSync } from 'fs';
// rsaPrivateKey: readFileSync('/secure/path/outside/project/api_pkcs8.pem', 'utf8'),

// WRONG -- never do this:
// apiKey: 'sk-live-xxxxxxxxxxxx',
// rsaPrivateKey: '-----BEGIN RSA PRIVATE KEY-----\nMIIE...',
```

---

## 2. Transfer & Whitelist Security

### Rules for Generated Code

**2-1. `customerRefId` must be a business-unique ID for idempotency.**
Generate it and persist it to your database **before** calling the Safeheron API. On timeout, retry with the same ID.

```typescript
import crypto from 'crypto';

// CORRECT: generate and save before API call
const customerRefId = crypto.randomUUID();
await db.saveWithdrawalOrder({ customerRefId, amount, toAddress, status: 'PENDING' });

// Then call Safeheron
await transactionApi.createTransactions({ customerRefId, ... });
```

**2-2. Whitelist addresses are required for formal transfers.**
One-time address (`ONE_TIME_ADDRESS`) must only be used for genuinely temporary, one-off payment scenarios.

**2-3. AML check is mandatory before every transfer.**
Call `toolsApi.amlCheckerRequest()` to screen the destination address before creating the transaction.

```typescript
const submitResp = await toolsApi.amlCheckerRequest({
  network: 'Ethereum',
  address: destinationAddress,
});
// Poll for result and block high-risk addresses
```

**2-4. Validate address format before whitelist add or transfer.**

```typescript
const checkResp = await coinApi.checkCoinAddress({
  coinKey: 'ETHEREUM_ETH',
  address: destinationAddress,
});
if (!checkResp.addressValid) {
  throw new Error(`Invalid address format: ${destinationAddress}`);
}
```

**2-5. Amounts must use `string` in API requests.**
Never use `number` for monetary values -- precision loss is possible with floating point.

```typescript
// CORRECT
txAmount: '0.001',

// WRONG -- never use number for amounts
// txAmount: 0.001,
```

---

## 3. Co-Signer Approval Callback

### Rules for Generated Code

**The Co-Signer callback service must never blindly approve all transactions.**
Every approval callback must implement the following business validations before returning `APPROVE`:

| Validation | What to Check |
|---|---|
| `customerRefId` | Must match a real pending business order in your system |
| Amount | Must exactly match the amount recorded in your business system for that order |
| Destination address | Must exactly match the address recorded in your business system for that order |

```typescript
app.post('/cosigner/callback', async (req, res) => {
  // 1. Decrypt and verify signature (always first)
  const payload = converter.requestV3convert(req.body);
  const data = JSON.parse(payload);

  // 2. Look up the business order
  const order = await orderService.findByCustomerRefId(data.customerRefId);
  if (!order) {
    console.warn('Unknown customerRefId:', data.customerRefId);
    return res.json(converter.responseV3convert({ action: 'REJECT', approvalId: data.approvalId }));
  }

  // 3. Validate amount matches business system
  if (data.txAmount !== order.amount) {
    console.warn('Amount mismatch');
    return res.json(converter.responseV3convert({ action: 'REJECT', approvalId: data.approvalId }));
  }

  // 4. Validate destination address matches business system
  if (data.destinationAddress?.toLowerCase() !== order.destinationAddress?.toLowerCase()) {
    console.warn('Address mismatch');
    return res.json(converter.responseV3convert({ action: 'REJECT', approvalId: data.approvalId }));
  }

  return res.json(converter.responseV3convert({ action: 'APPROVE', approvalId: data.approvalId }));
});
```

**Additional Co-Signer deployment rules:**
- The Co-Signer host must be in a **private isolated network** -- never exposed to the public internet.
- The Approval Callback Service must only accept requests from the Co-Signer host IP.
- Production API Keys configured for Co-Signer **must** have a Callback URL configured.

---

## 4. Webhook Security

### Rules for Generated Code

**4-1. Verify signature before any processing.**
The SDK's `WebHookConverter.convertWebHook()` verifies signatures automatically. If not using the SDK converter, manually verify before processing.

**4-2. Webhook endpoint must use HTTPS.**
Never expose an HTTP Webhook endpoint in production.

**4-3. Idempotency is required.**
Safeheron retries delivery up to 7 times. The handler must safely process duplicate events.

**4-4. IP whitelist -- only accept from Safeheron egress IPs.**
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback -- terminal states are final.**

**4-6. Handle out-of-order delivery.**

```typescript
app.post('/webhook', (req, res) => {
  // 1. Verify signature -- reject if invalid
  try {
    const decrypted = converter.convertWebHook(req.body);
    // 2. Return 200 immediately; process asynchronously
    eventQueue.publish(decrypted);
  } catch (e) {
    console.error('Webhook signature verification failed:', e);
  }
  res.json({ code: 200, message: 'SUCCESS' });
});

// In async processor:
async function processWebhookEvent(event: any) {
  const tx = await transactionRepository.findByTxKey(event.txKey);

  // 4-5 & 4-6: Never roll back status; terminal state wins
  if (tx && isTerminalStatus(tx.status)) {
    console.log(`Ignoring late event for terminal transaction: txKey=${event.txKey}`);
    return;
  }

  await transactionRepository.upsert({ txKey: event.txKey, status: event.transactionStatus });
}

function isTerminalStatus(status: string): boolean {
  return ['COMPLETED', 'SIGN_COMPLETED', 'FAILED', 'REJECTED', 'CANCELLED'].includes(status);
}
```

**4-7. Implement REST API polling as fallback.**

**4-8. Handle failed Webhook re-delivery.**

---

## 5. Policy Configuration Security

- Configure tiered approval based on transaction amount.
- Always add a **catch-all blocking rule at the bottom** of the policy stack.
- Subscribe to `NO_MATCHING_TRANSACTION_POLICY` Webhook event as an operational alert.
- After policy configuration, validate by testing with small amounts.

---

## 6. API Key Security

- Scope API Key permissions to the **minimum required** for each use case.
- Register only the minimum necessary server IPs in the IP whitelist.
- Non-essential personnel must not have access to RSA private keys.
- Rotate API Keys periodically and immediately upon suspected compromise.

---

## Quick Reference

| Area | Key Rule |
|---|---|
| Private keys | Vault/KMS only; never plaintext |
| Transfer addresses | Whitelist required; ONE_TIME_ADDRESS only for truly one-off payments |
| AML check | Mandatory before every outbound transfer via ToolsApi |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.checkCoinAddress() |
| Amounts | String in API; never floating-point number |
| Co-Signer callback | Validate customerRefId + amount + address against business system |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling + oneTransactions() |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
