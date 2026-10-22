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

> **Note:** Destination must be restricted to `VAULT_ACCOUNT` type.

---

## 4. Scenario: Manual External Transfer Policy

| Initiator | Source | Destination | Condition | Approval Node |
|-----------|--------|-------------|-----------|---------------|
| Designated Ops | All wallets | All whitelist addresses only | Any amount | Finance Team + Executive |

**Always restrict manual external transfers to whitelist destinations.** See [WHITELIST_API.md](WHITELIST_API.md).

---

## 5. Scenario: API Co-Signer -- Approval Callback Logic

When the API Co-Signer is the approver, it calls your **Approval Callback Service**. Start from the Flask callback service in [COSIGNER.md](COSIGNER.md) — it already handles callback types, per-`transactionType` business rules and AML/KYT — and add your policy tiers to its `decide()` function, **before** the AML check (so over-limit transactions are rejected immediately instead of waiting for KYT):

```python
from decimal import Decimal, InvalidOperation

# Limits are per coin, in the coin's own unit -- never compare raw amounts of different coins against one number
AUTO_APPROVAL_LIMITS = {
    'ETHEREUM_ETH': Decimal('10'),
    'USDT(ERC20)_ETHEREUM_USDT': Decimal('50000'),
}


def within_auto_approval_limit(tx):
    # Unknown coin, missing / malformed / NaN / Infinity / negative amount -> never within the limit (fail closed)
    limit = AUTO_APPROVAL_LIMITS.get(tx.get('coinKey'))
    if limit is None:
        return False
    try:
        amount = Decimal(tx.get('txAmount'))
    except (InvalidOperation, TypeError, ValueError):
        return False
    return amount.is_finite() and Decimal('0') <= amount <= limit


# Replaces the steps after the callback-type checks at the end of decide() in COSIGNER.md
# (do not keep both copies; detail = TransactionApproval dict):

    # 1. Business rules by transactionType (order matching for NORMAL / UTXO_BATCH_TRANSFER, system rules otherwise)
    if not passes_business_rules(detail):
        return 'REJECT'

    # 2. Policy tier -- defense in depth: your Safeheron policy should already route large amounts to human
    #    approvers; if one still reaches the Co-Signer, reject it
    if not within_auto_approval_limit(detail):
        app.logger.warning(f"REJECT: amount {detail.get('txAmount')} {detail.get('coinKey')} exceeds auto-approval limit, "
                           f"txKey={detail.get('txKey')}")
        return 'REJECT'

    # 3. AML / KYT -- only LOW / MEDIUM pass; waits (neither APPROVE nor REJECT) while screening is unfinished
    aml_decision = evaluate_aml(approval_id, detail)
    if aml_decision == AML_WAIT:
        return ACTION_WAIT
    return 'APPROVE' if aml_decision == AML_PASS else 'REJECT'
```

**The callback service is a critical security control point.** A weak implementation here undermines the entire approval system.

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use as source | Transaction blocked by policy |
| Withdrawal amount exceeds the per-tx auto-approval threshold | Routed by policy to the Ops Team for manual review; the Co-Signer callback rejects it if it ever reaches it |
| 24H cumulative total exceeds limit within a single day | Escalated to higher approval tier |
| Transaction with no matching policy | Blocked; `NO_MATCHING_TRANSACTION_POLICY` webhook fires |

---

## 7. Subscribe to the `NO_MATCHING_TRANSACTION_POLICY` Event

```python
# In your webhook handler
if event_type == 'NO_MATCHING_TRANSACTION_POLICY':
    alert_ops_team("Transaction blocked -- no matching policy", biz_content)
```

**Treat this as a severity-1 alert in your monitoring system.**

---

## 8. Whitelist Management for Manual Transfers

```python
from safeheron_api_sdk_python.api.whitelist_api import WhitelistApi, CreateWhitelistRequest

whitelist_api = WhitelistApi(config)

param = CreateWhitelistRequest()
param.whitelistName = "Cold Storage -- BTC Main"
param.chainType = "Bitcoin"
param.address = "bc1q..."

resp = whitelist_api.create_whitelist(param)
whitelist_key = resp['whitelistKey']
# Use destinationAccountType="WHITELISTING_ACCOUNT" + destinationAccountKey=whitelist_key
```

---

## 9. Checklist: Policy Configuration Review

Before go-live:

- [ ] All transaction types/scenarios are covered by at least one policy
- [ ] No policy is broader than necessary
- [ ] Manual external transfers require whitelist destination
- [ ] Large amounts require multi-person, multi-layer approval
- [ ] API Co-Signer callback validates `customerRefId`, amount, and destination
- [ ] `NO_MATCHING_TRANSACTION_POLICY` webhook wired to ops alert
- [ ] Attack simulation tests passed
- [ ] Policy reviewed with Safeheron CSM
