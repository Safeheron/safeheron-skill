# Python SDK Setup & Configuration

## Install via pip

```bash
pip install safeheron-api-sdk-python
```

## Clone and Build Locally

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-python.git
cd safeheron-api-sdk-python
pip install -e .
```

---

## Config File (config.yaml)

```yaml
apiKey: 080d****e06e60
privateKey: MIIJRQIBA*******DtGRBdennqu8g95jcrMxCUhsifVgzP6vUyg==
safeheronPublicKey: MIICI****QuTOTECAwEAAQ==
baseUrl: https://api.safeheron.vip
requestTimeout: 20000
```

## Loading Config Programmatically

```python
import yaml

with open("config.yaml", "r") as f:
    config = yaml.safe_load(f)

# config is a dict with keys: apiKey, privateKey, safeheronPublicKey, baseUrl, requestTimeout
```

## Config Dict Field Names

| Field | Dict Key | Notes |
|-------|----------|-------|
| API Base URL | `baseUrl` | |
| API Key | `apiKey` | From Safeheron Console |
| Your RSA private key (base64) | `privateKey` | PKCS8 encoded, base64 content without PEM headers |
| Your RSA private key (PEM file) | `privateKeyPemFile` | Path to PEM file (alternative to `privateKey`) |
| Safeheron platform public key (base64) | `safeheronPublicKey` | From Safeheron Console (note: NOT `safeheronRsaPublicKey`) |
| Timeout | `requestTimeout` | Milliseconds (SDK converts to seconds internally) |

> You must provide either `privateKey` (base64 string) or `privateKeyPemFile` (PEM file path), not both.

---

## Obtaining API Service Instances

```python
from safeheron_api_sdk_python.api.account_api import AccountApi
from safeheron_api_sdk_python.api.transaction_api import TransactionApi
from safeheron_api_sdk_python.api.mpc_sign_api import MPCSignApi
from safeheron_api_sdk_python.api.web3_api import Web3Api
from safeheron_api_sdk_python.api.whitelist_api import WhitelistApi
from safeheron_api_sdk_python.api.coin_api import CoinApi
from safeheron_api_sdk_python.api.compliance_api import ComplianceApi
from safeheron_api_sdk_python.api.gas_api import GasApi
from safeheron_api_sdk_python.api.tools_api import ToolsApi
from safeheron_api_sdk_python.api.webhook_api import WebhookApi

account_api     = AccountApi(config)
transaction_api = TransactionApi(config)
mpc_sign_api    = MPCSignApi(config)
web3_api        = Web3Api(config)
whitelist_api   = WhitelistApi(config)
coin_api        = CoinApi(config)
compliance_api  = ComplianceApi(config)
gas_api         = GasApi(config)
tools_api       = ToolsApi(config)
webhook_api     = WebhookApi(config)
```

## Executing Calls

API calls are synchronous — call the method directly on the API instance:

```python
# Correct — direct method call
param = CreateAccountRequest()
param.accountName = "my-wallet"
resp = account_api.create_account(param)

# resp is a dict containing the response data
account_key = resp['accountKey']
```

---

## Flask Integration

```python
import yaml
from flask import Flask
from safeheron_api_sdk_python.api.account_api import AccountApi
from safeheron_api_sdk_python.api.transaction_api import TransactionApi

app = Flask(__name__)

with open("config.yaml", "r") as f:
    config = yaml.safe_load(f)

account_api = AccountApi(config)
transaction_api = TransactionApi(config)
```

---

## Environment Variables (Recommended for Production)

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

---

## Key Generation (OpenSSL)

```bash
# Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Export public key
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Convert to PKCS8 (required for SDK's privateKey field)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
```

Use the base64 content of `api_pkcs8.pem` (strip `-----BEGIN/END-----` headers) as `privateKey`.
Upload `api_public.pem` contents (strip headers) as your RSA public key in Safeheron Console.

Alternatively, you can use `privateKeyPemFile` and point it to the PEM file path directly:

```python
config = {
    'apiKey': 'your-api-key',
    'privateKeyPemFile': '/path/to/api_pkcs8.pem',
    'safeheronPublicKey': 'MIICIjANBgkqhkiG9w0BAQEFAAOCAQ8A...',
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}
```
