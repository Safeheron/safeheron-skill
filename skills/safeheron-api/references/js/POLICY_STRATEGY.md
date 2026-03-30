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

```typescript
import express from 'express';

app.post('/cosigner/callback', async (req, res) => {
  // 1. Verify signature using Co-Signer identity public key -- REJECT if invalid
  const payload = converter.requestV3convert(req.body);
  const tx = JSON.parse(payload);

  // 2. Verify customerRefId exists in your DB -- REJECT unknown transactions
  const order = await withdrawalOrderDao.findByCustomerRefId(tx.customerRefId);
  if (!order) {
    console.warn(`REJECT: unknown customerRefId ${tx.customerRefId}`);
    const resp = converter.responseV3convert({ action: 'REJECT', approvalId: tx.approvalId });
    return res.json(resp);
  }

  // 3. Verify amount matches what the user requested
  // Compare as strings: Safeheron normalizes txAmount (e.g. "0.001"); store
  // expectedAmount in the same format when you create the withdrawal order.
  if (tx.txAmount !== order.expectedAmount) {
    console.warn(`REJECT: amount mismatch for ${tx.customerRefId}`);
    const resp = converter.responseV3convert({ action: 'REJECT', approvalId: tx.approvalId });
    return res.json(resp);
  }

  // 4. Verify destination address matches
  if (tx.destinationAddress !== order.destinationAddress) {
    console.warn(`REJECT: destination mismatch for ${tx.customerRefId}`);
    const resp = converter.responseV3convert({ action: 'REJECT', approvalId: tx.approvalId });
    return res.json(resp);
  }

  // 5. Check AML risk
  for (const aml of tx.amlList || []) {
    if (aml.riskLevel === 'HIGH') {
      const resp = converter.responseV3convert({ action: 'REJECT', approvalId: tx.approvalId });
      return res.json(resp);
    }
  }

  // 6. Check amount within auto-approval limit
  if (parseFloat(tx.txAmount) > AUTO_APPROVAL_LIMIT) {
    const resp = converter.responseV3convert({ action: 'REJECT', approvalId: tx.approvalId });
    return res.json(resp);
  }

  // 7. All checks passed -- approve
  const resp = converter.responseV3convert({ action: 'APPROVE', approvalId: tx.approvalId });
  res.json(resp);
});
```

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use as source | Transaction blocked by policy |
| Withdrawal amount exceeds the per-tx auto-approval threshold | Escalated to Ops Team |
| 24H cumulative total exceeds limit | Escalated to higher approval tier |
| Transaction with no matching policy | Blocked; `NO_MATCHING_TRANSACTION_POLICY` webhook fires |

---

## 7. Subscribe to the `NO_MATCHING_TRANSACTION_POLICY` Event

```typescript
// In your webhook handler
if (event.eventType === 'NO_MATCHING_TRANSACTION_POLICY') {
  alertOpsTeam('Transaction blocked -- no matching policy', event);
}
```

**Treat this as a severity-1 alert in your monitoring system.**

---

## 8. Whitelist Management for Manual Transfers

```typescript
import { WhitelistApi } from '@safeheron/api-sdk';

const whitelistApi = new WhitelistApi(config);

const resp = await whitelistApi.createWhitelist({
  whitelistName: 'Cold Storage -- BTC Main',
  chainType: 'Bitcoin',
  address: 'bc1q...',
});

const whitelistKey = resp.whitelistKey;
// Use destinationAccountType='WHITELISTING_ACCOUNT' + destinationAccountKey=whitelistKey in transactions
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
