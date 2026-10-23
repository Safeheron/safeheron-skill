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
Use KYA screening via `ComplianceApi` to screen the destination address before creating the transaction. Block or alert on HIGH / SEVERE (and, by default, UNKNOWN) risk results.

```typescript
const created = await complianceApi.createKyaScreening({
  address: destinationAddress,
  chainType: 'EVM',        // see kyaSupportedNetworks()
  network: 'Ethereum',
  providers: ['Chainalysis'],
});

let result: any;
for (let i = 0; i < 30; i++) {
  result = await complianceApi.kyaScreeningOne({ screenId: created.screenId });
  if (result.status === 'FINISHED') break;
  await new Promise(resolve => setTimeout(resolve, 2000));
}
// Fail closed: an unfinished screening is NOT a pass
if (result.status !== 'FINISHED') {
  throw new Error(`KYA screening not finished for ${destinationAddress}, route to manual review`);
}
if (!Array.isArray(result.orders) || result.orders.length === 0) {
  throw new Error(`KYA screening returned no provider result for ${destinationAddress}`);
}
// Whitelist: only LOW / MEDIUM pass (adjust per your policy); HIGH, SEVERE, UNKNOWN, a missing level or FAILED are blocked
const allowedRiskLevels = new Set(['LOW', 'MEDIUM']);
for (const order of result.orders) {
  if (order.status === 'SKIPPED') {
    continue;   // manually skipped in Web Console
  }
  if (order.status !== 'COMPLETED') {
    throw new Error(`KYA screening ${order.status} (${order.provider}), route to manual review`);
  }
  if (!allowedRiskLevels.has(order.riskLevel)) {
    throw new Error(`AML check failed: ${destinationAddress} is ${order.riskLevel} risk`);
  }
}
```

**2-4. Validate address format before whitelist add or transfer.**

```typescript
const checkResp = await coinApi.checkCoinAddress({
  coinKey: 'ETHEREUM_ETH',
  address: destinationAddress,
  checkAddressValid: true,  // required — without it addressValid is not checked
  checkAml: true,           // without it amlValid is always true
});
if (!checkResp.addressValid) {
  throw new Error(`Invalid address format: ${destinationAddress}`);
}
if (!checkResp.amlValid) {
  throw new Error(`Address is on Safeheron's AML blacklist: ${destinationAddress}`);  // blacklist only — still run KYA (2-3)
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

**The Co-Signer callback service must never blindly approve anything.** Use the handler and `decide()` in [COSIGNER.md](COSIGNER.md) as the REQUIRED pattern — do not write a reduced version. It enforces:

| Rule | What it checks |
|---|---|
| Default REJECT | Unknown callback types, unknown `transactionType`, evaluation errors → `REJECT`; unverifiable requests → HTTP 500, no signed response |
| `MPC_SIGN` / `WEB3_SIGN` | Verified against your pending sign request (`dataList`, `messageHash` / `message` / `transaction`) — never auto-approved |
| `NORMAL` / `UTXO_BATCH_TRANSFER` | `customerRefId` matches a real pending order; coin (`coinKey`) and **every** destination address + amount + memo match it (UTXO coins use `destinationAddressList`) |
| `UTXO_COLLECTION` / `AUTO_FUEL` | Destination account type is `VAULT_ACCOUNT` (+ fuel policy) |
| `AUTO_SWEEP` | Destination account and USD amount match your Auto Sweep policy |
| `GAS_SERVICE_DEPOSIT` | Deposit exists in your records and destination account type is `PROFILE_ACCOUNT` |
| AML / KYT | Only `LOW` / `MEDIUM` pass (`HIGH` / `SEVERE` / `UNKNOWN` / unexpected values rejected); `FAILED` or unknown status rejected; wait (neither APPROVE nor REJECT) while `IN_PROGRESS` / `PENDING`, then reject after ~5 min — see [AML / KYT Handling](COSIGNER.md#aml--kyt-handling-in-the-callback) |

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

**4-3. Idempotency and durability are required.**
Failed deliveries are retried 6 times (30s, 1m, 5m, 1h, 12h, 24h) and duplicates can arrive anyway. Ack only after the verified event is **durably** stored (DB inbox / durable MQ — not an in-memory queue), and commit the idempotency marker together with the business side-effects in one DB transaction. `resendFailed` re-pushes TRANSACTION_*, MPC_SIGN_*, WEB3_SIGN_*, ILLEGAL_IP_REQUEST, NO_MATCHING_TRANSACTION_POLICY, WHITELIST_*, GAS_BALANCE_WARNING and AML_KYT_ALERT — only `TOKEN_BLACKLIST_ADDRESS_HIT` cannot be re-pushed. Full pattern: [WEBHOOK.md](WEBHOOK.md) Step 2–3.

**4-4. IP whitelist -- only accept from Safeheron egress IPs.**
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback -- terminal states are final.**

**4-6. Handle out-of-order delivery.**

```typescript
app.post('/webhook', async (req, res) => {
  try {
    // 1. Verify signature + decrypt — convertWebHook() throws Error('webhook signature verification failed') on a bad sig
    const { eventType, eventDetail } = JSON.parse(converter.convertWebHook(req.body));
    // 2. Durably store before acking (DB inbox insert or durable MQ publish with confirmation) —
    //    never an in-memory queue: the event is lost if the process crashes after the ack
    await eventInbox.saveIfAbsent({ eventType, eventDetail });
  } catch (e) {
    if (e instanceof Error && e.message === 'webhook signature verification failed') {
      console.error('Webhook signature verification FAILED — possible tampering:', e);
      return res.status(401).end();
    }
    // Internal error (decrypt / parse / store) — a non-SUCCESS response makes Safeheron retry
    // (30s, 1m, 5m, 1h, 12h, 24h); after that, recover via re-push (`resendFailed` — every event type except TOKEN_BLACKLIST_ADDRESS_HIT) or REST polling
    console.error('Webhook processing error:', e);
    return res.status(500).end();
  }
  // 3. Ack ONLY after the event is verified and durably stored.
  //    Body per docs: code is the string '200'; any non-SUCCESS response is retried by Safeheron
  res.json({ code: '200', message: 'SUCCESS' });
});

// In the inbox worker — `event` is the eventDetail of a TRANSACTION_* event. Run it inside one DB transaction
// with marking the inbox row done (see WEBHOOK.md Step 3):
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
| AML check | Mandatory before every outbound transfer via ComplianceApi KYA screening |
| Address validation | Mandatory before whitelist add or transfer via CoinApi.checkCoinAddress() |
| Amounts | String in API; never floating-point number |
| Co-Signer callback | Use the handler and `decide()` in COSIGNER.md: rules per transactionType (order + coin + address + amount + memo for NORMAL), AML/KYT whitelist, default REJECT |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling + oneTransactions() |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |
