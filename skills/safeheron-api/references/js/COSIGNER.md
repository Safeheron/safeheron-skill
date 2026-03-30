# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a service you deploy that **automatically approves or rejects transactions** based on your business logic. It replaces manual approval workflows for programmatic use cases and is required for unattended operation in production environments.

When a transaction requires approval, Safeheron calls your **Approval Callback Service** (an HTTP endpoint you implement) with an encrypted callback payload. Your service evaluates the transaction and responds `APPROVE` or `REJECT`.

---

## How It Works

```
1. Client creates transaction / MPC sign / Web3 sign
2. Safeheron: transaction enters SUBMITTED status
3. API Co-Signer polls Safeheron (every 5s in v2.x, every 1s in v1.x)
4. API Co-Signer calls your Approval Callback Service (HTTP POST)
5. Your service returns APPROVE or REJECT
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
  ),
};

const converter = new CoSignerConverter(cosignerConfig);

app.post('/cosigner/callback', async (req, res) => {
  try {
    // 1. Decrypt and verify signature
    //    requestV3convert() handles:
    //    - Verifies RSA signature using Co-Signer identity public key
    //    - Decrypts AES key using your callback private key
    //    - Decrypts bizContent
    const decrypted = converter.requestV3convert(req.body);
    const payload = JSON.parse(decrypted);

    // 2. Business validation
    const action = await evaluateTransaction(payload);

    // 3. Encrypt response
    const encryptedResponse = converter.responseV3convert({
      action: action,  // "APPROVE" or "REJECT"
      approvalId: payload.approvalId,
    });

    res.json(encryptedResponse);
  } catch (e) {
    console.error('Callback error:', e);
    // On error, respond with REJECT to be safe
    const encryptedResponse = converter.responseV3convert({
      action: 'REJECT',
      approvalId: '',
    });
    res.json(encryptedResponse);
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
  "bizContent": "<AES-encrypted-callback-payload-base64>",
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
  "type": "TRANSACTION_APPROVAL",
  "customerContent": { ... }
}
```

### Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Regular transaction approval |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval |

---

## TRANSACTION_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `txHash` | String | Transaction hash (if available) |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `transactionType` | String | Transaction type |
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
| `amlLock` | String | AML status: `YES` / `NO` |
| `amlList` | List\<Aml\> | AML assessments (provider, riskLevel, etc.) |
| `createTime` | Long | Unix timestamp (ms) |

---

## WEB3_SIGN_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Web3 sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
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

## MPC_SIGN_APPROVAL Payload (`customerContent`)

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

Your callback service must return an **encrypted response** using the same AES+RSA scheme, signed with your Callback Private Key:

```json
{
  "action": "APPROVE",
  "approvalId": "tx*******"
}
```

| Field | Type | Values |
|-------|------|--------|
| `action` | String | `APPROVE` or `REJECT` |
| `approvalId` | String | Echo back the incoming `txKey` |

---

## Approval Logic Examples

### Allow Only Whitelisted Destinations

```typescript
function evaluateTransaction(payload: any): string {
  const allowedAddresses = loadWhitelistedAddresses(payload.coinKey);
  if (!allowedAddresses.has(payload.destinationAddress)) {
    return 'REJECT';
  }
  return 'APPROVE';
}
```

### Amount Limit per Account

```typescript
function evaluateTransaction(payload: any): string {
  const amount = parseFloat(payload.txAmount);
  const limit = getAccountLimit(payload.sourceAccountKey);
  if (amount > limit) {
    return 'REJECT';
  }
  return 'APPROVE';
}
```

### AML Risk Check

```typescript
function evaluateTransaction(payload: any): string {
  for (const aml of payload.amlList || []) {
    if (aml.riskLevel === 'HIGH') {
      return 'REJECT';
    }
  }
  return 'APPROVE';
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
3. **Validate `customerRefId`** — reject any callback for a `customerRefId` not found in your DB
4. **Validate amount and destination** — cross-check against the original withdrawal order
5. **Respond within 5 seconds** — Co-Signer will time out otherwise (sync server time via NTP)
