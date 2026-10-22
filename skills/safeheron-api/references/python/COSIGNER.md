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

## Approval Callback -- Python SDK Integration

### Configure CoSignerConverter

```python
from safeheron_api_sdk_python.cosigner.co_signer_converter import (
    CoSignerConverter,
    CoSignerResponseV3,
)

# Config for Co-Signer callback
cosigner_config = {
    'coSignerPubKey': 'MIICIjANBgkqhk...',  # Co-Signer identity public key (from export-public-key)
    'approvalCallbackServicePrivateKey': 'MIIJQgIBADANBgk...',  # Your callback private key (base64)
    # OR use PEM file:
    # 'approvalCallbackServicePrivateKeyPemFile': '/path/to/callback_private.pem',
}

converter = CoSignerConverter(cosigner_config)
```

### Flask Approval Callback Service (V3 -- Recommended)

> Placeholders you must implement: `find_order_by_ref_id` (returns your pending order or `None`; the order is a dict with `coin_key` and `outputs` — the expected destinations as `[{'address', 'amount', 'memo'}]`, `amount` a decimal string), `matches_fuel_policy`, `matches_sweep_policy`, `matches_gas_deposit_policy`, `matches_mpc_sign_request`, `matches_web3_sign_request`. Callback `destinationAddressList` items carry `address` / `amount` / `memo`.

```python
from flask import Flask, request, jsonify
from decimal import Context, Decimal, InvalidOperation

app = Flask(__name__)


@app.route('/cosigner/callback', methods=['POST'])
def handle_cosigner_callback():
    approval_id = ''
    try:
        # Decrypt and verify signature (V3 protocol) -- returns the decrypted bizContent as a dict
        biz_content = converter.request_v3_convert(request.get_json(force=True))
        approval_id = biz_content.get('approvalId', '')
        action = decide(approval_id, biz_content.get('type', ''), biz_content.get('detail') or {})
    except Exception:
        app.logger.exception("Callback processing error -- rejecting")
        action = 'REJECT'

    response = CoSignerResponseV3()
    response.action = action  # "APPROVE", "REJECT", or ACTION_WAIT (keeps the approval pending)
    response.approvalId = approval_id
    return jsonify(converter.response_v3_converter(response))


# Current Co-Signer sends TRANSACTION / MPC_SIGN / WEB3_SIGN / CALLBACK_TEST;
# the other values are legacy type names also recognised by the Safeheron SDK
TX_TYPES = ('TRANSACTION', 'TX_TRANSACTION', 'CONNECT_TX_SEND', 'TX_CONNECT_TX_SEND', 'TRANSACTION_BATCH_UTXO')
WEB3_TYPES = ('WEB3_SIGN', 'ETH_SIGN', 'PERSONAL_SIGN', 'ETH_SIGN_TYPED_DATA', 'ETH_SIGNTRANSACTION')


# Default REJECT -- return APPROVE only when every check passes
def decide(approval_id, callback_type, detail):
    if callback_type == 'CALLBACK_TEST':
        # Connectivity test sent by `cosigner callback-test` -- detail is empty, no real transaction involved
        return 'APPROVE'
    if callback_type == 'MPC_SIGN':
        # Verify txKey/customerRefId, sourceAccountKey and dataList against your pending MPC sign request
        return 'APPROVE' if matches_mpc_sign_request(detail) else 'REJECT'
    if callback_type in WEB3_TYPES:
        # Branch on detail['subjectType'] and verify message / messageHash / transaction against your pending request
        return 'APPROVE' if matches_web3_sign_request(detail) else 'REJECT'
    if callback_type not in TX_TYPES:
        app.logger.warning(f"REJECT: unknown callback type {callback_type}")
        return 'REJECT'

    # 1. Business rules by transactionType
    if not passes_business_rules(detail):
        app.logger.warning(f"REJECT: business rules failed, txKey={detail.get('txKey')}, "
                           f"transactionType={detail.get('transactionType')}")
        return 'REJECT'

    # 2. AML / KYT check (see "AML / KYT Handling in the Callback" below)
    aml_decision = evaluate_aml(approval_id, detail)
    if aml_decision == AML_WAIT:
        # Neither APPROVE nor REJECT -> Co-Signer re-sends this approval every 5s with the latest KYT report
        return ACTION_WAIT
    return 'APPROVE' if aml_decision == AML_PASS else 'REJECT'


# Per Safeheron "TRANSACTION Type Approval" best practices; any other subtype is rejected
def passes_business_rules(tx):
    transaction_type = tx.get('transactionType') or ''
    destination_account_type = tx.get('destinationAccountType')
    if transaction_type in ('NORMAL',                # API-created, or manually initiated in App / Web Console --
                                                     # a manual one has no business order and is rejected here
                            'UTXO_BATCH_TRANSFER'):
        return matches_business_order(tx)
    if transaction_type == 'UTXO_COLLECTION':        # API or manual sweep -- destination must be your own vault account
        return destination_account_type == 'VAULT_ACCOUNT'
    if transaction_type == 'AUTO_FUEL':              # system-initiated by Auto Sweep, no business order
        return destination_account_type == 'VAULT_ACCOUNT' and matches_fuel_policy(tx)
    if transaction_type == 'AUTO_SWEEP':             # system-initiated, no business order: destination + USD amount per your policy
        return matches_sweep_policy(tx)
    if transaction_type == 'GAS_SERVICE_DEPOSIT':    # confirm the deposit exists in your records; destination must be PROFILE_ACCOUNT
        return destination_account_type == 'PROFILE_ACCOUNT' and matches_gas_deposit_policy(tx)
    return False


def matches_business_order(tx):
    order = find_order_by_ref_id(tx.get('customerRefId', ''))
    if order is None:
        return False
    # Currency must match -- same address + amount in a different coin is a different transaction
    if not tx.get('coinKey') or tx.get('coinKey') != order.get('coin_key'):
        return False
    # Compare every (address, amount, memo) -- exactly one for NORMAL, several for UTXO_BATCH_TRANSFER.
    # The memo decides who is credited on TON / XRP Ledger.
    # Build all keys first: any missing address/amount or invalid amount means "no match" (never sort a list with None)
    actual = [output_key(d.get('address'), d.get('amount'), d.get('memo')) for d in destinations_of(tx)]
    expected = [output_key(o.get('address'), o.get('amount'), o.get('memo')) for o in order.get('outputs') or []]
    if not actual or None in actual or None in expected:
        return False
    return sorted(actual) == sorted(expected)


# UTXO coins carry destinations in destinationAddressList; account-model coins use destinationAddress + txAmount
def destinations_of(tx):
    if tx.get('destinationAddressList'):
        return tx['destinationAddressList']
    if not tx.get('destinationAddress'):
        return []
    return [{'address': tx['destinationAddress'], 'amount': tx.get('txAmount'), 'memo': tx.get('memo')}]


# Wide enough that normalize() never rounds a real on-chain amount (the default context keeps only 28 digits)
_AMOUNT_CONTEXT = Context(prec=100)


def output_key(address, amount, memo):
    if not isinstance(address, str) or not address or not isinstance(amount, str) or not amount:
        return None
    try:
        value = Decimal(amount)
    except InvalidOperation:
        return None
    if not value.is_finite():
        return None
    # Lower-casing suits EVM (hex) addresses only -- compare case-sensitively for base58 / XRP Ledger / TRON / Solana
    return f"{address.lower()}|{value.normalize(_AMOUNT_CONTEXT)}|{memo or ''}"
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

```python
import threading
import time

AML_PASS, AML_WAIT, AML_REJECT = 'PASS', 'WAIT', 'REJECT'

# Any value other than APPROVE / REJECT keeps the approval pending. Source: Safeheron API docs -> Compliance ->
# "Compliance Best Practices" (KYT Status Handling Guide): "Approval Callback Service returns a non-APPROVE /
# non-REJECT value ... API Co-Signer retries approval every 5 seconds with the latest KYT report".
# The literal 'WAIT' is our choice of such a value.
ACTION_WAIT = 'WAIT'
AML_WAIT_TIMEOUT_SECONDS = 5 * 60
# Whitelist: only these risk levels may pass; HIGH, SEVERE, UNKNOWN and any unexpected value are rejected
ALLOWED_RISK_LEVELS = {'LOW', 'MEDIUM'}
# True: reject when the callback carries no KYT data (Co-Signer < 2.1.1). Set False only if you do not use KYT
KYT_REQUIRED = True

# When this service first saw each pending approval. Multi-instance or multi-worker (e.g. gunicorn) deployments:
# this dict is per process -- use Redis SET NX with a TTL instead
_aml_wait_started_at = {}
_aml_wait_lock = threading.Lock()


def evaluate_aml(approval_id, tx):
    decision = evaluate_aml_state(tx)
    with _aml_wait_lock:
        if decision != AML_WAIT:
            _aml_wait_started_at.pop(approval_id, None)
            return decision
        started_at = _aml_wait_started_at.setdefault(approval_id, time.monotonic())
        if time.monotonic() - started_at > AML_WAIT_TIMEOUT_SECONDS:
            _aml_wait_started_at.pop(approval_id, None)
            app.logger.error(f"KYT screening not finished after timeout, txKey={tx.get('txKey')}")
            return AML_REJECT  # or route to manual review
    return AML_WAIT


# Whitelist style: anything not explicitly recognised as safe is rejected
def evaluate_aml_state(tx):
    state = tx.get('amlScreeningTriggeredState')
    if not state:
        app.logger.warning(f"No KYT data in approval callback (Co-Signer < 2.1.1?), txKey={tx.get('txKey')}")
        return AML_REJECT if KYT_REQUIRED else AML_PASS
    if state == 'UNTRIGGERED':
        return AML_PASS
    if state == 'IN_PROGRESS':
        return AML_WAIT
    if state != 'TRIGGERED':
        app.logger.error(f"Unknown amlScreeningTriggeredState {state}, txKey={tx.get('txKey')}")
        return AML_REJECT

    aml_list = tx.get('amlList') or []
    if not aml_list:
        app.logger.error(f"KYT triggered but amlList is empty -- set CALLBACK_INCLUDE_KYT_REPORT=true, txKey={tx.get('txKey')}")
        return AML_REJECT
    pending = False
    for aml in aml_list:
        status = str(aml.get('status') or '').strip().upper()
        if status == 'PENDING':
            pending = True
        elif status == 'COMPLETED':
            # A completed report without a risk level is treated as UNKNOWN
            risk_level = str(aml.get('riskLevel') or 'UNKNOWN').strip().upper()
            if risk_level not in ALLOWED_RISK_LEVELS:
                app.logger.warning(f"KYT risk {risk_level} from {aml.get('provider')}, txKey={tx.get('txKey')}")
                return AML_REJECT
        elif status != 'SKIPPED':
            # FAILED, missing or unknown status -> manual handling
            app.logger.error(f"KYT status {aml.get('status')} from {aml.get('provider')}, txKey={tx.get('txKey')}")
            return AML_REJECT
    return AML_WAIT if pending else AML_PASS
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
| 2.1.1 | 2026-04-08 | Added the AML screening trigger status field in callback notifications; upgraded underlying service framework to improve system stability |

> The 2.1.1 release note spells the field `amlScreeningTriggerState`; the callback field documented in the Approval Callback Service reference (and used by the SDKs) is `amlScreeningTriggeredState`. XRP chain support was added in **2.1.5**; KYT data in callbacks needs **2.1.1+** (see [AML / KYT Handling](#aml--kyt-handling-in-the-callback)).

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
https://github.com/Safeheron/safeheron-api-sdk-python

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
3. **Apply the rules for each `transactionType`** — for `NORMAL` / `UTXO_BATCH_TRANSFER`, the `customerRefId` must match a pending order and coin, every destination address, amount and memo must match it; system subtypes (`AUTO_SWEEP`, `AUTO_FUEL`, `UTXO_COLLECTION`, `GAS_SERVICE_DEPOSIT`) follow their own rules — see `passes_business_rules`
4. **Apply the AML / KYT rules** — see `evaluate_aml`
5. **Respond within 5 seconds** — Co-Signer will time out otherwise (sync server time via NTP)
