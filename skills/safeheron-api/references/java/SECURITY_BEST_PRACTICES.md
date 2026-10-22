# Security Best Practices

> **These rules are mandatory.** When this Skill is active, every piece of generated code and every architecture recommendation must comply without exception. When user requests involve credentials, transfers, Webhooks, Co-Signer, or deployment, proactively explain the applicable requirements — do not wait for the user to ask.

---

## 1. Credential & Key Management

### Rules for Generated Code

- **Private keys must never be stored in plaintext** — not in config files, environment variables, or source code.
- Use a secrets management service appropriate for the deployment target:

| Deployment Target | Required Solution |
|---|---|
| Cloud (AWS) | AWS KMS |
| Cloud (GCP) | GCP KMS |
| Self-hosted | HashiCorp Vault |
| Local development only | Read from a file **outside the project directory**; the file must never be committed to git |


### Code Pattern — Loading Credentials

```java
SafeheronConfig config = SafeheronConfig.builder()
        .apiKey("${SAFEHERON_API_KEY}")//todo Replace with the API Key you read from Safeheron Console
        .rsaPrivateKey("${RSA_PRIVATE_KEY}")//todo Replace with the RSA private key you read from Vault/KMS
        .safeheronRsaPublicKey("${SAFEHERON_PLATFORM_PUBLIC_KEY}")//todo Replace with the Safeheron platform public key from Safeheron Console
        .build();

// CORRECT: Load from AWS Secrets Manager (production)
// String privateKey = awsSecretsManagerClient.getSecretValue("safeheron/rsa-private-key");

// WRONG — never do this:
// .apiKey("sk-live-xxxxxxxxxxxx")
// .rsaPrivateKey("-----BEGIN RSA PRIVATE KEY-----\nMIIE...")
```

---

## 2. Transfer & Whitelist Security

### Rules for Generated Code

**2-1. `customerRefId` must be a business-unique ID for idempotency.**
Generate it and persist it to your database **before** calling the Safeheron API. On timeout, retry with the same ID.

**2-2. Whitelist addresses are required for formal transfers.**
One-time address (`ONE_TIME_ADDRESS`) must only be used for genuinely temporary, one-off payment scenarios. Exchange hot wallets, partner addresses, and internal accounts must all be added to the whitelist first.

```java
// CORRECT: use whitelist address type for recurring/formal transfers
req.setDestinationAccountType("WHITELISTING_ACCOUNT");

// ONE_TIME_ADDRESS: ONLY for a single-use, temporary payment that will never recur
// req.setDestinationAccountType("ONE_TIME_ADDRESS");
```

**2-3. AML check is mandatory before every transfer.**
Use KYA screening via `ComplianceApiService` to screen the destination address. Block or alert on HIGH / SEVERE (and, by default, UNKNOWN) risk results before creating the transaction.

```java
// Required before creating any outbound transaction
ComplianceApiService complianceApi = ServiceCreator.create(ComplianceApiService.class, config);
KyaScreeningRequest kyaReq = new KyaScreeningRequest();
kyaReq.setAddress(destinationAddress);
kyaReq.setChainType("EVM");          // valid values: complianceApi.kyaSupportedNetworks()
kyaReq.setNetwork("Ethereum");
kyaReq.setProviders(Arrays.asList("Chainalysis"));
KyaScreeningCreateResponse created = ServiceExecutor.execute(complianceApi.kyaScreeningCreate(kyaReq));

KyaScreeningOneRequest pollReq = new KyaScreeningOneRequest();
pollReq.setScreenId(created.getScreenId());
KyaScreeningOneResponse result = null;
for (int i = 0; i < 30; i++) {
    result = ServiceExecutor.execute(complianceApi.kyaScreeningOne(pollReq));
    if ("FINISHED".equals(result.getStatus())) break;
    Thread.sleep(2000);
}
// Fail closed: an unfinished screening is NOT a pass
if (!"FINISHED".equals(result.getStatus())) {
    throw new SecurityException("KYA screening not finished for " + destinationAddress + ", route to manual review");
}
if (result.getOrders() == null || result.getOrders().isEmpty()) {
    throw new SecurityException("KYA screening returned no provider result for " + destinationAddress);
}
// Whitelist: only LOW / MEDIUM pass (adjust per your policy); HIGH, SEVERE, UNKNOWN, a missing level or FAILED are blocked
Set<String> allowedRiskLevels = Set.of("LOW", "MEDIUM");
for (KyaScreeningOneResponse.Order order : result.getOrders()) {
    if ("SKIPPED".equals(order.getStatus())) {
        continue;   // manually skipped in Web Console
    }
    if (!"COMPLETED".equals(order.getStatus())) {
        throw new SecurityException("KYA screening " + order.getStatus() + " (" + order.getProvider() + "), route to manual review");
    }
    if (order.getRiskLevel() == null || !allowedRiskLevels.contains(order.getRiskLevel())) {
        throw new SecurityException("AML check failed: " + destinationAddress + " is " + order.getRiskLevel() + " risk");
    }
}
```

**2-4. Validate address format before whitelist add or transfer.**
Call `CoinApiService.checkCoinAddress()` to verify address validity. Never trust user-supplied address strings directly.

```java
CoinApiService coinApi = ServiceCreator.create(CoinApiService.class, config);
CheckCoinAddressRequest checkReq = new CheckCoinAddressRequest();
checkReq.setCoinKey("ETHEREUM_ETH");
checkReq.setAddress(destinationAddress);
checkReq.setCheckAddressValid(true);   // without this flag the format is not checked
checkReq.setCheckAml(true);            // without this flag amlValid is always true
CheckCoinAddressResponse checkResp = ServiceExecutor.execute(coinApi.checkCoinAddress(checkReq));
if (!Boolean.TRUE.equals(checkResp.getAddressValid())) {
    throw new IllegalArgumentException("Invalid address format: " + destinationAddress);
}
if (!Boolean.TRUE.equals(checkResp.getAmlValid())) {
    throw new IllegalArgumentException("Address is on the AML blacklist: " + destinationAddress);
}
```

**2-5. Amounts must use `String` in API requests and `BigDecimal` in application logic.**
Never use `float` or `double` for monetary values — precision loss is guaranteed.

```java
// CORRECT
req.setAmount(new BigDecimal("0.001").toPlainString());

// WRONG — never use float/double for amounts
// req.setAmount(0.001);
// req.setAmount(0.001f);
```

---

## 3. Co-Signer Approval Callback

### Rules for Generated Code

**The Co-Signer callback service must never blindly approve anything.** Use `CoSignerCallbackController` in [COSIGNER.md](COSIGNER.md) as the REQUIRED pattern — do not write a reduced version. It enforces:

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
- The Co-Signer host must be in a **private isolated network** — never exposed to the public internet.
- The Approval Callback Service must only accept requests from the Co-Signer host IP, not from the public internet.
- Production API Keys configured for Co-Signer **must** have a Callback URL configured — never skip this in production.

---

## 4. Webhook Security

### Rules for Generated Code

**4-1. Verify signature before any processing.**
Every incoming Webhook payload must have its `sig` field verified using Safeheron's RSA public key. Reject events with invalid signatures immediately — do not process, do not respond.

**4-2. Webhook endpoint must use HTTPS.**
Never expose an HTTP Webhook endpoint in production.

**4-3. Idempotency is required.**
Failed deliveries are retried 6 times (30s → 1m → 5m → 1h → 12h → 24h), and duplicates can arrive anyway. The handler must safely process duplicate events — never double-credit a deposit or double-process a withdrawal. Commit the idempotency marker in the **same DB transaction** as the business side effects; never insert a dedup row and commit before doing the work.

**4-4. IP whitelist — only accept from Safeheron egress IPs.**
Configure firewall/VPC security groups to only allow inbound Webhook/callback traffic from (not listed in the published API reference — confirm the current list with Safeheron Support):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback — terminal states are final.**
If the current DB status is already a terminal state (`COMPLETED`, `FAILED`, `REJECTED`), discard any later-arriving Webhook event for the same transaction, regardless of its status.

**4-6. Handle out-of-order delivery.**
Webhooks may arrive out of order. If a terminal-state event was already processed, a subsequent intermediate-state event for the same transaction must be silently discarded (never applied).

```java
@PostMapping("/webhook")
public ResponseEntity<WebHookResponse> handleWebhook(@RequestBody WebHook webHook) {
    // 1. Verify signature + decrypt — convert() throws SafeheronException if the signature is invalid
    WebHookBizContent content;
    try {
        content = webhookConverter.convert(webHook);
    } catch (SafeheronException e) {
        log.error("Webhook signature verification failed", e);
        return ResponseEntity.status(401).build();
    } catch (Exception e) {
        log.error("Webhook decrypt/parse error — check your webhook RSA private key", e);
        return ResponseEntity.status(500).build();
    }

    // 2. Store the verified event durably (DB inbox with a unique key, or a durable MQ), then ack.
    //    In-memory queues / @Async lose events on a crash, and TOKEN_BLACKLIST_ADDRESS_HIT cannot be re-pushed
    //    by resend/failed. If storing throws, the 500 makes Safeheron retry (6 times).
    //    Body per docs: {"code":"200","message":"SUCCESS"}. Inbox pattern: see WEBHOOK.md Steps 2–3.
    try {
        webhookInbox.saveIfAbsent(eventKey(content), content.getEventType(), objectMapper.writeValueAsString(content));
    } catch (Exception e) {
        log.error("Failed to store webhook event", e);
        return ResponseEntity.status(500).build();
    }
    WebHookResponse ack = new WebHookResponse();
    ack.setCode("200");
    ack.setMessage("SUCCESS");
    return ResponseEntity.ok(ack);
}

// In the inbox worker (inside the same @Transactional method that marks the inbox row DONE):
void processWebhookEvent(TransactionParam event) {  // eventDetail of TRANSACTION_* events
    Transaction tx = transactionRepository.findByTxKey(event.getTxKey());
    if (tx == null) {
        tx = new Transaction();
    }

    // 4-5 & 4-6: Never roll back status; terminal state wins
    if (isTerminalStatus(tx.getStatus())) {
        log.info("Ignoring late event for terminal transaction: txKey={}, incomingStatus={}",
                event.getTxKey(), event.getTransactionStatus());
        return;
    }

    tx.setStatus(event.getTransactionStatus());
    transactionRepository.save(tx);
}

private boolean isTerminalStatus(String status) {
    return status != null &&
           (status.equals("COMPLETED") || status.equals("SIGN_COMPLETED") || status.equals("FAILED") || status.equals("REJECTED") || status.equals("CANCELLED"));
}
```

**4-7. Implement REST API polling as fallback.**
Do not rely solely on Webhooks for transaction state synchronization. Run a periodic polling job against the transaction list API to catch any missed events.

**4-8. Handle failed Webhook re-delivery.**
Call `/v1/webhook/resend/failed` (`resendFailed`) periodically to re-request delivery of events that failed to reach your server. Supported event types (per the published docs): `TRANSACTION_CREATED`, `TRANSACTION_STATUS_CHANGED`, `MPC_SIGN_CREATED`, `MPC_SIGN_STATUS_CHANGED`, `WEB3_SIGN_CREATED`, `WEB3_SIGN_STATUS_CHANGED`, `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, `WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`, `GAS_BALANCE_WARNING`, `AML_KYT_ALERT`. `TOKEN_BLACKLIST_ADDRESS_HIT` is **not** supported and cannot be re-pushed — so every event must still be stored durably before acking.

---

## 5. Policy Configuration Security

### Rules for Generated Code / Architecture Advice

- Configure tiered approval based on transaction amount (e.g., small amounts → Co-Signer auto-approve; large amounts → multi-person team approval).
- Always add a **catch-all blocking rule at the bottom** of the policy stack to intercept all transactions that don't match any defined rule.
- Subscribe to `NO_MATCHING_TRANSACTION_POLICY` Webhook event as an operational alert.
- After policy configuration, validate by testing with small amounts to confirm the approval flow works as expected.
- Perform regular security audits of policy rules.

---

## 6. API Key Security

- Scope API Key permissions to the **minimum required** for each use case. An API Key for withdrawals must not also have account management permission.
- Register only the minimum necessary server IPs in the IP whitelist. Remove stale IPs promptly.
- Non-essential personnel must not have access to RSA private keys.
- Rotate API Keys periodically and immediately upon suspected compromise.

---

## Quick Reference

| Area | Key Rule |
|---|---|
| Private keys | Vault/KMS only; never plaintext |
| Transfer addresses | Whitelist required; ONE_TIME_ADDRESS only for truly one-off payments |
| AML check | Mandatory before every outbound transfer via ComplianceApiService KYA screening |
| Address validation | Mandatory before whitelist add or transfer via CoinApiService.checkCoinAddress() |
| Amounts | String in API, BigDecimal in code; never float/double |
| Co-Signer callback | Use `CoSignerCallbackController` (COSIGNER.md): rules per transactionType (order + coin + address + amount + memo for NORMAL), AML/KYT whitelist, default REJECT |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling + /v1/transactions/one |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |