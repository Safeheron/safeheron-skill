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

When the API Co-Signer is the approver, it calls your **Approval Callback Service**. Implement rigorous checks:

```python
from flask import Flask, request, jsonify
from decimal import Decimal
from safeheron_api_sdk_python.cosigner.co_signer_converter import CoSignerConverter, CoSignerResponseV3

app = Flask(__name__)

AUTO_APPROVAL_LIMIT = Decimal('100000')

@app.route('/cosigner/callback', methods=['POST'])
def handle_callback():
    raw_body = request.get_json(force=True)

    # 1. Verify signature using Co-Signer identity public key -- REJECT if invalid
    biz_content = converter.request_v3_convert(raw_body)
    tx = biz_content.get('customerContent', {})

    action = "APPROVE"

    # 2. Verify customerRefId exists in your DB
    order = find_order_by_ref_id(tx.get('customerRefId'))
    if order is None:
        action = "REJECT"

    # 3. Verify amount matches
    elif Decimal(tx.get('txAmount', '0')) != order['expected_amount']:
        action = "REJECT"

    # 4. Verify destination address matches
    elif tx.get('destinationAddress', '').lower() != order['destination_address'].lower():
        action = "REJECT"

    # 5. Check AML risk
    else:
        for aml in tx.get('amlList', []):
            if aml.get('riskLevel', '').upper() == 'HIGH':
                action = "REJECT"
                break

    # 6. Check amount within auto-approval limit
    if action == "APPROVE" and Decimal(tx.get('txAmount', '0')) > AUTO_APPROVAL_LIMIT:
        action = "REJECT"

    response = CoSignerResponseV3()
    response.action = action
    response.approvalId = biz_content.get('approvalId', '')
    encrypted_response = converter.response_v3_converter(response)
    return jsonify(encrypted_response)
```

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use as source | Transaction blocked by policy |
| Withdrawal amount exceeds the per-tx auto-approval threshold | Escalated to Ops Team |
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
