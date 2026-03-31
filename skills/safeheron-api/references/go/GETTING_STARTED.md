# Getting Started: From Zero to First API Call

## Overview

This guide walks through the complete setup to make your first Safeheron API call using the Go SDK.

**Prerequisites:**
- Go 1.18+
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
```

After running these commands, you will have:

| File | Purpose | Used by |
|------|---------|---------|
| `api_private.pem` | Your RSA private key | **Set file path in SDK config as `RsaPrivateKey`** |
| `api_public.pem` | Public key | **Upload to Safeheron Console** |

> The Go SDK reads PEM files directly by file path -- no need to convert to PKCS8 or strip headers.

> **Security:** Never commit PEM key files to version control. Store them outside the project directory or use a secrets manager.

---

## Step 2 -- Configure API Account in Safeheron Console

Log in to **Safeheron Web Console** and complete the following steps.

### 2-1. Get the Safeheron Platform Public Key

1. Go to **Settings -> API** in the Web Console.
2. Copy the **"Safeheron Public Key"** and save it as a PEM file (e.g. `safeheron_public.pem`).
   This is the `SafeheronRsaPublicKey` file path in your SDK config.

### 2-2. Create an API Key

1. Go to **Settings -> API -> API Keys -> Create API Key**.
2. Fill in:
   - **Name**: any descriptive label
   - **RSA Public Key**: paste the content from `api_public.pem`
   - **Permissions**: select the permissions your application needs
   - **IP Whitelist**: add your server's IP address(es) -- **required**, IPv4 and IPv6 both supported
3. Submit -- an **API Key** string is generated. Copy and save it.

> **IP Whitelist is mandatory.** If your server IP is not registered, all API calls will be rejected with "Illegal IP".

### 2-3. Configure Webhook (Optional for First Call)

Go to **Settings -> API -> Webhook** to configure a callback URL for real-time transaction notifications.
See [WEBHOOK.md](WEBHOOK.md) for details.

### 2-4. Summary of Values Needed

| Config Item | Where to Get | SDK Field |
|-------------|-------------|-----------|
| API Key string | Created in Console | `ApiKey` |
| Safeheron Platform Public Key PEM file | Downloaded from Console | `SafeheronRsaPublicKey` (file path) |
| Your RSA Private Key PEM file | Generated locally | `RsaPrivateKey` (file path) |

---

## Step 3 -- Add SDK to Project Dependencies

```bash
go get github.com/Safeheron/safeheron-api-sdk-go
```

For UUID generation (used for `customerRefId`):
```bash
go get github.com/google/uuid
```

For decimal arithmetic (used for amount calculations):
```bash
go get github.com/shopspring/decimal
```

> SDK updates are **backward compatible** -- new versions only add methods, existing APIs are preserved.

---

## Step 4 -- Inject Credentials

**Never hardcode credentials in source code.**

```go
import "github.com/Safeheron/safeheron-api-sdk-go/safeheron"

sc := safeheron.Client{Config: safeheron.ApiConfig{
    BaseUrl:               "https://api.safeheron.vip",
    ApiKey:                "${SAFEHERON_API_KEY}",
    RsaPrivateKey:         "${RSA_PRIVATE_KEY_PATH}",
    SafeheronRsaPublicKey: "${SAFEHERON_PUBLIC_KEY_PATH}",
    RequestTimeout:        20000,                        // milliseconds
}}
```

### Common Mistakes

| Wrong | Correct |
|-------|---------|
| `RsaPrivateKey` set to base64 string | Set to **file path** (e.g. `"pems/my_private.pem"`) |
| `SafeheronRsaPublicKey` set to your own public key | Set to **Safeheron platform** public key file path |
| PEM file with wrong permissions | `chmod 600 *.pem` |

---

## Step 5 -- Your First API Request

Below is a complete, runnable example: create a wallet, add ETH, then list accounts.

```go
package main

import (
    "fmt"
    "os"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)

func main() {
    // -- Step 1: Build client --
    sc := safeheron.Client{Config: safeheron.ApiConfig{
		BaseUrl:               "https://api.safeheron.vip",
		ApiKey:                "${SAFEHERON_API_KEY}",
		RsaPrivateKey:         "${RSA_PRIVATE_KEY_PATH}",
		SafeheronRsaPublicKey: "${SAFEHERON_PUBLIC_KEY_PATH}",
        RequestTimeout:        20000,
    }}

    // -- Step 2: Create API instance --
    accountApi := api.AccountApi{Client: sc}

    // -- Step 3: Create a wallet account --
    createReq := api.CreateAccountRequest{
        AccountName: "my-first-wallet",
    }
    var createResp api.CreateAccountResponse
    if err := accountApi.CreateAccount(createReq, &createResp); err != nil {
        panic(fmt.Errorf("failed to create wallet account: %w", err))
    }
    fmt.Printf("Wallet created. accountKey: %s\n", createResp.AccountKey)

    // -- Step 4: Add coin to the wallet --
    addCoinReq := api.AddCoinRequest{
        AccountKey: createResp.AccountKey,
        CoinKey:    "ETH(SEPOLIA)_ETHEREUM_SEPOLIA",
    }
    var addCoinResp api.AddCoinResponse
    if err := accountApi.AddCoin(addCoinReq, &addCoinResp); err != nil {
        panic(fmt.Errorf("failed to add coin: %w", err))
    }
    fmt.Printf("Coin added. Address: %s\n", addCoinResp[0].Address)

    // -- Step 5: List wallet accounts --
    listReq := api.ListAccountRequest{
        PageNumber: 1,
        PageSize:   10,
    }
    var listResp api.ListAccountResponse
    if err := accountApi.ListAccounts(listReq, &listResp); err != nil {
        panic(fmt.Errorf("failed to list accounts: %w", err))
    }
    fmt.Printf("Total wallets: %d\n", listResp.TotalElements)
    for _, acct := range listResp.Content {
        fmt.Printf("  %s  %s\n", acct.AccountKey, acct.AccountName)
    }
}
```

### Expected Output

```
Wallet created. accountKey: 4J9rX...
Coin added. Address: 0xAbCd1234...
Total wallets: 1
  4J9rX...  my-first-wallet
```

---

## Troubleshooting First Call

| Error | Cause | Fix |
|-------|-------|-----|
| `1012 Signature verification failed` | Wrong `RsaPrivateKey` file path or file content | Verify PEM file exists and matches the public key uploaded in Console |
| `1010 Parameter decryption failed` | Wrong `SafeheronRsaPublicKey` | Copy the **Safeheron Platform Public Key** from Console (not your own public key) |
| `Illegal IP` / connection rejected | Server IP not in API Key IP whitelist | Add server IP in Console -> API Keys -> IP Whitelist |
| `open pems/xxx.pem: no such file or directory` | PEM file path incorrect | Verify file path is correct relative to working directory |

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
