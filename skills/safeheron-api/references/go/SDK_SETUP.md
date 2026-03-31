# Go SDK Setup & Configuration

## Installation

```bash
go get github.com/Safeheron/safeheron-api-sdk-go
```

Module path: `github.com/Safeheron/safeheron-api-sdk-go`

Requires: Go 1.18+

## Clone and Build Locally

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-go.git
cd safeheron-api-sdk-go
go build ./...
```

---

## Key Generation (OpenSSL)

```bash
# Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Export public key (upload to Safeheron Console)
openssl rsa -in api_private.pem -out api_public.pem -pubout
```

The Go SDK reads PEM files directly -- no need to convert to PKCS8 or strip headers.

- `api_private.pem` -- set as `RsaPrivateKey` (file path)
- `api_public.pem` -- upload to Safeheron Console
- The Safeheron platform public key (downloaded from Console) -- save as a PEM file and set as `SafeheronRsaPublicKey` (file path)

---

## ApiConfig Fields

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

| Field | Type | Description |
|-------|------|-------------|
| `BaseUrl` | `string` | Safeheron API base URL |
| `ApiKey` | `string` | API key from Safeheron Web Console |
| `RsaPrivateKey` | `string` | **File path** to your RSA private PEM file |
| `SafeheronRsaPublicKey` | `string` | **File path** to Safeheron platform public PEM file |
| `RequestTimeout` | `int64` | Request timeout in milliseconds (default 20000) |

> **Important:** Unlike the Java SDK which takes base64 strings, the Go SDK takes **PEM file paths** for RSA keys. The PEM files should include the `-----BEGIN/END-----` headers.

---

## Obtaining API Instances

All API instances are created by embedding the `safeheron.Client`:

```go
import "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"

accountApi     := api.AccountApi{Client: sc}
transactionApi := api.TransactionApi{Client: sc}
mpcSignApi     := api.MpcSignApi{Client: sc}
web3Api        := api.Web3Api{Client: sc}
whitelistApi   := api.WhitelistApi{Client: sc}
coinApi        := api.CoinApi{Client: sc}
complianceApi  := api.ComplianceApi{Client: sc}
gasApi         := api.GasApi{Client: sc}
toolsApi       := api.ToolsApi{Client: sc}
webhookApi     := api.WebhookApi{Client: sc}
```

## Executing Calls

All API methods follow the pattern `err := xxxApi.Method(req, &res)`:

```go
var res api.ListAccountResponse
err := accountApi.ListAccounts(req, &res)
if err != nil {
    // handle error
    log.Fatalf("API call failed: %v", err)
}
```

> **Note:** The response is passed as a pointer (`&res`). There is no `ServiceExecutor.execute()` wrapper like in the Java SDK -- you call the method directly and check `err`.
