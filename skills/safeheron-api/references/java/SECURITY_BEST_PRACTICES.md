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
Call `ToolsApiService` to screen the destination address. Intercept or alert on high-risk addresses before creating the transaction.

```java
// Required before creating any outbound transaction
// Step 1: Submit AML check request
ToolsApiService toolsApi = ServiceCreator.create(ToolsApiService.class, config);
AmlCheckerRequestRequest amlReq = new AmlCheckerRequestRequest();
amlReq.setNetwork("Ethereum");   // "Bitcoin" | "Ethereum" | "Tron"
amlReq.setAddress(destinationAddress);
AmlCheckerRequestResponse amlRequestResp = ServiceExecutor.execute(toolsApi.amlCheckerRequest(amlReq));
String requestId = amlRequestResp.getRequestId();

// Step 2: Retrieve result (poll until status is ready)
AmlCheckerRetrievesRequest retrievesReq = new AmlCheckerRetrievesRequest();
retrievesReq.setRequestId(requestId);
AmlCheckerRetrievesResponse amlResp = ServiceExecutor.execute(toolsApi.amlCheckerRetrieves(retrievesReq));
if (Boolean.TRUE.equals(amlResp.getIsMaliciousAddress())) {
    throw new SecurityException("AML check failed for address: " + destinationAddress);
}
```

**2-4. Validate address format before whitelist add or transfer.**
Call `CoinApiService.checkCoinAddress()` to verify address validity. Never trust user-supplied address strings directly.

```java
CoinApiService coinApi = ServiceCreator.create(CoinApiService.class, config);
CheckCoinAddressRequest checkReq = new CheckCoinAddressRequest();
checkReq.setCoinKey("ETH");
checkReq.setAddress(destinationAddress);
CheckCoinAddressResponse checkResp = ServiceExecutor.execute(coinApi.checkCoinAddress(checkReq));
if (!checkResp.getAddressValid()) {
    throw new IllegalArgumentException("Invalid address format: " + destinationAddress);
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

**The Co-Signer callback service must never blindly approve all transactions.**
Every approval callback must implement the following business validations before returning `APPROVE`:

| Validation | What to Check |
|---|---|
| `customerRefId` | Must match a real pending business order in your system |
| Amount | Must exactly match the amount recorded in your business system for that order |
| Destination address | Must exactly match the address recorded in your business system for that order |

```java
// REQUIRED pattern for Co-Signer approval callback
@PostMapping("/cosigner/callback")
public ApprovalResponse handleCallback(@RequestBody CallbackRequest request) {
    // 1. Decrypt and verify signature (always first)
    CallbackPayload payload = decryptAndVerify(request);

    // 2. Look up the business order
    BusinessOrder order = orderService.findByCustomerRefId(payload.getCustomerRefId());
    if (order == null) {
        log.warn("Unknown customerRefId: {}", payload.getCustomerRefId());
        return ApprovalResponse.reject("Unknown order");
    }

    // 3. Validate amount matches business system
    if (!order.getAmount().equals(new BigDecimal(payload.getAmount()))) {
        log.warn("Amount mismatch: expected={}, actual={}", order.getAmount(), payload.getAmount());
        return ApprovalResponse.reject("Amount mismatch");
    }

    // 4. Validate destination address matches business system
    if (!order.getDestinationAddress().equalsIgnoreCase(payload.getDestinationAddress())) {
        log.warn("Address mismatch: expected={}, actual={}", order.getDestinationAddress(), payload.getDestinationAddress());
        return ApprovalResponse.reject("Address mismatch");
    }

    return ApprovalResponse.approve();
}
```

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
Safeheron retries delivery up to 7 times (30s → 1m → 5m → 1h → 12h → 24h). The handler must safely process duplicate events — never double-credit a deposit or double-process a withdrawal.

**4-4. IP whitelist — only accept from Safeheron egress IPs.**
Configure firewall/VPC security groups to only allow inbound Webhook/callback traffic from:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

**4-5. No status rollback — terminal states are final.**
If the current DB status is already a terminal state (`COMPLETED`, `FAILED`, `REJECTED`), discard any later-arriving Webhook event for the same transaction, regardless of its status.

**4-6. Handle out-of-order delivery.**
Webhooks may arrive out of order. If a terminal-state event was already processed, a subsequent intermediate-state event for the same transaction must be silently discarded (never applied).

```java
@PostMapping("/webhook")
public ResponseEntity<String> handleWebhook(@RequestBody WebhookRequest request) {
    // 1. Verify signature — reject if invalid
    if (!webhookVerifier.verify(request)) {
        log.warn("Webhook signature verification failed");
        return ResponseEntity.status(401).body("Signature verification failed");
    }

    // 2. Return 200 immediately; process asynchronously
    eventQueue.publish(request);
    return ResponseEntity.ok("OK");
}

// In async processor:
void processWebhookEvent(WebhookEvent event) {
    Transaction tx = transactionRepository.findByTxKey(event.getTxKey());
    if (tx == null) {
        tx = new Transaction();
    }

    // 4-5 & 4-6: Never roll back status; terminal state wins
    if (isTerminalStatus(tx.getStatus())) {
        log.info("Ignoring late event for terminal transaction: txKey={}, incomingStatus={}",
                event.getTxKey(), event.getStatus());
        return;
    }

    tx.setStatus(event.getStatus());
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
Call `/v1/transactions/one` periodically to re-request delivery of events that failed to reach your server.

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
| AML check | Mandatory before every outbound transfer via ToolsApiService |
| Address validation | Mandatory before whitelist add or transfer via CoinApiService.checkCoinAddress() |
| Amounts | String in API, BigDecimal in code; never float/double |
| Co-Signer callback | Validate customerRefId + amount + address against business system |
| Webhook | HTTPS only; verify signature; idempotent; IP whitelist |
| Webhook ordering | Terminal state wins; discard late intermediate-state events |
| Webhook fallback | REST API polling + /v1/transactions/one |
| Policy | Tiered approval + catch-all blocking rule at bottom |
| Co-Signer host | Private network only; never public internet |