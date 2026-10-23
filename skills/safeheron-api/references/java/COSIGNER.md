# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a service you deploy that **automatically approves or rejects transactions** based on your business logic. It replaces manual approval workflows for programmatic use cases and is required for unattended operation in production environments.

When a transaction requires approval, Safeheron calls your **Approval Callback Service** (an HTTP endpoint you implement) with an encrypted callback payload. Your service evaluates the transaction and responds `APPROVE` or `REJECT` (or any other value to keep it pending, e.g. while KYT screening is still running).

---

## How It Works

```
1. A transaction / MPC sign / Web3 sign is created (via API, Safeheron App or Web Console)
2. Safeheron: transaction enters SUBMITTED status
3. API Co-Signer polls Safeheron (every 5s in v2.x, every 1s in v1.x)
4. API Co-Signer calls your Approval Callback Service (HTTP POST)
5. Your service returns APPROVE or REJECT (any other value → Co-Signer retries every 5s)
6. Transaction proceeds to SIGNING or is REJECTED
```

**Approval Callback timeout:** Response timestamp must be within **5 seconds** of the API Co-Signer server's current time.

---

## Approval Callback Request

Safeheron sends an **encrypted HTTP POST** to your callback URL.

```json
{
  "timestamp": "1623038312088",
  "sig":        "<signed-with-cosigner-identity-private-key-base64>",
  "bizContent": "<callback-payload-base64>",
  "version":    "v3"
}
```

### Key Difference from Regular Webhooks

- The callback `sig` is signed with the **API Co-Signer's identity private key** (not the Safeheron platform key).
- You verify the signature using the **Co-Signer's exported identity public key**:
  ```bash
  sudo ./cosigner export-public-key
  ```
- The API Co-Signer, acting as the client of the Approval Callback Service, sends a POST request and signs the request data using its private key.
- Upon receiving the request, the Approval Callback Service uses the API Co-Signer's public key to authenticate the request data, ensuring that the request originates from the API Co-Signer.
---

## Approval Callback Service (Spring Boot Implementation)

### Configure CoSignerConverter

```java
import com.safeheron.client.cosigner.CoSignerConverter;
import com.safeheron.client.cosigner.CoSignerBizContentV3;
import com.safeheron.client.cosigner.CoSignerCallBackV3;
import com.safeheron.client.cosigner.CoSignerResponseV3;

@Configuration
public class CoSignerConfig {
    @Bean
    public CoSignerConverter coSignerConverter() {
        return new CoSignerConverter("YOUR_COSIGNER_PUBLIC_KEY","YOUR_CALLBACK_PRIVATE_KEY_BASE64");
    }
}
```

### Spring Boot Approval Callback Controller (V3 — Recommended)

> Placeholders you must implement: `OrderService` / `BusinessOrder` (your order store — `getCoinKey()` and `getOutputs()`, the expected destinations as `OrderOutput` with `getAddress()`, `getAmount()` as a decimal string and `getMemo()`), `matchesFuelPolicy`, `matchesSweepPolicy`, `matchesGasDepositPolicy`, `matchesMpcSignRequest`, `matchesWeb3SignRequest`.

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.safeheron.client.cosigner.CoSignerBizContentV3;
import com.safeheron.client.cosigner.CoSignerCallBackV3;
import com.safeheron.client.cosigner.CoSignerConverter;
import com.safeheron.client.cosigner.CoSignerResponseV3;
import com.safeheron.client.cosigner.MPCSignApproval;
import com.safeheron.client.cosigner.TransactionApproval;
import com.safeheron.client.cosigner.Web3SignApproval;
import com.safeheron.client.request.DestinationAddress;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collections;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@RestController
public class CoSignerCallbackController {

    // Current Co-Signer sends TRANSACTION / MPC_SIGN / WEB3_SIGN / CALLBACK_TEST;
    // the other values are legacy type names also recognised by the Safeheron SDK
    private static final Set<String> TX_TYPES = Set.of("TRANSACTION", "TX_TRANSACTION", "CONNECT_TX_SEND", "TX_CONNECT_TX_SEND", "TRANSACTION_BATCH_UTXO");
    private static final Set<String> WEB3_TYPES = Set.of("WEB3_SIGN", "ETH_SIGN", "PERSONAL_SIGN", "ETH_SIGN_TYPED_DATA", "ETH_SIGNTRANSACTION");

    private final CoSignerConverter coSignerConverter;
    private final ObjectMapper objectMapper;
    private final OrderService orderService;

    public CoSignerCallbackController(CoSignerConverter coSignerConverter, ObjectMapper objectMapper, OrderService orderService) {
        this.coSignerConverter = coSignerConverter;
        this.objectMapper = objectMapper;
        this.orderService = orderService;
    }

    @PostMapping("/cosigner/callback")
    public Map<String, String> handleCallback(@RequestBody CoSignerCallBackV3 encryptedBody) throws Exception {
        // 1. Decrypt and verify signature using Co-Signer identity public key
        CoSignerBizContentV3 bizContent = coSignerConverter.requestV3Convert(encryptedBody);
        // SDK <= 1.0.15 TransactionApproval does not model the top-level `memo` (TON / XRP Ledger) —
        // read it from the already signature-verified raw payload
        JsonNode rawDetail = objectMapper.readTree(Base64.getDecoder().decode(encryptedBody.getBizContent())).path("detail");
        String action = decide(bizContent, rawDetail.path("memo").isTextual() ? rawDetail.path("memo").asText() : null);

        CoSignerResponseV3 response = new CoSignerResponseV3();
        response.setApprovalId(bizContent.getApprovalId());
        response.setAction(action);
        return coSignerConverter.responseV3Converter(response);
    }

    // Default REJECT — return APPROVE only when every check passes
    private String decide(CoSignerBizContentV3 bizContent, String memo) {
        String callbackType = bizContent.getType();
        if (callbackType == null) {
            return "REJECT";
        }
        if ("CALLBACK_TEST".equals(callbackType)) {
            // Connectivity test sent by `cosigner callback-test` — detail is empty, no real transaction involved
            return "APPROVE";
        }
        if ("MPC_SIGN".equals(callbackType)) {
            // Verify txKey/customerRefId, sourceAccountKey and dataList against your pending MPC sign request
            MPCSignApproval mpc = objectMapper.convertValue(bizContent.getDetail(), MPCSignApproval.class);
            return matchesMpcSignRequest(mpc) ? "APPROVE" : "REJECT";
        }
        if (WEB3_TYPES.contains(callbackType)) {
            // SDK <= 1.0.15 does not convert the "WEB3_SIGN" detail — it stays a Map, so convert it yourself.
            // Branch on subjectType and verify message / messageHash / transaction against your pending request
            Web3SignApproval web3 = objectMapper.convertValue(bizContent.getDetail(), Web3SignApproval.class);
            return matchesWeb3SignRequest(web3) ? "APPROVE" : "REJECT";
        }
        if (!TX_TYPES.contains(callbackType)) {
            log.warn("REJECT: unknown callback type {}", callbackType);
            return "REJECT";
        }

        TransactionApproval tx = objectMapper.convertValue(bizContent.getDetail(), TransactionApproval.class);
        if (tx == null) {
            return "REJECT";
        }

        // 2. Business rules by transactionType (Safeheron "TRANSACTION Type Approval" best practices)
        if (!passesBusinessRules(tx, memo)) {
            log.warn("REJECT: business rules failed, txKey={}, transactionType={}", tx.getTxKey(), tx.getTransactionType());
            return "REJECT";
        }

        // 3. AML / KYT check (see "AML / KYT Handling in the Callback" below)
        AmlDecision amlDecision = evaluateAml(bizContent.getApprovalId(), tx);
        if (amlDecision == AmlDecision.WAIT) {
            // Neither APPROVE nor REJECT → Co-Signer re-sends this approval every 5s with the latest KYT report
            return ACTION_WAIT;
        }
        return amlDecision == AmlDecision.PASS ? "APPROVE" : "REJECT";
    }

    // Per Safeheron "TRANSACTION Type Approval" best practices; any other subtype is rejected
    private boolean passesBusinessRules(TransactionApproval tx, String memo) {
        String transactionType = tx.getTransactionType() == null ? "" : tx.getTransactionType();
        switch (transactionType) {
            case "NORMAL":               // API-created, or manually initiated in App / Web Console —
                                         // a manual one has no business order and is rejected here
            case "UTXO_BATCH_TRANSFER":
                return matchesBusinessOrder(tx, memo);
            case "UTXO_COLLECTION":      // API or manual sweep — destination must be your own vault account
                return "VAULT_ACCOUNT".equals(tx.getDestinationAccountType());
            case "AUTO_FUEL":            // system-initiated by Auto Sweep, no business order
                return "VAULT_ACCOUNT".equals(tx.getDestinationAccountType()) && matchesFuelPolicy(tx);
            case "AUTO_SWEEP":           // system-initiated, no business order: destination + USD amount per your policy
                return matchesSweepPolicy(tx);
            case "GAS_SERVICE_DEPOSIT":  // confirm the deposit exists in your records; destination must be PROFILE_ACCOUNT
                return "PROFILE_ACCOUNT".equals(tx.getDestinationAccountType()) && matchesGasDepositPolicy(tx);
            default:
                return false;
        }
    }

    private boolean matchesBusinessOrder(TransactionApproval tx, String memo) {
        BusinessOrder order = orderService.findByCustomerRefId(tx.getCustomerRefId());
        if (order == null) {
            return false;
        }
        // Currency must match — same address + amount in a different coin is a different transaction
        if (tx.getCoinKey() == null || !tx.getCoinKey().equals(order.getCoinKey())) {
            return false;
        }
        // Compare every (address, amount, memo) — exactly one for NORMAL, several for UTXO_BATCH_TRANSFER.
        // The memo decides who is credited on TON / XRP Ledger
        List<String> actual = new ArrayList<>();
        for (DestinationAddress d : destinationsOf(tx, memo)) {
            actual.add(outputKey(d.getAddress(), d.getAmount(), d.getMemo()));
        }
        List<String> expected = new ArrayList<>();
        for (OrderOutput o : order.getOutputs()) {
            expected.add(outputKey(o.getAddress(), o.getAmount(), o.getMemo()));
        }
        if (actual.isEmpty() || actual.contains(null) || expected.contains(null)) {
            return false;   // missing or malformed address / amount
        }
        Collections.sort(actual);
        Collections.sort(expected);
        return actual.equals(expected);
    }

    // UTXO coins carry destinations in destinationAddressList; account-model coins use destinationAddress + txAmount
    private List<DestinationAddress> destinationsOf(TransactionApproval tx, String memo) {
        if (tx.getDestinationAddressList() != null && !tx.getDestinationAddressList().isEmpty()) {
            return tx.getDestinationAddressList();
        }
        if (tx.getDestinationAddress() == null) {
            return List.of();
        }
        DestinationAddress single = new DestinationAddress();
        single.setAddress(tx.getDestinationAddress());
        single.setAmount(tx.getTxAmount());
        single.setMemo(memo);
        return List.of(single);
    }

    private String outputKey(String address, String amount, String memo) {
        if (address == null || address.isEmpty() || amount == null) {
            return null;
        }
        try {
            // Lower-casing suits EVM (hex) addresses only — compare case-sensitively for base58 / XRP Ledger / TRON / Solana
            return address.toLowerCase(Locale.ROOT) + "|" + new BigDecimal(amount).stripTrailingZeros().toPlainString()
                    + "|" + (memo == null ? "" : memo);
        } catch (NumberFormatException e) {
            return null;
        }
    }

    // ---- AML / KYT ----

    enum AmlDecision { PASS, WAIT, REJECT }

    // Source: Safeheron Compliance docs → Compliance Best Practices → KYT Status Handling Guide (PENDING):
    // "Approval Callback Service returns a non-APPROVE / non-REJECT value ... API Co-Signer retries approval
    // every 5 seconds with the latest KYT report". The literal "WAIT" is our choice of such a value.
    private static final String ACTION_WAIT = "WAIT";
    private static final long AML_WAIT_TIMEOUT_MS = 5 * 60 * 1000L;
    // Whitelist: only these risk levels may pass; HIGH, SEVERE, UNKNOWN and any unexpected value are rejected
    private static final Set<String> ALLOWED_RISK_LEVELS = Set.of("LOW", "MEDIUM");
    // true: reject when the callback carries no KYT data (Co-Signer < 2.1.1). Set false only if you do not use KYT
    private static final boolean KYT_REQUIRED = true;

    // When this service first saw each pending approval. Multi-instance deployments: use Redis SET NX with a TTL
    private final ConcurrentHashMap<String, Long> amlWaitStartedAt = new ConcurrentHashMap<>();

    private AmlDecision evaluateAml(String approvalId, TransactionApproval tx) {
        AmlDecision decision = evaluateAmlState(tx);
        if (decision != AmlDecision.WAIT) {
            amlWaitStartedAt.remove(approvalId);
            return decision;
        }
        long startedAt = amlWaitStartedAt.computeIfAbsent(approvalId, k -> System.currentTimeMillis());
        if (System.currentTimeMillis() - startedAt > AML_WAIT_TIMEOUT_MS) {
            amlWaitStartedAt.remove(approvalId);
            log.error("KYT screening not finished after timeout, txKey={}", tx.getTxKey());
            return AmlDecision.REJECT;   // or route to manual review
        }
        return AmlDecision.WAIT;
    }

    // Whitelist style: anything not explicitly recognised as safe is rejected
    private AmlDecision evaluateAmlState(TransactionApproval tx) {
        String state = tx.getAmlScreeningTriggeredState();
        if (state == null) {
            log.warn("No KYT data in approval callback (Co-Signer < 2.1.1?), txKey={}", tx.getTxKey());
            return KYT_REQUIRED ? AmlDecision.REJECT : AmlDecision.PASS;
        }
        switch (state) {
            case "UNTRIGGERED":
                return AmlDecision.PASS;
            case "IN_PROGRESS":
                return AmlDecision.WAIT;
            case "TRIGGERED":
                break;
            default:
                log.error("Unknown amlScreeningTriggeredState {}, txKey={}", state, tx.getTxKey());
                return AmlDecision.REJECT;
        }
        List<TransactionApproval.Aml> amlList = tx.getAmlList();
        if (amlList == null || amlList.isEmpty()) {
            log.error("KYT triggered but amlList is empty — set CALLBACK_INCLUDE_KYT_REPORT=true, txKey={}", tx.getTxKey());
            return AmlDecision.REJECT;
        }
        boolean pending = false;
        for (TransactionApproval.Aml aml : amlList) {
            String status = aml.getStatus() == null ? "" : aml.getStatus().trim().toUpperCase(Locale.ROOT);
            if ("PENDING".equals(status)) {
                pending = true;
            } else if ("COMPLETED".equals(status)) {
                // A completed report without a risk level is treated as UNKNOWN
                String riskLevel = aml.getRiskLevel() == null ? "UNKNOWN" : aml.getRiskLevel().trim().toUpperCase(Locale.ROOT);
                if (!ALLOWED_RISK_LEVELS.contains(riskLevel)) {
                    log.warn("KYT risk {} from {}, txKey={}", riskLevel, aml.getProvider(), tx.getTxKey());
                    return AmlDecision.REJECT;
                }
            } else if (!"SKIPPED".equals(status)) {
                // FAILED, missing or unknown status → manual handling
                log.error("KYT status {} from {}, txKey={}", aml.getStatus(), aml.getProvider(), tx.getTxKey());
                return AmlDecision.REJECT;
            }
        }
        return pending ? AmlDecision.WAIT : AmlDecision.PASS;
    }
}
```

---

## AML / KYT Handling in the Callback

**Prerequisites:**
- `amlScreeningTriggeredState` — included from API Co-Signer **2.1.1**.
- `amlList` — additionally requires `CALLBACK_INCLUDE_KYT_REPORT=true` (restart required). Without it `amlList` is absent — an empty list does **not** mean "no risk".

Follow Safeheron's KYT status handling guide — **never treat "not finished" or "unknown" as "safe"**:

| Condition | Decision |
|-----------|----------|
| `amlScreeningTriggeredState` missing | Co-Signer < 2.1.1 — reject if your policy requires KYT (`KYT_REQUIRED`), otherwise pass |
| `UNTRIGGERED` | No screening for this transaction — AML check passes |
| `IN_PROGRESS`, or any `amlList[].status = PENDING` | **Wait**: return neither APPROVE nor REJECT; Co-Signer retries every 5s. Measure the wait from the **first callback you received** for this `approvalId` (not `createTime`); after ~5 min (business-dependent) fall back to REJECT / manual review |
| `TRIGGERED` but `amlList` empty | `CALLBACK_INCLUDE_KYT_REPORT` is off — reject and alert |
| any `status = FAILED`, or a missing / unknown status or state | Manual handling — reject and alert |
| `status = SKIPPED` | Manually skipped in Web Console — passes |
| `status = COMPLETED` | Act on `riskLevel`: `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` (report obtained, no level; a missing level is treated the same). With multiple providers, screening is complete only when **all** are COMPLETED |

> `riskLevel` handling is your own compliance policy. The example only lets `LOW` and `MEDIUM` pass (so `HIGH`, `SEVERE`, `UNKNOWN` and any unexpected value are rejected); adjust `ALLOWED_RISK_LEVELS` per your jurisdiction.

---

## Decrypted Callback Payload Structure

After decryption, `bizContent` is a JSON object:

```json
{
  "approvalId": "approval_xxxxxx",
  "type": "TRANSACTION",
  "detail": { ... }
}
```

### Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Transaction approval — branch further on `detail.transactionType` |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval — branch further on `detail.subjectType` (`ETH_SIGN` / `PERSONAL_SIGN` / `ETH_SIGN_TYPED_DATA` / `ETH_SIGNTRANSACTION`) |
| `CALLBACK_TEST` | Connectivity test from the `cosigner callback-test` CLI; `detail` is empty |

> Legacy values `TX_TRANSACTION`, `CONNECT_TX_SEND`, `TX_CONNECT_TX_SEND`, `TRANSACTION_BATCH_UTXO` (transaction) and `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGN_TYPED_DATA`, `ETH_SIGNTRANSACTION` (Web3) are also recognised by the Safeheron SDK — handle them in the same groups.

---

## TRANSACTION Payload (`detail`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `txHash` | String | Transaction hash (if available) |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `transactionType` | String | Transaction subtype: `NORMAL` / `UTXO_COLLECTION` / `AUTO_SWEEP` / `AUTO_FUEL` / `UTXO_BATCH_TRANSFER` / `GAS_SERVICE_DEPOSIT` — approve each with different rules |
| `transactionStatus` | String | Current status |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Sender wallet key |
| `sourceAccountType` | String | e.g. `VAULT_ACCOUNT` |
| `sourceAddress` | String | Sender address |
| `destinationAccountKey` | String | Recipient wallet key |
| `destinationAccountType` | String | e.g. `ONE_TIME_ADDRESS` |
| `destinationAddress` | String | Recipient address |
| `destinationAddressList` | List | Multi-dest address list |
| `estimateFee` | String | Estimated fee |
| `feeCoinKey` | String | Fee coin |
| `note` | String | Transaction note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |
| `amlScreeningTriggeredState` | String | `IN_PROGRESS` (not yet decided, amlList unavailable) / `TRIGGERED` / `UNTRIGGERED` (amlList empty) |
| `amlList` | List\<Aml\> | KYT results; only present when Co-Signer ≥ 2.1.1 and `CALLBACK_INCLUDE_KYT_REPORT=true`. Item fields: `provider` (`Elliptic` / `Chainalysis` / `MistTrack`), `status` (`PENDING` / `COMPLETED` / `SKIPPED` / `FAILED`), `riskLevel` (`LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN`), `timestamp`, `lastUpdateTime` |
| `createTime` | Long | Unix timestamp (ms) |

---

## Web3 Payload (type: WEB3_SIGN — branch on `subjectType`: ETH_SIGN / PERSONAL_SIGN / ETH_SIGN_TYPED_DATA / ETH_SIGNTRANSACTION)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Web3 sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGN_TYPED_DATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | String | Web3 wallet account key |
| `sourceAddress` | String | Signing address |
| `createTime` | Long | Unix timestamp (ms) |
| `note` | String | Note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `message` | Object | For personalSign/ethSignTypedData: `{chainId, data}` |
| `messageHash` | Object | For ethSign: `{chainId, data:[hashes]}` |
| `transaction` | Object | For ethSignTransaction: `{chainId, to, value, gasLimit, gasPrice, maxFeePerGas, maxPriorityFeePerGas, nonce, data}` |

---

## MPC_SIGN Payload (`detail`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | MPC sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `sourceAccountKey` | String | Wallet key |
| `createTime` | Long | Unix timestamp (ms) |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `signAlg` | String | `Secp256k1` or `Ed25519` |
| `hashs` | List | Hashes to sign: `[{note, hash}]` |
| `dataList` | List | Raw data list: `[{note, data}]` |

---

## Approval Callback Response

Your callback service must return a **signed response** signed with your Callback Private Key:

```json
{
  "action": "APPROVE",
  "approvalId": "tx*******"
}
```

| Field | Type | Values |
|-------|------|--------|
| `action` | String | `APPROVE` or `REJECT`. Any other value leaves the approval pending and Co-Signer retries every 5s (used to wait for KYT results — per the Compliance docs, KYT Status Handling Guide) |
| `approvalId` | String | Echo back the incoming `approvalId` |

---

## Approval Logic Examples

These are **extra rejection conditions** to add inside `decide()` of `CoSignerCallbackController` — never use them on their own as the approval decision (passing one of them must not mean APPROVE).

### Destination Must Be Whitelisted

```java
// In decide(), after passesBusinessRules(): if (!isWhitelistedDestination(tx)) return "REJECT";
private boolean isWhitelistedDestination(TransactionApproval tx) {
    Set<String> allowedAddresses = loadWhitelistedAddresses(tx.getCoinKey());
    return tx.getDestinationAddress() != null && allowedAddresses.contains(tx.getDestinationAddress());
}
```

### Amount Limit per Account

```java
// In decide(), after passesBusinessRules(): if (!isWithinAccountLimit(tx)) return "REJECT";
private boolean isWithinAccountLimit(TransactionApproval tx) {
    BigDecimal limit = getAccountLimit(tx.getSourceAccountKey(), tx.getCoinKey());  // limit in the coin's own unit
    if (limit == null || tx.getTxAmount() == null) {
        return false;
    }
    try {
        return new BigDecimal(tx.getTxAmount()).compareTo(limit) <= 0;
    } catch (NumberFormatException e) {
        return false;
    }
}
```

### AML Risk Check

Use `evaluateAml(approvalId, tx)` defined in `CoSignerCallbackController` above (rules: [AML / KYT Handling in the Callback](#aml--kyt-handling-in-the-callback)). Do not just loop over `amlList` checking `riskLevel` — that approves transactions whose screening is still `IN_PROGRESS` / `PENDING`, has `FAILED`, or whose `amlList` is missing.

---

## API Co-Signer Deployment Notes

| Topic | Detail                                                                  |
|-------|-------------------------------------------------------------------------|
| Polling interval | v2.x: every 5s; v1.x: every 1s                                          |
| Callback timeout | Response must arrive within 5s of Co-Signer server time                 |
| Production | Callback URL is **required** for production teams                       |
| Test environment | Callback URL is optional                                                |
| KMS support | AWS KMS, GCP KMS, Alibaba Cloud KMS                                    |
| CLI commands | `sudo ./cosigner start`, `sudo ./cosigner stop`, `sudo ./cosigner logs` |
| Export Co-Signer public key | `sudo ./cosigner export-public-key`                                     |
| Minimum server specs | Refer to Safeheron Console deployment guide                             |
| Config changes | Modify `.env` then run `stop` + `start` to reload                       |
| Update Callback URL | Web Console update takes effect within 5 min — no Co-Signer restart needed |
| IP whitelist | Add Co-Signer host IP to the API Key IP whitelist in Console            |

---

## Co-Signer Version History (recent)

| Version | Date | Changes |
|---------|------|---------|
| 2.1.5 | 2026-09-21 | Added support for the XRP chain; added `schemaType` field in the TEE signing structure to indicate the format of the data to be signed; upgraded dependencies to strengthen system security |
| 2.1.4 | 2026-07-14 | Fixed an issue where restarting an inactive Co-Signer caused it to be locked and unable to activate |
| 2.1.3 | 2026-05-10 | Adjusted MPC Sign structure to improve support for message signing initiated by the Message Signing Tool |
| 2.1.2 | 2026-04-21 | Fixed incorrect serialization of hex strings in personal sign; fixed ABI parsing failure caused by dependency version inconsistency; fixed validation failure when the target address is empty in contract deployment scenarios |
| 2.1.1 | 2026-04-08 | Added the AML screening trigger state field in callback notifications; upgraded underlying service framework to improve system stability |

> The 2.1.1 release note spells the field `amlScreeningTriggerState`; the callback field documented in the Approval Callback Service docs and modelled by the SDKs is `amlScreeningTriggeredState`. XRP chain support was added in Co-Signer 2.1.5. Verify the SHA-256 of the downloaded package against the Safeheron API Co-Signer docs.

---

## New vs. Old Co-Signer Version Differences

| Aspect | New Version (v2.x+) | Old Version (v1.x) |
|--------|--------------------|--------------------|
| RSA key pairs required | **1 pair** | 2 pairs |
| Polling interval | 5 seconds | 1 second |
| Cancel transaction in DB | Automatic | Manual SQL required |
| Config management | Secrets Manager (recommended) + .env | Config file |

**Old version** requires configuring the public keys `biz_pubkey` and `api_privkey` in two pairs of public and private keys in the configuration file. The new version only requires registering `biz_pubkey` in the Web console.

---

## CLI Commands Reference

```bash
sudo ./cosigner start               # Start Co-Signer
sudo ./cosigner stop                # Stop Co-Signer
sudo ./cosigner setup               # Initial setup (must run before start)
sudo ./cosigner export-public-key   # Export Co-Signer identity public key
./cosigner logs -f                  # Stream live logs
./cosigner logs -s                  # Export logs to file (for Support)
```

---

## Key Pair Configuration Summary

| Key | Owner | Purpose |
|-----|-------|---------|
| **Co-Signer Identity Private Key** | Safeheron (Co-Signer internal) | Signs callback requests sent to your service |
| **Co-Signer Identity Public Key** | Exported via `export-public-key` | You use this to verify callback request signatures |
| **Callback Public Key** | You generate & upload to Console | Safeheron Co-Signer needs to use this public key to verify the signature in the Callback's response data |
| **Callback Private Key** | You hold securely | Your callback service needs to use this private key to sign the Callback response body |

---

## Common Co-Signer Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Illegal IP` in logs | Co-Signer host IP not in API Key whitelist | Add IP to whitelist in Console, restart Co-Signer |
| `The final private key fragment is not exists` | Co-Signer not yet activated | Normal — proceed with activation workflow |
| Transaction stays in `SUBMITTED` | Co-Signer not running or not polling | Check `./cosigner logs -f` for errors |
| `Timestamp out of range` | Server clock skew > 5s | Sync Co-Signer server time with NTP |
| Docker login failure | Token expired or firewall blocks `registry.gitlab.com` | Re-download CLI from Console; check firewall rules |

---

## Official SDK Demo

See the SDK repository for a working callback demo:
https://github.com/Safeheron/safeheron-api-sdk-java

---

## Security Deployment Requirements (API Co-Signer Host)

These requirements apply to the server where API Co-Signer is deployed. Non-compliance can result in asset loss.

### Mandatory Security Principles

| Principle | Requirement |
|-----------|------------|
| Strong passwords | All accounts (DB, cloud, OS) must use randomly generated strong passwords — no weak/default credentials |
| MFA everywhere | Enable 2FA/MFA on every account and service that supports it |
| Minimum privilege | Each account/role has only the permissions needed for its specific function |
| Minimum exposure | Close all unnecessary ports. Only port `9999` (business) needs to be open |
| Secure secret storage | Store all secrets (passwords, private keys, Secret Keys) in a dedicated secrets manager (e.g., 1Password, AWS Secrets Manager). Never transmit secrets via chat/email/documents |

### Host Security Controls

| Control | Description |
|---------|-------------|
| Network isolation | Deploy Co-Signer in a **private isolated network**. No public internet inbound access |
| IP whitelist | Only your internal systems can reach the Co-Signer host |
| MFA (host login) | All host access requires MFA |
| CWPP | Cloud Workload Protection Platform — real-time threat monitoring on the Co-Signer instance |
| EDR/XDR | Endpoint/Extended Detection & Response — continuous threat detection and automated response |
| SIEM | Collect host login and system activity logs; configure real-time alerts |
| PAM | Privileged Access Management — audit all privileged operations on the host |

### Required Network Connectivity (Outbound Only)

API Co-Signer has **zero inbound** from the internet. Configure firewall to allow only these outbound connections:

**Always required (runtime):**
- `https://api.safeheron.vip:443`
- `wss://gm-gateway.safeheron.vip:443`
- MySQL (your DB host)
- Approval Callback Service (your internal service)

**If using AWS KMS:**
- `https://*.amazonaws.com:443`
- `https://*.amazonaws.com.cn:443`

**If using Alibaba Cloud KMS:**
- `https://*.cryptoservice.kms.aliyuncs.com:443`

**If using GCP Cloud KMS:**
- `https://*.cloudkms.googleapis.com:443`
- `https://cloudkms.googleapis.com:443`
- `http://metadata.google.internal:80`

**Only during install/update:**
- `registry.gitlab.com:443`
- `https://prod-safeheron-openapi.s3.ap-east-1.amazonaws.com:443`

**Only on first install (Docker bootstrap):**
- `https://get.docker.com:443`
- `https://download.docker.com:443`
- `https://api.github.com:443`
- `https://github.com:443`

### Callback Service Security

The Approval Callback Service must:

1. **Verify the Co-Signer identity signature** on every request before processing
2. **Only accept connections from the Co-Signer host IP** — configure firewall/IP whitelist on the callback service
3. **Apply the rules for each `transactionType`** — for `NORMAL` / `UTXO_BATCH_TRANSFER`, the `customerRefId` must match a pending order and coin, every destination address, amount and memo must match it; system subtypes (`AUTO_SWEEP`, `AUTO_FUEL`, `UTXO_COLLECTION`, `GAS_SERVICE_DEPOSIT`) follow their own rules — see `passesBusinessRules`
4. **Apply the AML / KYT rules** — see `evaluateAml`
5. **Respond within 5 seconds** — Co-Signer will time out otherwise (sync server time via NTP)
