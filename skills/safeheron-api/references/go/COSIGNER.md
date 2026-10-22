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

## Approval Callback Service (Go Implementation)

### Using the SDK's CoSignerConverter

```go
import (
    "encoding/json"
    "log"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/cosigner"
)

coSignerConverter := cosigner.CoSignerConverter{Config: cosigner.CoSignerConfig{
    CoSignerPubKey:                    "pems/cosigner_public.pem",
    ApprovalCallbackServicePrivateKey: "pems/callback_private.pem",
}}

// Decrypt and verify the callback request (returns JSON string)
bizContent, err := coSignerConverter.RequestV3Convert(callbackPayload)
if err != nil {
    log.Printf("Co-Signer verification failed: %v", err)
    return
}

// Parse the V3 envelope (bizContent is a JSON string); callbackEnvelope is defined in the full handler below
var env callbackEnvelope
if err := json.Unmarshal([]byte(bizContent), &env); err != nil {
    log.Printf("Failed to parse envelope: %v", err)
    return
}

// Build encrypted response — never hard-code APPROVE; decide() (full handler below) returns
// "APPROVE", "REJECT" or ActionWait
response := cosigner.CoSignerResponseV3{
    ApprovalId: env.ApprovalId,
    Action:     decide(env),
}
encryptedResp, err := coSignerConverter.ResponseV3Converter(response)
if err != nil {
    log.Printf("Failed to encrypt response: %v", err)
    return
}
// Return encryptedResp to the Co-Signer
```

### Full net/http Handler

> Placeholders you must implement, with exactly these signatures (anything else will not compile against the code below):
> - `type BusinessOrder struct { CoinKey string; Outputs []output }` — `Outputs` are the expected destinations (`output{Address, Amount, Memo}`, amounts as decimal strings)
> - `func findOrderByCustomerRefId(customerRefId string) *BusinessOrder` — `nil` when no pending order exists
> - `func matchesFuelPolicy(tx TransactionApproval) bool`, `func matchesSweepPolicy(tx TransactionApproval) bool`, `func matchesGasDepositPolicy(tx TransactionApproval) bool`
> - `func matchesMpcSignRequest(detail json.RawMessage) bool`, `func matchesWeb3SignRequest(detail json.RawMessage) bool`
>
> The AML code in the next section belongs to the same `package main`.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "sort"
    "strings"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/cosigner"
    "github.com/shopspring/decimal"
)

var coSignerConverter cosigner.CoSignerConverter

func init() {
    coSignerConverter = cosigner.CoSignerConverter{Config: cosigner.CoSignerConfig{
        CoSignerPubKey:                    "pems/cosigner_public.pem",
        ApprovalCallbackServicePrivateKey: "pems/callback_private.pem",
    }}
}

// V3 envelope after decryption: {approvalId, type, detail}
type callbackEnvelope struct {
    ApprovalId string          `json:"approvalId"`
    Type       string          `json:"type"`
    Detail     json.RawMessage `json:"detail"`
}

func callbackHandler(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    // Parse callback payload
    var callbackPayload cosigner.CoSignerCallBackV3
    if err := json.Unmarshal(body, &callbackPayload); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Decrypt and verify Co-Signer identity signature (returns the bizContent JSON string)
    bizContent, err := coSignerConverter.RequestV3Convert(callbackPayload)
    if err != nil {
        log.Printf("Co-Signer verification failed: %v", err)
        http.Error(w, "Verification failed", http.StatusForbidden)
        return
    }

    var env callbackEnvelope
    if err := json.Unmarshal([]byte(bizContent), &env); err != nil {
        log.Printf("Failed to parse V3 envelope: %v", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    // Build encrypted response
    response := cosigner.CoSignerResponseV3{
        ApprovalId: env.ApprovalId,
        Action:     decide(env),
    }
    encryptedResp, err := coSignerConverter.ResponseV3Converter(response)
    if err != nil {
        log.Printf("Failed to encrypt response: %v", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(encryptedResp)
}

// Default REJECT — return APPROVE only when every check passes.
// Current Co-Signer sends TRANSACTION / MPC_SIGN / WEB3_SIGN / CALLBACK_TEST;
// the other values are legacy type names also recognised by the Safeheron SDK
func decide(env callbackEnvelope) string {
    switch env.Type {
    case "CALLBACK_TEST":
        // Connectivity test sent by `cosigner callback-test` — detail is empty, no real transaction involved
        return "APPROVE"

    case "MPC_SIGN":
        // Verify txKey/customerRefId, sourceAccountKey and dataList against your pending MPC sign request
        if matchesMpcSignRequest(env.Detail) {
            return "APPROVE"
        }
        return "REJECT"

    case "WEB3_SIGN", "ETH_SIGN", "PERSONAL_SIGN", "ETH_SIGN_TYPED_DATA", "ETH_SIGNTRANSACTION":
        // Branch on detail.subjectType and verify message / messageHash / transaction against your pending request
        if matchesWeb3SignRequest(env.Detail) {
            return "APPROVE"
        }
        return "REJECT"

    case "TRANSACTION", "TX_TRANSACTION", "CONNECT_TX_SEND", "TX_CONNECT_TX_SEND", "TRANSACTION_BATCH_UTXO":
        var tx TransactionApproval
        if err := json.Unmarshal(env.Detail, &tx); err != nil {
            log.Printf("REJECT: failed to parse TRANSACTION detail: %v", err)
            return "REJECT"
        }

        // 1. Business rules by transactionType (Safeheron "TRANSACTION Type Approval" best practices)
        if !passesBusinessRules(tx) {
            log.Printf("REJECT: business rules failed, txKey=%s transactionType=%s", tx.TxKey, tx.TransactionType)
            return "REJECT"
        }

        // 2. AML / KYT check (see "AML / KYT Handling in the Callback" below)
        switch evaluateAml(env.ApprovalId, tx) {
        case AmlPass:
            return "APPROVE"
        case AmlWait:
            // Neither APPROVE nor REJECT → Co-Signer re-sends this approval every 5s with the latest KYT report
            return ActionWait
        default:
            return "REJECT"
        }

    default:
        log.Printf("REJECT: unknown callback type: %s", env.Type)
        return "REJECT"
    }
}

// TransactionApproval — fields of the TRANSACTION callback `detail` (the Go SDK has no struct for it)
type TransactionApproval struct {
    TxKey                  string `json:"txKey"`
    CustomerRefId          string `json:"customerRefId"`
    CoinKey                string `json:"coinKey"`
    TxAmount               string `json:"txAmount"`
    TransactionType        string `json:"transactionType"`
    DestinationAccountType string `json:"destinationAccountType"`
    DestinationAddress     string `json:"destinationAddress"`
    Memo                   string `json:"memo"` // destination memo (TON / XRP Ledger)
    // UTXO coins carry their destinations here instead of destinationAddress
    DestinationAddressList []struct {
        Address string `json:"address"`
        Amount  string `json:"amount"`
        Memo    string `json:"memo"`
    } `json:"destinationAddressList"`
    CreateTime                 int64  `json:"createTime"`
    AmlScreeningTriggeredState string `json:"amlScreeningTriggeredState"`
    AmlList                    []struct {
        Provider  string `json:"provider"`
        Status    string `json:"status"`
        RiskLevel string `json:"riskLevel"`
    } `json:"amlList"`
}

// Per Safeheron "TRANSACTION Type Approval" best practices; any other subtype is rejected
func passesBusinessRules(tx TransactionApproval) bool {
    switch tx.TransactionType {
    case "NORMAL", // API-created, or manually initiated in App / Web Console —
        // a manual one has no business order and is rejected here
        "UTXO_BATCH_TRANSFER":
        return matchesBusinessOrder(tx)
    case "UTXO_COLLECTION": // API or manual sweep — destination must be your own vault account
        return tx.DestinationAccountType == "VAULT_ACCOUNT"
    case "AUTO_FUEL": // system-initiated by Auto Sweep, no business order
        return tx.DestinationAccountType == "VAULT_ACCOUNT" && matchesFuelPolicy(tx)
    case "AUTO_SWEEP": // system-initiated, no business order: destination + USD amount per your policy
        return matchesSweepPolicy(tx)
    case "GAS_SERVICE_DEPOSIT": // confirm the deposit exists in your records; destination must be PROFILE_ACCOUNT
        return tx.DestinationAccountType == "PROFILE_ACCOUNT" && matchesGasDepositPolicy(tx)
    default:
        return false
    }
}

func matchesBusinessOrder(tx TransactionApproval) bool {
    order := findOrderByCustomerRefId(tx.CustomerRefId)
    if order == nil {
        return false
    }
    // Currency must match — same address + amount in a different coin is a different transaction
    if tx.CoinKey == "" || tx.CoinKey != order.CoinKey {
        return false
    }
    // Compare every (address, amount, memo) — exactly one for NORMAL, several for UTXO_BATCH_TRANSFER.
    // The memo decides who is credited on TON / XRP Ledger
    actual, ok := outputKeys(destinationsOf(tx))
    if !ok || len(actual) == 0 {
        return false
    }
    expected, ok := outputKeys(order.Outputs)
    if !ok || len(expected) != len(actual) {
        return false
    }
    for i := range actual {
        if actual[i] != expected[i] {
            return false
        }
    }
    return true
}

type output struct{ Address, Amount, Memo string }

// UTXO coins carry destinations in destinationAddressList; account-model coins use destinationAddress + txAmount
func destinationsOf(tx TransactionApproval) []output {
    if len(tx.DestinationAddressList) > 0 {
        outs := make([]output, 0, len(tx.DestinationAddressList))
        for _, d := range tx.DestinationAddressList {
            outs = append(outs, output{d.Address, d.Amount, d.Memo})
        }
        return outs
    }
    if tx.DestinationAddress == "" {
        return nil
    }
    return []output{{tx.DestinationAddress, tx.TxAmount, tx.Memo}}
}

// Sorted "address|amount|memo" keys; ok=false if any address or amount is empty or not a number
func outputKeys(outs []output) ([]string, bool) {
    keys := make([]string, 0, len(outs))
    for _, o := range outs {
        amount, err := decimal.NewFromString(o.Amount)
        if o.Address == "" || err != nil {
            return nil, false
        }
        // Lower-casing suits EVM (hex) addresses only — compare case-sensitively for base58 / XRP Ledger / TRON / Solana
        keys = append(keys, strings.ToLower(o.Address)+"|"+amount.String()+"|"+o.Memo)
    }
    sort.Strings(keys)
    return keys, true
}

func main() {
    http.HandleFunc("/cosigner/callback", callbackHandler)
    fmt.Println("Callback service listening on :9090")
    log.Fatal(http.ListenAndServe(":9090", nil))
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

## AML / KYT Handling in the Callback

**Prerequisites:**
- `amlScreeningTriggeredState` — included from API Co-Signer **2.1.1**.
- `amlList` — additionally requires `CALLBACK_INCLUDE_KYT_REPORT=true` (restart required). Without it `amlList` is absent — an empty list does **not** mean "no risk".

Follow Safeheron's KYT status handling guide — **never treat "not finished" or "unknown" as "safe"**:

| Condition | Decision |
|-----------|----------|
| `amlScreeningTriggeredState` missing | Co-Signer < 2.1.1 — reject if your policy requires KYT (`kytRequired`), otherwise pass |
| `UNTRIGGERED` | No screening for this transaction — AML check passes |
| `IN_PROGRESS`, or any `amlList[].status = PENDING` | **Wait**: return neither APPROVE nor REJECT; Co-Signer retries every 5s. Measure the wait from the **first callback you received** for this `approvalId` (not `createTime`); after ~5 min (business-dependent) fall back to REJECT / manual review |
| `TRIGGERED` but `amlList` empty | `CALLBACK_INCLUDE_KYT_REPORT` is off — reject and alert |
| any `status = FAILED`, or a missing / unknown status or state | Manual handling — reject and alert |
| `status = SKIPPED` | Manually skipped in Web Console — passes |
| `status = COMPLETED` | Act on `riskLevel`: `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` (report obtained, no level; a missing level is treated the same). With multiple providers, screening is complete only when **all** are COMPLETED |

> `riskLevel` handling is your own compliance policy. The example only lets `LOW` and `MEDIUM` pass (so `HIGH`, `SEVERE`, `UNKNOWN` and any unexpected value are rejected); adjust `allowedRiskLevels` per your jurisdiction.

Same `package main` as the handler above; add `"sync"` and `"time"` to its import block (`log` and `strings` are already imported).

```go
type AmlDecision int

const (
    AmlPass AmlDecision = iota
    AmlWait
    AmlReject
)

// Source: Safeheron API docs → Compliance → Compliance Best Practices → KYT Status Handling Guide (PENDING):
// "Approval Callback Service returns a non-APPROVE / non-REJECT value … API Co-Signer retries approval
// every 5 seconds with the latest KYT report". The literal "WAIT" is our choice of such a value.
const ActionWait = "WAIT"
const amlWaitTimeout = 5 * time.Minute

// true: reject when the callback carries no KYT data (Co-Signer < 2.1.1). Set false only if you do not use KYT
const kytRequired = true

// Whitelist: only these risk levels may pass; HIGH, SEVERE, UNKNOWN and any unexpected value are rejected
var allowedRiskLevels = map[string]bool{"LOW": true, "MEDIUM": true}

// When this service first saw each pending approval (approvalId → time.Time).
// Multi-instance deployments: use Redis SET NX with a TTL instead
var amlWaitStartedAt sync.Map

func evaluateAml(approvalId string, tx TransactionApproval) AmlDecision {
    decision := evaluateAmlState(tx)
    if decision != AmlWait {
        amlWaitStartedAt.Delete(approvalId)
        return decision
    }
    startedAt, _ := amlWaitStartedAt.LoadOrStore(approvalId, time.Now())
    if time.Since(startedAt.(time.Time)) > amlWaitTimeout {
        amlWaitStartedAt.Delete(approvalId)
        log.Printf("ERROR: KYT screening not finished after timeout, txKey=%s", tx.TxKey)
        return AmlReject // or route to manual review
    }
    return AmlWait
}

// Whitelist style: anything not explicitly recognised as safe is rejected
func evaluateAmlState(tx TransactionApproval) AmlDecision {
    switch tx.AmlScreeningTriggeredState {
    case "":
        log.Printf("WARN: no KYT data in approval callback (Co-Signer < 2.1.1?), txKey=%s", tx.TxKey)
        if kytRequired {
            return AmlReject
        }
        return AmlPass
    case "UNTRIGGERED":
        return AmlPass
    case "IN_PROGRESS":
        return AmlWait
    case "TRIGGERED":
        // checked below
    default:
        log.Printf("ERROR: unknown amlScreeningTriggeredState %s, txKey=%s", tx.AmlScreeningTriggeredState, tx.TxKey)
        return AmlReject
    }
    if len(tx.AmlList) == 0 {
        log.Printf("ERROR: KYT triggered but amlList is empty — set CALLBACK_INCLUDE_KYT_REPORT=true, txKey=%s", tx.TxKey)
        return AmlReject
    }
    pending := false
    for _, aml := range tx.AmlList {
        switch strings.ToUpper(strings.TrimSpace(aml.Status)) {
        case "PENDING":
            pending = true
        case "SKIPPED":
            // manually skipped — passes
        case "COMPLETED":
            // A completed report without a risk level is treated as UNKNOWN
            riskLevel := strings.ToUpper(strings.TrimSpace(aml.RiskLevel))
            if riskLevel == "" {
                riskLevel = "UNKNOWN"
            }
            if !allowedRiskLevels[riskLevel] {
                log.Printf("WARN: KYT risk %s from %s, txKey=%s", riskLevel, aml.Provider, tx.TxKey)
                return AmlReject
            }
        default:
            // FAILED, missing or unknown status → manual handling
            log.Printf("ERROR: KYT status %q from %s, txKey=%s", aml.Status, aml.Provider, tx.TxKey)
            return AmlReject
        }
    }
    if pending {
        return AmlWait
    }
    return AmlPass
}
```

### AML Risk Check

Use `evaluateAml(approvalId, tx)` above (rules: [AML / KYT Handling in the Callback](#aml--kyt-handling-in-the-callback)). Do not just loop over `amlList` checking `riskLevel` — that approves transactions whose screening is still `IN_PROGRESS` / `PENDING`, has `FAILED`, or whose `amlList` is missing.

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
| `action` | String | `APPROVE` or `REJECT`. Any other value leaves the approval pending and Co-Signer retries every 5s (used to wait for KYT results) |
| `approvalId` | String | Echo back the incoming `approvalId` |

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

## Co-Signer Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.1.5 | 2026-09-21 | Added support for the XRP chain; added `schemaType` field in the TEE signing structure to indicate the format of the data to be signed; upgraded dependencies to strengthen system security |
| 2.1.4 | 2026-07-14 | Fixed an issue where restarting an inactive Co-Signer caused it to be locked and unable to activate |
| 2.1.3 | 2026-05-10 | Adjusted MPC Sign structure to improve support for message signing initiated by the Message Signing Tool |
| 2.1.2 | 2026-04-21 | Fixed incorrect serialization of hex strings in personal sign; fixed ABI parsing failure caused by dependency version inconsistency; fixed validation failure when the target address is empty in contract deployment scenarios |
| 2.1.1 | 2026-04-08 | Added the AML screening trigger state field in callback notifications; upgraded underlying service framework to improve system stability |

> The 2.1.1 release note spells the field `amlScreeningTriggerState`; the callback field documented in the Approval Callback Service (`TransactionApproval`) is `amlScreeningTriggeredState`. Use 2.1.1+ for KYT data in callbacks (see [AML / KYT Handling in the Callback](#aml--kyt-handling-in-the-callback)). XRP chain support was added in Co-Signer 2.1.5.

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
https://github.com/Safeheron/safeheron-api-sdk-go

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
