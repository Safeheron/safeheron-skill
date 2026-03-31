# Getting Started: From Zero to First API Call

## Overview

This guide walks through the complete setup to make your first Safeheron API call using the TypeScript/JavaScript SDK.

**Prerequisites:**
- Node.js 16+
- npm or yarn
- OpenSSL (pre-installed on macOS/Linux; Windows: use Git Bash or WSL)
- Access to Safeheron Web Console

---

## Step 1 -- Generate RSA Key Pair

Safeheron uses RSA-4096 for request signing and payload encryption. You need to generate your own key pair.

### Generate Keys (OpenSSL)

```bash
# Step 1a: Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Step 1b: Export the public key (this goes to Safeheron Console)
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Step 1c: Convert private key to PKCS8 format (required by the SDK)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt \
    -in api_private.pem -out api_pkcs8.pem
```

After running these commands, you will have:

| File | Purpose | Used by |
|------|---------|---------|
| `api_private.pem` | Original private key | Intermediate only |
| `api_public.pem` | Public key | **Upload to Safeheron Console** |
| `api_pkcs8.pem` | PKCS8-encoded private key | **Set in SDK config as `rsaPrivateKey`** |

> **Security:** Never commit private key files to version control. Inject them via a secrets manager.

---

## Step 2 -- Configure API Account in Safeheron Console

Log in to **Safeheron Web Console** and complete the following steps.

### 2-1. Get the Safeheron Platform Public Key

1. Go to **Settings -> API** in the Web Console.
2. Copy the **"Safeheron Public Key"** (PEM string).
   This is the `safeheronRsaPublicKey` value in your SDK config.

### 2-2. Create an API Key

1. Go to **Settings -> API -> API Keys -> Create API Key**.
2. Fill in:
   - **Name**: any descriptive label
   - **RSA Public Key**: paste the content from `api_public.pem`
   - **Permissions**: select the permissions your application needs (e.g., Initiate Transaction, API Management)
   - **IP Whitelist**: add your server's IP address(es) -- **required**, IPv4 and IPv6 both supported
3. Submit -- an **API Key** string is generated. Copy and save it.

> **IP Whitelist is mandatory.** If your server IP is not registered, all API calls will be rejected with "Illegal IP".

### 2-3. Configure Webhook (Optional for First Call)

Go to **Settings -> API -> Webhook** to configure a callback URL for real-time transaction notifications.
See [WEBHOOK.md](WEBHOOK.md) for details.

### 2-4. Summary of Values Needed

After completing Console setup, you should have:

| Config Item | Where to Get | SDK Field |
|-------------|-------------|-----------|
| API Key string | Created in Console | `apiKey` |
| Safeheron Platform Public Key | Copied from Console | `safeheronRsaPublicKey` |
| Your RSA Private Key (PKCS8 PEM) | From `api_pkcs8.pem` | `rsaPrivateKey` |

---

## Step 3 -- Add SDK to Project

```bash
npm install @safeheron/api-sdk
```

Or with yarn:

```bash
yarn add @safeheron/api-sdk
```

If you need to build from source:

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-js.git
cd safeheron-api-sdk-js
npm install && npm run build
```

> SDK updates are **backward compatible** -- new versions only add methods, existing APIs are preserved.

---

## Step 4 -- Inject Credentials

**Never hardcode credentials in source code.**

```typescript
import { SafeheronConfig } from '@safeheron/api-sdk';

const config: SafeheronConfig = {
  baseUrl: 'https://api.safeheron.vip',
  apiKey: '${SAFEHERON_API_KEY}',//todo Replace with the API Key you read from Safeheron Console
  rsaPrivateKey: '${RSA_PRIVATE_KEY}', //todo Replace with the RSA private key you read from Vault/KMS
  safeheronRsaPublicKey: '${SAFEHERON_PLATFORM_PUBLIC_KEY}',//todo Replace with the Safeheron platform public key from Safeheron Console
  requestTimeout: 20000,  // milliseconds
};
```

### Common Config Mistakes

| Wrong | Correct |
|-------|---------|
| `rsaPrivateKey` set to Safeheron's public key | `rsaPrivateKey` is YOUR PKCS8 private key |
| `safeheronRsaPublicKey` set to your own public key | `safeheronRsaPublicKey` is the PLATFORM public key from Console |
| Private key in PKCS1 format | Must be **PKCS8** (`openssl pkcs8 ...`) |

---

## Step 5 -- Your First API Request

Below is a complete, runnable example: create a wallet, add ETH and USDT, then list accounts.

```typescript
import { AccountApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
import { readFileSync } from 'fs';
import path from 'path';

async function main() {
  // -- Step 1: Build config --
  const config: SafeheronConfig = {
    baseUrl: 'https://api.safeheron.vip',
    apiKey: '${SAFEHERON_API_KEY}',//todo Replace with the API Key you read from Safeheron Console
    rsaPrivateKey: '${RSA_PRIVATE_KEY}', //todo Replace with the RSA private key you read from Vault/KMS
    safeheronRsaPublicKey: '${SAFEHERON_PLATFORM_PUBLIC_KEY}',//todo Replace with the Safeheron platform public key from Safeheron Console
    requestTimeout: 20000,
  };

  // -- Step 2: Create API instance --
  const accountApi = new AccountApi(config);

  try {
    // -- Step 3: Create a wallet account --
    const createResp = await accountApi.createAccount({
      accountName: 'my-first-wallet',
      hiddenOnUI: false,
    });

    const accountKey = createResp.accountKey;
    console.log('Wallet created. accountKey:', accountKey);

    // -- Step 4: Add coins to the wallet --
    const coinResp = await accountApi.createAccountCoinV2({
      accountKey,
      coinKeyList: ['ETHEREUM_ETH', 'USDT(ERC20)_ETHEREUM_USDT'],
    });

    console.log('Coins added:');
    for (const coin of coinResp.coinAddressList) {
      console.log(`  ${coin.coinKey} -> ${coin.addressList[0]?.address}`);
    }

    // -- Step 5: List wallet accounts --
    const listResp = await accountApi.listAccounts({
      pageSize: 10,
      pageNumber: 1,
    });

    console.log('Total wallets:', listResp.totalElements);
    for (const acct of listResp.content) {
      console.log(`  ${acct.accountKey}  ${acct.accountName}`);
    }
  } catch (e) {
    if (e instanceof SafeheronError) {
      console.error(`Error code: ${e.code}, message: ${e.message}`);
    } else {
      console.error(e);
    }
  }
}

main();
```

### Expected Output

```
Wallet created. accountKey: 4J9rX...
Coins added:
  ETHEREUM_ETH -> 0xAbCd1234...
  USDT(ERC20)_ETHEREUM_USDT -> 0xAbCd1234...
Total wallets: 1
  4J9rX...  my-first-wallet
```

---

## Troubleshooting First Call

| Error | Cause | Fix |
|-------|-------|-----|
| `1012 Signature verification failed` | Wrong `rsaPrivateKey` or not PKCS8 format | Re-run `openssl pkcs8 ...`; use full PEM string |
| `1010 Parameter decryption failed` | Wrong `safeheronRsaPublicKey` | Copy the **Safeheron Platform Public Key** from Console (not your own public key) |
| `Illegal IP` / connection rejected | Server IP not in API Key IP whitelist | Add server IP in Console -> API Keys -> IP Whitelist |
| `undefined` on config field | Environment variable not set | Verify `process.env.*` returns non-null values |

---

## Next Steps

| Goal | Reference |
|------|-----------|
| Send a transaction | [TRANSACTION_API.md](TRANSACTION_API.md) |
| Add more coins to wallets | [COIN_API.md](COIN_API.md) |
| Set up a whitelist | [WHITELIST_API.md](WHITELIST_API.md) |
| Receive real-time notifications | [WEBHOOK.md](WEBHOOK.md) |
| Auto-approve transactions | [COSIGNER.md](COSIGNER.md) |
| MPC raw signing | [MPC_SIGN_API.md](MPC_SIGN_API.md) |
| Web3 wallet signing | [WEB3_API.md](WEB3_API.md) |
| Authentication details | [AUTH.md](../AUTH.md) |
| Error codes & troubleshooting | [ERROR_CODES.md](ERROR_CODES.md) |
