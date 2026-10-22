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

When the API Co-Signer is the approver, it calls your **Approval Callback Service**. Start from the handler in [COSIGNER.md](COSIGNER.md) — it already handles callback types, per-`transactionType` business rules and AML/KYT — and add your policy tiers to its `decide()` function, **before** the AML check (so over-limit transactions are rejected immediately instead of waiting for KYT):

```go
// Replaces steps 1–2 at the end of the TRANSACTION case in decide() (do not keep both copies):

// 1. Business rules by transactionType (order matching for NORMAL / UTXO_BATCH_TRANSFER, system rules otherwise)
if !passesBusinessRules(tx) {
    return "REJECT"
}

// 2. Policy tier — defense in depth: your Safeheron policy should already route large amounts to human
//    approvers; if one still reaches the Co-Signer, reject it
if !isWithinAutoApprovalLimit(tx) {
    log.Printf("REJECT: amount %s %s exceeds auto-approval limit, txKey=%s", tx.TxAmount, tx.CoinKey, tx.TxKey)
    return "REJECT"
}

// 3. AML / KYT — only LOW / MEDIUM pass; waits (neither APPROVE nor REJECT) while screening is unfinished
switch evaluateAml(env.ApprovalId, tx) {
case AmlPass:
    return "APPROVE"
case AmlWait:
    return ActionWait
default:
    return "REJECT"
}
```

```go
// Package-level, same package as COSIGNER.md's handler (decimal is already imported there).
// Limits are per coin, in the coin's own unit — never compare raw amounts of different coins against one number
var autoApprovalLimits = map[string]decimal.Decimal{
    "ETHEREUM_ETH":              decimal.RequireFromString("10"),
    "USDT(ERC20)_ETHEREUM_USDT": decimal.RequireFromString("50000"),
}

func isWithinAutoApprovalLimit(tx TransactionApproval) bool {
    limit, ok := autoApprovalLimits[tx.CoinKey]
    if !ok {
        return false // unknown coin — fail closed
    }
    amount, err := decimal.NewFromString(tx.TxAmount)
    if err != nil {
        return false // missing or malformed amount — fail closed
    }
    return !amount.IsNegative() && amount.LessThanOrEqual(limit)
}
```

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use | Transaction blocked by policy |
| Withdrawal amount exceeds per-tx auto-approval threshold | Routed by policy to the Ops Team for manual review; the Co-Signer callback rejects it if it ever reaches it |
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
