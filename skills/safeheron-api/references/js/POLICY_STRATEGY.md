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

When the API Co-Signer is the approver, it calls your **Approval Callback Service**. Start from the handler and `decide()` in [COSIGNER.md](COSIGNER.md) — they already handle signature verification, callback types (incl. `CALLBACK_TEST`), per-`transactionType` business rules and AML/KYT — and add your policy tiers to `decide()`, **before** the AML check (so over-limit transactions are rejected immediately instead of waiting for KYT):

```typescript
// Replaces the business-rules + AML steps at the end of decide() in COSIGNER.md (do not keep both copies):

// Business rules by transactionType (order matching for NORMAL / UTXO_BATCH_TRANSFER, system rules otherwise)
if (!(await passesBusinessRules(tx))) {
  return 'REJECT';
}

// Policy tier — defense in depth: your Safeheron policy should already route large amounts to human
// approvers; if one still reaches the Co-Signer, reject it
if (!isWithinAutoApprovalLimit(tx)) {
  console.warn(`REJECT: amount ${tx.txAmount} ${tx.coinKey} exceeds auto-approval limit, txKey=${tx.txKey}`);
  return 'REJECT';
}

// AML / KYT — only LOW / MEDIUM pass; waits (neither APPROVE nor REJECT) while screening is unfinished
const amlDecision = evaluateAml(approvalId, tx);
if (amlDecision === 'WAIT') {
  return ACTION_WAIT;
}
return amlDecision === 'PASS' ? 'APPROVE' : 'REJECT';
```

```typescript
// Limits are per coin, in the coin's own unit (decimal strings) — never compare raw amounts of different coins against one number
const AUTO_APPROVAL_LIMITS: Record<string, string> = {
  'ETHEREUM_ETH': '10',
  'USDT(ERC20)_ETHEREUM_USDT': '50000',
};

function isWithinAutoApprovalLimit(tx: any): boolean {
  const limit = typeof tx.coinKey === 'string' ? AUTO_APPROVAL_LIMITS[tx.coinKey] : undefined;
  // normalizeAmount (COSIGNER.md) returns null for missing, negative or malformed amounts — fail closed.
  // Never use Number(): Number('abc') is NaN and `NaN > limit` is false, which would let it through
  const amount = normalizeAmount(tx.txAmount);
  if (limit === undefined || amount === null) {
    return false;
  }
  return compareDecimal(amount, normalizeAmount(limit)!) <= 0;
}

// Compares two normalized non-negative decimal strings without floating point
function compareDecimal(a: string, b: string): number {
  const [aInt, aFrac = ''] = a.split('.');
  const [bInt, bFrac = ''] = b.split('.');
  if (aInt.length !== bInt.length) {
    return aInt.length < bInt.length ? -1 : 1;
  }
  const len = Math.max(aFrac.length, bFrac.length);
  const aDigits = aInt + aFrac.padEnd(len, '0');
  const bDigits = bInt + bFrac.padEnd(len, '0');
  return aDigits === bDigits ? 0 : aDigits < bDigits ? -1 : 1;
}
```

**The callback service is a critical security control point.** A weak implementation here undermines the entire approval system.

---

## 6. Policy Attack Simulation (Required Before Go-Live)

| Attack Scenario | Expected Result |
|----------------|----------------|
| API Key sends to an address NOT in the whitelist | Transaction blocked by policy |
| API Key initiates from a wallet it's not authorized to use as source | Transaction blocked by policy |
| Withdrawal amount exceeds the per-tx auto-approval threshold | Routed by policy to the Ops Team for manual review; the Co-Signer callback rejects it if it ever reaches it |
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
