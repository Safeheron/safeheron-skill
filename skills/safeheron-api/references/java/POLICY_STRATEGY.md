# Policy Strategy Configuration Guide

Policies control the approval flow for every transaction. **Every transaction must match exactly one policy** — an unmatched transaction triggers the `NO_MATCHING_TRANSACTION_POLICY` webhook event and will not proceed.

Configure policies in: **Safeheron Web Console → Manage → Policy Engine**
Each new policy requires approval by team admins in the Safeheron App before it takes effect.

---

## Core Principle: Least Privilege

> Policies should follow the **minimum control principle**: each policy must be scoped to the narrowest possible source wallet, destination, initiator, and amount range. Never create a policy broader than required. An over-broad policy can allow unintended fund movement.

---

## 1. Pre-Configuration: Define Approval Nodes

Before configuring scenario policies, define the approval nodes (人员组) you will reference:

| Node Name | Approvers | Threshold |
|-----------|-----------|-----------|
| Auto-Approval | API Co-Signer | 1-of-1 |
| Ops Team | Ops1, Ops2, Ops3 | 2-of-3 |
| Finance Team | Finance1, Finance2 | 2-of-2 |
| Executive | Boss1, Boss2 | 1-of-2 |
| Internal Transfer | All members (or designated ops) | 1-of-N |

---

## 2. Scenario: User Withdrawal Policy

**Goal:** API Key can only initiate transactions from withdrawal hot wallets. Amount tiers determine approval level.

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| API Key (withdrawal) | Hot withdrawal wallet | Any | Single tx: 0–100K | Auto-Approval |
| API Key (withdrawal) | Hot withdrawal wallet | Any | Single tx: >100K | Ops Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 0–100K | Auto-Approval |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 100K–500K | Ops Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: 500K–2M | Finance Team |
| API Key (withdrawal) | Hot withdrawal wallet | Any | 24H cumulative: >2M | Executive |

**Why separate single-tx and 24H limits?** A single small withdrawal may be auto-approved, but if many are issued within 24H and the cumulative total exceeds the safe threshold, escalation is triggered automatically.

---

## 3. Scenario: Internal Asset Transfer Policy

**Goal:** Internal consolidation/scheduling between team wallets. Assets never leave Safeheron. Approval can be relaxed since there's no external exposure risk.

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| Designated Ops | All wallets | All team wallets (VAULT_ACCOUNT only) | Any amount | Internal Transfer |

> **Note:** Destination must be restricted to `VAULT_ACCOUNT` type — this ensures funds cannot accidentally leave the team boundary.

---

## 4. Scenario: Manual External Transfer Policy

**Goal:** Manual outflow to external addresses (e.g., cold storage, OTC, partner wallets). Must be strictly controlled — only to pre-approved whitelist addresses, with high-level approval.

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| Designated Ops | All wallets | All whitelist addresses only | Any amount | Finance Team + Executive |

**Always restrict manual external transfers to whitelist destinations.** See [WHITELIST_API.md](WHITELIST_API.md) for managing whitelisted addresses.

---

## 5. Scenario: API Co-Signer — Approval Callback Logic

When the API Co-Signer is the approver, it calls your **Approval Callback Service**. Start from `CoSignerCallbackController` in [COSIGNER.md](COSIGNER.md) — it already handles callback types, per-`transactionType` business rules and AML/KYT — and add your policy tiers to its `decide()` method, **before** the AML check (so over-limit transactions are rejected immediately instead of waiting for KYT):

```java
// Replaces steps 2–3 at the end of CoSignerCallbackController.decide() (do not keep both copies):

// 2. Business rules by transactionType (order matching for NORMAL / UTXO_BATCH_TRANSFER, system rules otherwise)
if (!passesBusinessRules(tx, memo)) {
    return "REJECT";
}

// 3. Policy tier — defense in depth: your Safeheron policy should already route large amounts to human
//    approvers; if one still reaches the Co-Signer, reject it
if (!isWithinAutoApprovalLimit(tx)) {
    log.warn("REJECT: amount {} {} exceeds auto-approval limit, txKey={}", tx.getTxAmount(), tx.getCoinKey(), tx.getTxKey());
    return "REJECT";
}

// 4. AML / KYT — only LOW / MEDIUM pass; waits (neither APPROVE nor REJECT) while screening is unfinished
AmlDecision amlDecision = evaluateAml(bizContent.getApprovalId(), tx);
if (amlDecision == AmlDecision.WAIT) {
    return ACTION_WAIT;
}
return amlDecision == AmlDecision.PASS ? "APPROVE" : "REJECT";
```

```java
// Limits are per coin, in the coin's own unit — never compare raw amounts of different coins against one number
private static final Map<String, BigDecimal> AUTO_APPROVAL_LIMITS = Map.of(
        "ETHEREUM_ETH", new BigDecimal("10"),
        "USDT(ERC20)_ETHEREUM_USDT", new BigDecimal("50000"));

private boolean isWithinAutoApprovalLimit(TransactionApproval tx) {
    BigDecimal limit = tx.getCoinKey() == null ? null : AUTO_APPROVAL_LIMITS.get(tx.getCoinKey());
    if (limit == null || tx.getTxAmount() == null) {
        return false;   // unknown coin or missing amount — fail closed
    }
    try {
        BigDecimal amount = new BigDecimal(tx.getTxAmount());
        return amount.signum() >= 0 && amount.compareTo(limit) <= 0;
    } catch (NumberFormatException e) {
        return false;
    }
}
```

**The callback service is a critical security control point.** A weak implementation here undermines the entire approval system.

---

## 6. Policy Attack Simulation (Required Before Go-Live)

After configuring all policies, simulate attacks to verify the policies hold:

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use as source | Transaction blocked by policy |
| Withdrawal amount exceeds the per-tx auto-approval threshold | Routed by policy to the Ops Team for manual review; the Co-Signer callback rejects it if it ever reaches it |
| 24H cumulative total exceeds limit within a single day | Escalated to higher approval tier |
| Transaction with no matching policy | Blocked; `NO_MATCHING_TRANSACTION_POLICY` webhook fires |

Run these tests in your **test team** before configuring the production team. Adjust and re-test until every attack is blocked.

---

## 7. Subscribe to the `NO_MATCHING_TRANSACTION_POLICY` Event

This webhook event fires whenever a transaction cannot match any policy. In production, this is a critical alert — it means a transaction was either:
- Submitted with incorrect parameters
- An unauthorized initiator or source wallet
- A policy gap (a scenario you forgot to configure)

```java
// In your webhook handler
if ("NO_MATCHING_TRANSACTION_POLICY".equals(eventType)) {
    alertOpsTeam("Transaction blocked — no matching policy", event);
}
```

**Treat this as a severity-1 alert in your monitoring system.**

---

## 8. Whitelist Management for Manual Transfers

For the manual external transfer policy, maintain a whitelist of approved external addresses in Safeheron:

```java
// Create a whitelist entry
CreateWhitelistRequest req = new CreateWhitelistRequest();
req.setWhitelistName("Cold Storage — BTC Main");
req.setChainType("Bitcoin");
req.setAddress("bc1q...");

CreateWhitelistResponse resp = ServiceExecutor.execute(whitelistApi.createWhitelist(req));
String whitelistKey = resp.getWhitelistKey();
// Use destinationAccountType=WHITELISTING_ACCOUNT + destinationAccountKey=whitelistKey in transactions
```

See [WHITELIST_API.md](WHITELIST_API.md) for full CRUD operations.

---

## 9. Checklist: Policy Configuration Review

Before go-live:

- [ ] All transaction types/scenarios are covered by at least one policy
- [ ] No policy is broader than necessary (source, destination, and initiator are all constrained)
- [ ] Manual external transfers require whitelist destination
- [ ] Large amounts require multi-person, multi-layer approval
- [ ] API Co-Signer callback validates `customerRefId`, amount, and destination
- [ ] `NO_MATCHING_TRANSACTION_POLICY` webhook wired to ops alert
- [ ] Attack simulation tests passed
- [ ] Policy reviewed with Safeheron CSM
