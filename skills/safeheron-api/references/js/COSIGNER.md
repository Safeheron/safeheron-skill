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

## Approval Callback Service (Express.js)

### Using SDK CoSignerConverter

```typescript
import express from 'express';
import { CoSignerConverter } from '@safeheron/api-sdk';
import { SafeheronCoSignerConfig } from '@safeheron/api-sdk';
import { readFileSync } from 'fs';
import path from 'path';

const app = express();
app.use(express.json());

// Configure the converter
const cosignerConfig: SafeheronCoSignerConfig = {
  approvalCallbackServicePrivateKey: readFileSync(
    path.resolve('./keys/callback_private.pem'), 'utf8'
  ),
  coSignerPubKey: readFileSync(
    path.resolve('./keys/cosigner_identity_public.pem'), 'utf8'
    // WARNING: ensure ./keys/ is listed in .gitignore — never commit private keys
  ),
};

const converter = new CoSignerConverter(cosignerConfig);

app.post('/cosigner/callback', async (req, res) => {
  let payload: any;
  try {
    // 1. Verify signature (Co-Signer identity public key) + decrypt bizContent.
    //    requestV3convert() returns a Buffer — convert it to a string before JSON.parse
    payload = JSON.parse(converter.requestV3convert(req.body).toString());
  } catch (e) {
    // Unverifiable or undecryptable request: there is no trusted approvalId to sign a response for
    console.error('Callback verification/decryption failed:', e);
    return res.status(500).end();
  }

  let action: string;
  try {
    // 2. Business validation (see "Approval Decision" below)
    action = await decide(payload);
  } catch (e) {
    console.error(`Callback evaluation error, approvalId=${payload?.approvalId}:`, e);
    action = 'REJECT';
  }

  try {
    // 3. Sign the response: "APPROVE", "REJECT", or ACTION_WAIT (keeps the approval pending)
    res.json(converter.responseV3convert({ action, approvalId: payload?.approvalId }));
  } catch (e) {
    // e.g. misconfigured callback private key — Express 4 does not catch async errors, so handle it here
    console.error('Failed to sign callback response:', e);
    res.status(500).end();
  }
});

app.listen(9999, () => {
  console.log('Approval Callback Service listening on port 9999');
});
```

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
| `action` | String | `APPROVE` or `REJECT`. Any other value leaves the approval pending and Co-Signer retries every 5s (used to wait for KYT results) |
| `approvalId` | String | Echo back the incoming `approvalId` |

---

## Approval Decision (Default REJECT)

> Placeholders you must implement: `findOrderByCustomerRefId` (your order store — `order.coinKey` and `order.outputs`, the expected `[{address, amount, memo}]` list with `amount` as a **decimal string**), `matchesFuelPolicy`, `matchesSweepPolicy`, `matchesGasDepositPolicy`, `matchesMpcSignRequest`, `matchesWeb3SignRequest`.

```typescript
// Current Co-Signer sends TRANSACTION / MPC_SIGN / WEB3_SIGN / CALLBACK_TEST;
// the other values are legacy type names also recognised by the Safeheron SDK
const TX_TYPES = new Set(['TRANSACTION', 'TX_TRANSACTION', 'CONNECT_TX_SEND', 'TX_CONNECT_TX_SEND', 'TRANSACTION_BATCH_UTXO']);
const WEB3_TYPES = new Set(['WEB3_SIGN', 'ETH_SIGN', 'PERSONAL_SIGN', 'ETH_SIGN_TYPED_DATA', 'ETH_SIGNTRANSACTION']);

// Default REJECT — return APPROVE only when every check passes
async function decide(payload: any): Promise<string> {
  const { approvalId, type, detail } = payload ?? {};
  if (typeof type !== 'string') {
    return 'REJECT';
  }
  if (type === 'CALLBACK_TEST') {
    // Connectivity test sent by `cosigner callback-test` — detail is empty, no real transaction involved
    return 'APPROVE';
  }
  if (type === 'MPC_SIGN') {
    // Verify txKey/customerRefId, sourceAccountKey and dataList against your pending MPC sign request
    return (await matchesMpcSignRequest(detail)) ? 'APPROVE' : 'REJECT';
  }
  if (WEB3_TYPES.has(type)) {
    // Branch on detail.subjectType and verify message / messageHash / transaction against your pending request
    return (await matchesWeb3SignRequest(detail)) ? 'APPROVE' : 'REJECT';
  }
  if (!TX_TYPES.has(type)) {
    console.warn(`REJECT: unknown callback type ${type}`);
    return 'REJECT';
  }

  if (detail == null || typeof detail !== 'object') {
    return 'REJECT';
  }
  const tx = detail;

  // Business rules by transactionType (Safeheron "TRANSACTION Type Approval" best practices)
  if (!(await passesBusinessRules(tx))) {
    console.warn(`REJECT: business rules failed, txKey=${tx.txKey}, transactionType=${tx.transactionType}`);
    return 'REJECT';
  }

  // AML / KYT check (see "AML / KYT Handling in the Callback" below)
  const amlDecision = evaluateAml(approvalId, tx);
  if (amlDecision === 'WAIT') {
    // Neither APPROVE nor REJECT → Co-Signer re-sends this approval every 5s with the latest KYT report
    return ACTION_WAIT;
  }
  return amlDecision === 'PASS' ? 'APPROVE' : 'REJECT';
}

// Per Safeheron "TRANSACTION Type Approval" best practices; any other subtype is rejected
async function passesBusinessRules(tx: any): Promise<boolean> {
  switch (tx.transactionType ?? '') {
    case 'NORMAL':              // API-created, or manually initiated in App / Web Console —
                                // a manual one has no business order and is rejected here
    case 'UTXO_BATCH_TRANSFER':
      return matchesBusinessOrder(tx);
    case 'UTXO_COLLECTION':     // API or manual sweep — destination must be your own vault account
      return tx.destinationAccountType === 'VAULT_ACCOUNT';
    case 'AUTO_FUEL':           // system-initiated by Auto Sweep, no business order
      return tx.destinationAccountType === 'VAULT_ACCOUNT' && matchesFuelPolicy(tx);
    case 'AUTO_SWEEP':          // system-initiated, no business order: destination + USD amount per your policy
      return matchesSweepPolicy(tx);
    case 'GAS_SERVICE_DEPOSIT': // confirm the deposit exists in your records; destination must be PROFILE_ACCOUNT
      return tx.destinationAccountType === 'PROFILE_ACCOUNT' && matchesGasDepositPolicy(tx);
    default:
      return false;
  }
}

async function matchesBusinessOrder(tx: any): Promise<boolean> {
  const order = await findOrderByCustomerRefId(tx.customerRefId);
  if (!order) {
    console.warn(`REJECT: unknown customerRefId ${tx.customerRefId}`);
    return false;
  }
  // Currency must match — same address + amount in a different coin is a different transaction
  if (typeof tx.coinKey !== 'string' || tx.coinKey === '' || tx.coinKey !== order.coinKey) {
    return false;
  }
  // Compare every (address, amount, memo) — exactly one for NORMAL, several for UTXO_BATCH_TRANSFER.
  // The memo decides who is credited on TON / XRP Ledger
  const actual = destinationsOf(tx).map((d) => outputKey(d.address, d.amount, d.memo));
  const expected = (order.outputs ?? []).map((o: any) => outputKey(o.address, o.amount, o.memo));
  if (actual.length === 0 || actual.includes(null) || expected.includes(null)) {
    return false;
  }
  actual.sort();
  expected.sort();
  return actual.length === expected.length && actual.every((key, i) => key === expected[i]);
}

// UTXO coins carry destinations in destinationAddressList (each with its own memo);
// account-model coins use destinationAddress + txAmount + the top-level memo
function destinationsOf(tx: any): Array<{ address?: string; amount?: string; memo?: string }> {
  if (Array.isArray(tx.destinationAddressList) && tx.destinationAddressList.length > 0) {
    return tx.destinationAddressList;
  }
  if (!tx.destinationAddress) {
    return [];
  }
  return [{ address: tx.destinationAddress, amount: tx.txAmount, memo: tx.memo }];
}

function outputKey(address: unknown, amount: unknown, memo: unknown): string | null {
  const normalizedAmount = normalizeAmount(amount);
  if (typeof address !== 'string' || address === '' || normalizedAmount === null) {
    return null;
  }
  // Lower-casing suits EVM (hex) addresses only — compare case-sensitively for base58 / XRP Ledger / TRON / Solana
  return `${address.toLowerCase()}|${normalizedAmount}|${typeof memo === 'string' ? memo : ''}`;
}

// Decimal string normalisation without floats: "1.500" → "1.5", "007" → "7"
function normalizeAmount(amount: unknown): string | null {
  if (typeof amount !== 'string' || !/^\d+(\.\d+)?$/.test(amount)) {
    return null;
  }
  const [intPart, fracPart = ''] = amount.split('.');
  const integer = intPart.replace(/^0+(?=\d)/, '');
  const fraction = fracPart.replace(/0+$/, '');
  return fraction ? `${integer}.${fraction}` : integer;
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

```typescript
type AmlDecision = 'PASS' | 'WAIT' | 'REJECT';

// Source: Safeheron API docs → Compliance → Compliance Best Practices → KYT Status Handling Guide (PENDING):
//   "Approval Callback Service returns a non-APPROVE / non-REJECT value ... API Co-Signer retries approval
//    every 5 seconds with the latest KYT report."
// The literal 'WAIT' is our choice of such a value — verify against your Co-Signer in a test environment.
const ACTION_WAIT = 'WAIT';
const AML_WAIT_TIMEOUT_MS = 5 * 60 * 1000;
// Whitelist: only these risk levels may pass; HIGH, SEVERE, UNKNOWN and any unexpected value are rejected
const ALLOWED_RISK_LEVELS = new Set(['LOW', 'MEDIUM']);
// true: reject when the callback carries no KYT data (Co-Signer < 2.1.1). Set false only if you do not use KYT
const KYT_REQUIRED = true;

// When this service first saw each pending approval. Multi-instance deployments: use Redis SET NX with a TTL
const amlWaitStartedAt = new Map<string, number>();

function evaluateAml(approvalId: string, tx: any): AmlDecision {
  const decision = evaluateAmlState(tx);
  if (decision !== 'WAIT') {
    amlWaitStartedAt.delete(approvalId);
    return decision;
  }
  if (!amlWaitStartedAt.has(approvalId)) {
    amlWaitStartedAt.set(approvalId, Date.now());
  }
  if (Date.now() - amlWaitStartedAt.get(approvalId)! > AML_WAIT_TIMEOUT_MS) {
    amlWaitStartedAt.delete(approvalId);
    console.error(`KYT screening not finished after timeout, txKey=${tx.txKey}`);
    return 'REJECT';   // or route to manual review
  }
  return 'WAIT';
}

// Whitelist style: anything not explicitly recognised as safe is rejected
function evaluateAmlState(tx: any): AmlDecision {
  const state = tx.amlScreeningTriggeredState;
  if (!state) {
    console.warn(`No KYT data in approval callback (Co-Signer < 2.1.1?), txKey=${tx.txKey}`);
    return KYT_REQUIRED ? 'REJECT' : 'PASS';
  }
  switch (state) {
    case 'UNTRIGGERED':
      return 'PASS';
    case 'IN_PROGRESS':
      return 'WAIT';
    case 'TRIGGERED':
      break;
    default:
      console.error(`Unknown amlScreeningTriggeredState ${state}, txKey=${tx.txKey}`);
      return 'REJECT';
  }
  const amlList: any[] = Array.isArray(tx.amlList) ? tx.amlList : [];
  if (amlList.length === 0) {
    console.error(`KYT triggered but amlList is empty — set CALLBACK_INCLUDE_KYT_REPORT=true, txKey=${tx.txKey}`);
    return 'REJECT';
  }
  let pending = false;
  for (const aml of amlList) {
    const status = typeof aml?.status === 'string' ? aml.status.trim().toUpperCase() : '';
    if (status === 'PENDING') {
      pending = true;
    } else if (status === 'COMPLETED') {
      // A completed report without a risk level is treated as UNKNOWN
      const riskLevel = typeof aml.riskLevel === 'string' && aml.riskLevel.trim() !== '' ? aml.riskLevel.trim().toUpperCase() : 'UNKNOWN';
      if (!ALLOWED_RISK_LEVELS.has(riskLevel)) {
        console.warn(`KYT risk ${riskLevel} from ${aml.provider}, txKey=${tx.txKey}`);
        return 'REJECT';
      }
    } else if (status !== 'SKIPPED') {
      // FAILED, missing or unknown status → manual handling
      console.error(`KYT status ${aml?.status} from ${aml?.provider}, txKey=${tx.txKey}`);
      return 'REJECT';
    }
  }
  return pending ? 'WAIT' : 'PASS';
}
```

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

## New vs. Old Co-Signer Version Differences

| Aspect | New Version (v2.x+) | Old Version (v1.x) |
|--------|--------------------|--------------------|
| RSA key pairs required | **1 pair** | 2 pairs |
| Polling interval | 5 seconds | 1 second |
| Cancel transaction in DB | Automatic | Manual SQL required |
| Config management | Secrets Manager (recommended) + .env | Config file |

**Old version** requires configuring the public keys `biz_pubkey` and `api_privkey` in two pairs of public and private keys in the configuration file. The new version only requires registering `biz_pubkey` in the Web console.

---

## Co-Signer Version History

Recent releases (see the Safeheron API docs for SHA-256 checksums and older versions):

| Version | Date | Changes |
|---------|------|---------|
| **2.1.5** | 2026-09-21 | Added support for the XRP chain; added `schemaType` field in the TEE signing structure to indicate the format of the data to be signed; upgraded dependencies to strengthen system security |
| **2.1.4** | 2026-07-14 | Fixed an issue where restarting an inactive Co-Signer caused it to be locked and unable to activate |
| **2.1.3** | 2026-05-10 | Adjusted MPC Sign structure to improve support for message signing initiated by the Message Signing Tool |
| **2.1.2** | 2026-04-21 | Fixed incorrect serialization of hex strings in personal sign; fixed ABI parsing failure caused by dependency version inconsistency; fixed validation failure when the target address is empty in contract deployment scenarios |
| **2.1.1** | 2026-04-08 | Added the AML screening trigger status field in callback notifications; upgraded underlying service framework to improve system stability |

> **Field name note (2.1.1):** the release notes spell it `amlScreeningTriggerState`, while the callback field documented in TransactionApproval (and used by the SDKs) is `amlScreeningTriggeredState`. The examples in this file read `amlScreeningTriggeredState`.
>
> Use **≥ 2.1.1** for KYT data in callbacks (`amlList` additionally needs `CALLBACK_INCLUDE_KYT_REPORT=true`). XRP chain support was added in Co-Signer 2.1.5.

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
https://github.com/Safeheron/safeheron-api-sdk-js

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
