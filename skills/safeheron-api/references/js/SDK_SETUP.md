# TypeScript/JavaScript SDK Setup & Configuration

## Install via npm

```bash
npm install @safeheron/api-sdk
```

## Install via yarn

```bash
yarn add @safeheron/api-sdk
```

## Clone and Build Locally

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-js.git
cd safeheron-api-sdk-js
npm install
npm run build
```

---

## SafeheronConfig Interface

```typescript
import { SafeheronConfig } from '@safeheron/api-sdk';

const config: SafeheronConfig = {
  baseUrl: 'https://api.safeheron.vip',
  apiKey: 'your-api-key',
  rsaPrivateKey: 'your-pkcs8-private-key-pem-string',
  safeheronRsaPublicKey: 'safeheron-platform-public-key-pem-string',
  requestTimeout: 20000,  // milliseconds
};
```

### Config Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `baseUrl` | `string` | Safeheron API base URL |
| `apiKey` | `string` | API Key from Safeheron Console |
| `rsaPrivateKey` | `string` | Your RSA private key, PEM string or `file:/path/to/key.pem` |
| `safeheronRsaPublicKey` | `string` | Safeheron platform public key, PEM string or `file:/path/to/key.pem` |
| `requestTimeout` | `number` | Request timeout in milliseconds |

### Two Ways to Provide RSA Keys

**Option A -- PEM string directly:**

```typescript
import { readFileSync } from 'fs';
import path from 'path';

const config: SafeheronConfig = {
  baseUrl: 'https://api.safeheron.vip',
  apiKey: process.env.SAFEHERON_API_KEY!,
  rsaPrivateKey: readFileSync(path.resolve('./keys/api_pkcs8.pem'), 'utf8'),
  safeheronRsaPublicKey: readFileSync(path.resolve('./keys/safeheron_public.pem'), 'utf8'),
  requestTimeout: 20000,
};
```

**Option B -- Environment variables:**

```typescript
const config: SafeheronConfig = {
  baseUrl: 'https://api.safeheron.vip',
  apiKey: process.env.SAFEHERON_API_KEY!,
  rsaPrivateKey: process.env.SAFEHERON_RSA_PRIVATE_KEY!,
  safeheronRsaPublicKey: process.env.SAFEHERON_PLATFORM_PUBLIC_KEY!,
  requestTimeout: 20000,
};
```

---

## Obtaining API Instances

Each API domain has its own class. Instantiate with the same `SafeheronConfig`:

```typescript
import { AccountApi } from '@safeheron/api-sdk';
import { TransactionApi } from '@safeheron/api-sdk';
import { CoinApi } from '@safeheron/api-sdk';
import { MPCSignApi } from '@safeheron/api-sdk';
import { Web3Api } from '@safeheron/api-sdk';
import { WhitelistApi } from '@safeheron/api-sdk';
import { WebhookApi } from '@safeheron/api-sdk';
import { ComplianceApi } from '@safeheron/api-sdk';
import { GasApi } from '@safeheron/api-sdk';
import { ToolsApi } from '@safeheron/api-sdk';

const accountApi     = new AccountApi(config);
const transactionApi = new TransactionApi(config);
const coinApi        = new CoinApi(config);
const mpcSignApi     = new MPCSignApi(config);
const web3Api        = new Web3Api(config);
const whitelistApi   = new WhitelistApi(config);
const webhookApi     = new WebhookApi(config);
const complianceApi  = new ComplianceApi(config);
const gasApi         = new GasApi(config);
const toolsApi       = new ToolsApi(config);
```

## Executing Calls

All API methods return Promises. Use `async/await`:

```typescript
try {
  const result = await accountApi.createAccount({ accountName: 'my-wallet' });
  console.log('accountKey:', result.accountKey);
} catch (e) {
  if (e instanceof SafeheronError) {
    console.error(`Error code: ${e.code}, message: ${e.message}`);
  } else {
    console.error(e);
  }
}
```

---

## Key Generation (OpenSSL)

```bash
# Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Export public key (upload this to Safeheron Console)
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Convert to PKCS8 (required for SDK's rsaPrivateKey field)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
```

Use the **full PEM content** of `api_pkcs8.pem` (including `-----BEGIN/END-----` headers) as `rsaPrivateKey`.
Upload `api_public.pem` contents as your RSA public key in Safeheron Console.

---

## Webhook & Co-Signer Configurations

### WebhookConverter

```typescript
import { WebHookConverter } from '@safeheron/api-sdk';
import { SafeheronWebHookConfig } from '@safeheron/api-sdk';

const webhookConfig: SafeheronWebHookConfig = {
  webHookRsaPrivateKey: 'your-webhook-rsa-private-key-pem',
  safeheronWebHookRsaPublicKey: 'safeheron-webhook-rsa-public-key-pem',
};

const converter = new WebHookConverter(webhookConfig);
```

### CoSignerConverter

```typescript
import { CoSignerConverter } from '@safeheron/api-sdk';
import { SafeheronCoSignerConfig } from '@safeheron/api-sdk';

const cosignerConfig: SafeheronCoSignerConfig = {
  approvalCallbackServicePrivateKey: 'your-callback-private-key-pem',
  coSignerPubKey: 'cosigner-identity-public-key-pem',
};

const cosignerConverter = new CoSignerConverter(cosignerConfig);
```
