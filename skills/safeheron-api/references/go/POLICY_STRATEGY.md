# Policy Strategy Configuration Guide

Policies control the approval flow for every transaction. **Every transaction must match exactly one policy** -- an unmatched transaction triggers the `NO_MATCHING_TRANSACTION_POLICY` webhook event and will not proceed.

Configure policies in: **Safeheron Web Console -> Manage -> Policy Engine**
Each new policy requires approval by team admins in the Safeheron App before it takes effect.

---

## Core Principle: Least Privilege

> Policies should follow the **minimum control principle**: each policy must be scoped to the narrowest possible source wallet, destination, initiator, and amount range. Never create a policy broader than required.

---

## 1. Pre-Configuration: Define Approval Nodes

| Node Name | Approvers | Threshold |
|-----------|-----------|-----------|
| Auto-Approval | API Co-Signer | 1-of-1 |
| Ops Team | Ops1, Ops2, Ops3 | 2-of-3 |
| Finance Team | Finance1, Finance2 | 2-of-2 |
| Executive | Boss1, Boss2 | 1-of-2 |
| Internal Transfer | All members (or designated ops) | 1-of-N |

---

## 2. Scenario: User Withdrawal Policy

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| API Key (withdrawal) | Hot withdrawal wallet | Any | Single tx: 0-100K | Auto-Approval |
| API Key (withdrawal) | Hot withdrawal wallet | Any | Single tx: >100K | Ops Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 0-100K | Auto-Approval |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 100K-500K | Ops Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 500K-2M | Finance Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: >2M | Executive |

---

## 3. Scenario: Internal Asset Transfer Policy

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| Designated Ops | All wallets | All team wallets (VAULT_ACCOUNT only) | Any amount | Internal Transfer |

> Destination must be restricted to `VAULT_ACCOUNT` type to prevent accidental external transfers.

---

## 4. Scenario: Manual External Transfer Policy

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| Designated Ops | All wallets | All whitelist addresses only | Any amount | Finance Team + Executive |

**Always restrict manual external transfers to whitelist destinations.** See [WHITELIST_API.md](WHITELIST_API.md).

---

## 5. Scenario: API Co-Signer -- Approval Callback Logic

When the API Co-Signer is the approver, it calls your **Approval Callback Service**:

```go
func callbackHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    defer r.Body.Close()

    var callbackPayload cosigner.CoSignerCallBackV3
    json.Unmarshal(body, &callbackPayload)

    // 1. Verify signature using Co-Signer identity public key
    bizContent, err := coSignerConverter.RequestV3Convert(callbackPayload)
    if err != nil {
        log.Printf("Co-Signer verification failed: %v", err)
        http.Error(w, "Verification failed", http.StatusForbidden)
        return
    }

    var content map[string]interface{}
    json.Unmarshal([]byte(bizContent), &content)

    // 2. Verify customerRefId exists in your DB
    customerRefId, _ := content["customerRefId"].(string)
    order := findOrderByCustomerRefId(customerRefId)
    if order == nil {
        log.Printf("REJECT: unknown customerRefId %s", customerRefId)
        respondReject(w, "Unknown transaction")
        return
    }

    // 3. Verify amount matches
    txAmount, _ := decimal.NewFromString(content["txAmount"].(string))
    if !txAmount.Equal(order.ExpectedAmount) {
        log.Printf("REJECT: amount mismatch for %s", customerRefId)
        respondReject(w, "Amount mismatch")
        return
    }

    // 4. Verify destination address matches
    destAddress, _ := content["destinationAddress"].(string)
    if destAddress != order.DestinationAddress {
        log.Printf("REJECT: destination mismatch for %s", customerRefId)
        respondReject(w, "Destination mismatch")
        return
    }

    // 5. Check AML risk
    if amlList, ok := content["amlList"].([]interface{}); ok {
        for _, aml := range amlList {
            amlMap, _ := aml.(map[string]interface{})
            if riskLevel, _ := amlMap["riskLevel"].(string); riskLevel == "HIGH" {
                respondReject(w, "High AML risk")
                return
            }
        }
    }

    // 6. Check amount within auto-approval limit
    autoApprovalLimit, _ := decimal.NewFromString("100000")
    if txAmount.GreaterThan(autoApprovalLimit) {
        respondReject(w, "Exceeds auto-approval limit")
        return
    }

    // 7. All checks passed
    respondApprove(w, customerRefId)
}
```

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use | Transaction blocked by policy |
| Withdrawal amount exceeds per-tx auto-approval threshold | Escalated to Ops Team |
| 24H cumulative total exceeds limit | Escalated to higher approval tier |
| Transaction with no matching policy | Blocked; `NO_MATCHING_TRANSACTION_POLICY` webhook fires |

---

## 7. Subscribe to `NO_MATCHING_TRANSACTION_POLICY` Event

```go
func processWebhookEvent(event map[string]interface{}) {
    eventType, _ := event["eventType"].(string)
    if eventType == "NO_MATCHING_TRANSACTION_POLICY" {
        alertOpsTeam("Transaction blocked -- no matching policy", event)
    }
}
```

**Treat this as a severity-1 alert in your monitoring system.**

---

## 8. Checklist: Policy Configuration Review

Before go-live:

- [ ] All transaction types/scenarios are covered by at least one policy
- [ ] No policy is broader than necessary
- [ ] Manual external transfers require whitelist destination
- [ ] Large amounts require multi-person, multi-layer approval
- [ ] API Co-Signer callback validates `customerRefId`, amount, and destination
- [ ] `NO_MATCHING_TRANSACTION_POLICY` webhook wired to ops alert
- [ ] Attack simulation tests passed
- [ ] Policy reviewed with Safeheron CSM
