# Getting Started: From Zero to First API Call

## Overview

This guide walks through the complete setup to make your first Safeheron API call using the Python SDK.

**Prerequisites:**
- Python 3.7+
- pip
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

# Step 1c: Convert private key to PKCS8 format (required by the Python SDK)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt \
    -in api_private.pem -out api_pkcs8.pem
```

After running these commands, you will have:

| File | Purpose | Used by |
|------|---------|---------|
| `api_private.pem` | Original private key | Intermediate only |
| `api_public.pem` | Public key | **Upload to Safeheron Console** |
| `api_pkcs8.pem` | PKCS8-encoded private key | **Set in SDK config as `privateKey` or `privateKeyPemFile`** |

### Extract the Base64 Values

The SDK requires the **raw base64 content** without PEM headers/footers when using the `privateKey` config field.

```bash
# Extract public key base64 (remove header/footer lines)
grep -v "BEGIN\|END" api_public.pem | tr -d '\n'

# Extract PKCS8 private key base64 (remove header/footer lines)
grep -v "BEGIN\|END" api_pkcs8.pem | tr -d '\n'
```

> **Security:** Never commit private key files to version control. Inject them via environment variables or a secrets manager.

---

## Step 2 -- Configure API Account in Safeheron Console

Log in to **Safeheron Web Console** and complete the following steps.

### 2-1. Get the Safeheron Platform Public Key

1. Go to **Settings -> API** in the Web Console.
2. Copy the **"Safeheron Public Key"** (base64 string).
   This is the `safeheronPublicKey` value in your SDK config.

### 2-2. Create an API Key

1. Go to **Settings -> API -> API Keys -> Create API Key**.
2. Fill in:
   - **Name**: any descriptive label
   - **RSA Public Key**: paste the base64 content from `api_public.pem` (no headers)
   - **Permissions**: select the permissions your application needs (e.g., Initiate Transaction, API Management)
   - **IP Whitelist**: add your server's IP address(es) -- **required**, IPv4 and IPv6 both supported
3. Submit -- an **API Key** string is generated. Copy and save it.

> **IP Whitelist is mandatory.** If your server IP is not registered, all API calls will be rejected with "Illegal IP".

### 2-3. Configure Webhook (Optional for First Call)

Go to **Settings -> API -> Webhook** to configure a callback URL for real-time transaction notifications.
See [WEBHOOK.md](WEBHOOK.md) for details.

### 2-4. Summary of Values Needed

After completing Console setup, you should have:

| Config Item | Where to Get | SDK Config Key |
|-------------|-------------|----------------|
| API Key string | Created in Console | `apiKey` |
| Safeheron Platform Public Key | Copied from Console | `safeheronPublicKey` |
| Your RSA Private Key (PKCS8 base64) | From `api_pkcs8.pem` | `privateKey` |

---

## Step 3 -- Install the SDK

```bash
pip install safeheron-api-sdk-python
```

If you also need YAML config loading:

```bash
pip install pyyaml
```

### Install from Source (Optional)

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-python.git
cd safeheron-api-sdk-python
pip install -e .
```

> SDK updates are **backward compatible** -- new versions only add methods, existing APIs are preserved.

---

## Step 4 -- Configure Environment & Inject Credentials

**Never hardcode credentials in source code.**

Choose one of the following approaches.

### Option A -- Environment Variables (Recommended)

Set environment variables on your server or in your CI/CD pipeline:

```bash
export SAFEHERON_API_KEY="your-api-key-here"
export SAFEHERON_RSA_PRIVATE_KEY="MIIJQgIBADANBgkqhkiG9w0BAQEFAASC..."
export SAFEHERON_PLATFORM_PUBLIC_KEY="MIICIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."
```

Load in Python:

```python
import os

config = {
    'apiKey': os.environ['SAFEHERON_API_KEY'],
    'privateKey': os.environ['SAFEHERON_RSA_PRIVATE_KEY'],
    'safeheronPublicKey': os.environ['SAFEHERON_PLATFORM_PUBLIC_KEY'],
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}
```

### Option B -- Local Config File (Development Only)

Create `config.yaml` outside your source tree (never committed):

```yaml
apiKey: "your-api-key"
privateKey: "MIIJQgIBADANBgkqhkiG9w0BAQEFAASC..."    # PKCS8 base64, no headers
safeheronPublicKey: "MIICIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."  # Platform public key
baseUrl: "https://api.safeheron.vip"
requestTimeout: 20000
```

Load it:

```python
import yaml

with open("config.yaml", "r") as f:
    config = yaml.safe_load(f)
```

Add to `.gitignore`:
```
config.yaml
*.pem
```

### Common Config Mistakes

| Wrong | Correct |
|-------|---------|
| `privateKey` set to Safeheron's public key | `privateKey` must be YOUR PKCS8 private key |
| `safeheronPublicKey` set to your public key | `safeheronPublicKey` must be the PLATFORM public key from Console |
| Private key with `-----BEGIN...` headers | Strip headers, raw base64 only (or use `privateKeyPemFile` instead) |
| Private key in PKCS1 format | Must be **PKCS8** (`openssl pkcs8 ...`) |
| Using `safeheronRsaPublicKey` as config key | Python SDK uses `safeheronPublicKey` (no `Rsa` in the name) |

---

## Step 5 -- Your First API Request

Below is a complete, runnable example: create a wallet, add ETH, then list accounts.

```python
import os
import uuid
from safeheron_api_sdk_python.api.account_api import (
    AccountApi,
    CreateAccountRequest,
    ListAccountRequest,
    CreateAccountCoinRequest,
)

# -- Step 1: Build config --
config = {
    'apiKey': os.environ['SAFEHERON_API_KEY'],
    'privateKey': os.environ['SAFEHERON_RSA_PRIVATE_KEY'],
    'safeheronPublicKey': os.environ['SAFEHERON_PLATFORM_PUBLIC_KEY'],
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}

# -- Step 2: Create API instance --
account_api = AccountApi(config)

# -- Step 3: Create a wallet account --
create_param = CreateAccountRequest()
create_param.accountName = "my-first-wallet"
create_param.hiddenOnUI = False

create_resp = account_api.create_account(create_param)
account_key = create_resp['accountKey']
print(f"Wallet created. accountKey: {account_key}")

# -- Step 4: Add coins to the wallet --
coin_param = CreateAccountCoinRequest()
coin_param.accountKey = account_key
coin_param.coinKey = "ETHEREUM_ETH"

coin_resp = account_api.create_account_coin(coin_param)
print("Coin added:")
for coin in coin_resp:
    print(f"  address: {coin['address']}")

# -- Step 5: List wallet accounts --
list_param = ListAccountRequest()
list_param.pageSize = 10
list_param.pageNumber = 1

list_resp = account_api.list_accounts(list_param)
print(f"Total wallets: {list_resp['totalElements']}")
for acct in list_resp['content']:
    print(f"  {acct['accountKey']}  {acct['accountName']}")
```

### Expected Output

```
Wallet created. accountKey: 4J9rX...
Coin added:
  address: 0xAbCd1234...
Total wallets: 1
  4J9rX...  my-first-wallet
```

---

## Troubleshooting First Call

| Error | Cause | Fix |
|-------|-------|-----|
| `1012 Signature verification failed` | Wrong `privateKey` or not PKCS8 format | Re-run `openssl pkcs8 ...`; ensure raw base64, no headers |
| `1010 Parameter decryption failed` | Wrong `safeheronPublicKey` | Copy the **Safeheron Platform Public Key** from Console (not your own public key) |
| `Illegal IP` / connection rejected | Server IP not in API Key IP whitelist | Add server IP in Console -> API Keys -> IP Whitelist |
| `KeyError` on config | Config dict missing required keys | Verify all required keys are present: `apiKey`, `privateKey`/`privateKeyPemFile`, `safeheronPublicKey`, `baseUrl` |
| `requests.exceptions.Timeout` | Request timed out | Increase `requestTimeout` value (in milliseconds) |

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
