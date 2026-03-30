# Getting Started: From Zero to First API Call

## Overview

This guide walks through the complete setup to make your first Safeheron API call using the Java SDK.

**Prerequisites:**
- Java 8+
- Maven 3.x or Gradle 7+
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

# Step 1c: Convert private key to PKCS8 format (required by the Java SDK)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt \
    -in api_private.pem -out api_pkcs8.pem
```

After running these commands, you will have:

| File | Purpose | Used by |
|------|---------|---------|
| `api_private.pem` | Original private key | Intermediate only |
| `api_public.pem` | Public key | **Upload to Safeheron Console** |
| `api_pkcs8.pem` | PKCS8-encoded private key | **Set in SDK config as `rsaPrivateKey`** |

### Extract the Base64 Values

The SDK requires the **raw base64 content** without PEM headers/footers.

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

1. Go to **Settings → API** in the Web Console.
2. Copy the **"Safeheron Public Key"** (base64 string).
   This is the `safeheronRsaPublicKey` value in your SDK config.

### 2-2. Create an API Key

1. Go to **Settings → API → API Keys → Create API Key**.
2. Fill in:
   - **Name**: any descriptive label
   - **RSA Public Key**: paste the base64 content from `api_public.pem` (no headers)
   - **Permissions**: select the permissions your application needs (e.g., Initiate Transaction, API Management)
   - **IP Whitelist**: add your server's IP address(es) — **required**, IPv4 and IPv6 both supported
3. Submit — an **API Key** string is generated. Copy and save it.

> ⚠️ **IP Whitelist is mandatory.** If your server IP is not registered, all API calls will be rejected with "Illegal IP".

### 2-3. Configure Webhook (Optional for First Call)

Go to **Settings → API → Webhook** to configure a callback URL for real-time transaction notifications.
See [WEBHOOK.md](WEBHOOK.md) for details.

### 2-4. Summary of Values Needed

After completing Console setup, you should have:

| Config Item | Where to Get | SDK Field |
|-------------|-------------|-----------|
| API Key string | Created in Console | `.apiKey(...)` |
| Safeheron Platform Public Key | Copied from Console | `.safeheronRsaPublicKey(...)` |
| Your RSA Private Key (PKCS8 base64) | From `api_pkcs8.pem` | `.rsaPrivateKey(...)` |

---

## Step 3 -- Add SDK to Project Dependencies

### Maven

Add to `pom.xml`:

```xml
<dependency>
    <groupId>com.safeheron</groupId>
    <artifactId>api-sdk-java</artifactId>
    <version>1.0.10</version>
</dependency>
```

If using SnakeYAML for config file loading (optional):

```xml
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>2.2</version>
</dependency>
```

### Gradle

Add to `build.gradle`:

```groovy
dependencies {
    implementation 'com.safeheron:api-sdk-java:1.0.10'
}
```

Or for `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.safeheron:api-sdk-java:1.0.10")
}
```

### Build from Source (Optional)

```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-java.git
cd safeheron-api-sdk-java
mvn install -Dmaven.test.skip=true
```

> SDK updates are **backward compatible** — new versions only add methods, existing APIs are preserved.

---

## Step 4 -- Configure Environment & Inject Credentials

**Never hardcode credentials in source code.**

Choose one of the following approaches based on your project type.

### Option A — Environment Variables (Recommended)

Set environment variables on your server or in your CI/CD pipeline:

```bash
export SAFEHERON_API_KEY="your-api-key-here"
export SAFEHERON_RSA_PRIVATE_KEY="MIIJQgIBADANBgkqhkiG9w0BAQEFAASC..."
export SAFEHERON_PLATFORM_PUBLIC_KEY="MIICIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."
```

Load in Java:

```java
SafeheronConfig config = SafeheronConfig.builder()
    .baseUrl("https://api.safeheron.vip")
    .apiKey(System.getenv("SAFEHERON_API_KEY"))
    .rsaPrivateKey(System.getenv("SAFEHERON_RSA_PRIVATE_KEY"))
    .safeheronRsaPublicKey(System.getenv("SAFEHERON_PLATFORM_PUBLIC_KEY"))
    .requestTimeout(20000L)
    .build();
```

### Option B — Spring Boot application.yml

`application.yml`:

```yaml
safeheron:
  baseUrl: https://api.safeheron.vip
  apiKey: ${SAFEHERON_API_KEY}
  rsaPrivateKey: ${SAFEHERON_RSA_PRIVATE_KEY}
  safeheronRsaPublicKey: ${SAFEHERON_PLATFORM_PUBLIC_KEY}
  requestTimeout: 20000
```

Spring Boot configuration class:

```java
@Configuration
public class SafeheronConfiguration {

    @Bean
    public SafeheronConfig safeheronConfig(
            @Value("${safeheron.baseUrl}") String baseUrl,
            @Value("${safeheron.apiKey}") String apiKey,
            @Value("${safeheron.rsaPrivateKey}") String rsaPrivateKey,
            @Value("${safeheron.safeheronRsaPublicKey}") String safeheronRsaPublicKey,
            @Value("${safeheron.requestTimeout:20000}") long requestTimeout) {
        return SafeheronConfig.builder()
                .baseUrl(baseUrl)
                .apiKey(apiKey)
                .rsaPrivateKey(rsaPrivateKey)
                .safeheronRsaPublicKey(safeheronRsaPublicKey)
                .requestTimeout(requestTimeout)
                .build();
    }

    @Bean
    public AccountApiService accountApiService(SafeheronConfig config) {
        return ServiceCreator.create(AccountApiService.class, config);
    }

    @Bean
    public TransactionApiService transactionApiService(SafeheronConfig config) {
        return ServiceCreator.create(TransactionApiService.class, config);
    }

    @Bean
    public MPCSignApiService mpcSignApiService(SafeheronConfig config) {
        return ServiceCreator.create(MPCSignApiService.class, config);
    }

    @Bean
    public Web3ApiService web3ApiService(SafeheronConfig config) {
        return ServiceCreator.create(Web3ApiService.class, config);
    }

    @Bean
    public WhitelistApiService whitelistApiService(SafeheronConfig config) {
        return ServiceCreator.create(WhitelistApiService.class, config);
    }

    @Bean
    public CoinApiService coinApiService(SafeheronConfig config) {
        return ServiceCreator.create(CoinApiService.class, config);
    }
}
```

### Option C — Local Config File (Development Only)

Create `config.yaml` outside your source tree (never committed):

```yaml
apiKey: "your-api-key"
privateKey: "MIIJQgIBADANBgkqhkiG9w0BAQEFAASC..."    # PKCS8 base64, no headers
safeheronPublicKey: "MIICIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."  # Platform public key
baseUrl: "https://api.safeheron.vip"
requestTimeout: 20000
```

Load it:

```java
import org.yaml.snakeyaml.Yaml;
import java.io.FileInputStream;
import java.util.Map;

Yaml yaml = new Yaml();
Map<String, Object> cfg = yaml.load(new FileInputStream("config.yaml"));

SafeheronConfig config = SafeheronConfig.builder()
        .baseUrl(cfg.get("baseUrl").toString())
        .apiKey(cfg.get("apiKey").toString())
        .safeheronRsaPublicKey(cfg.get("safeheronPublicKey").toString())
        .rsaPrivateKey(cfg.get("privateKey").toString())
        .requestTimeout(Long.valueOf(cfg.get("requestTimeout").toString()))
        .build();
```

Add to `.gitignore`:
```
config.yaml
*.pem
```

### ⚠️ SafeheronConfig Builder — Common Mistakes

| ❌ Wrong | ✅ Correct |
|---------|----------|
| `.rsaPrivateKey(safeheronPublicKey)` | `.rsaPrivateKey(yourPKCS8PrivateKey)` |
| `.safeheronRsaPublicKey(yourPublicKey)` | `.safeheronRsaPublicKey(platformPublicKey)` |
| `.requestTimeout(20000)` (int) | `.requestTimeout(20000L)` (Long) |
| Private key with `-----BEGIN...` headers | Strip headers, raw base64 only |
| Private key in PKCS1 format | Must be **PKCS8** (`openssl pkcs8 ...`) |

---

## Step 5 -- Your First API Request

Below is a complete, runnable example: create a wallet, add ETH and USDT, then list accounts.

```java
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.CreateAccountCoinV2Request;
import com.safeheron.client.request.CreateAccountRequest;
import com.safeheron.client.request.ListAccountRequest;
import com.safeheron.client.response.AccountResponse;
import com.safeheron.client.response.CreateAccountCoinResponse;
import com.safeheron.client.response.CreateAccountResponse;
import com.safeheron.client.response.PageResult;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;

import java.util.Arrays;
import java.util.List;

public class SafeheronQuickStart {

    public static void main(String[] args) throws Exception {

        // ── Step 1: Build config ──────────────────────────────────────────
        SafeheronConfig config = SafeheronConfig.builder()
                .baseUrl("https://api.safeheron.vip")
                .apiKey(System.getenv("SAFEHERON_API_KEY"))
                .rsaPrivateKey(System.getenv("SAFEHERON_RSA_PRIVATE_KEY"))
                .safeheronRsaPublicKey(System.getenv("SAFEHERON_PLATFORM_PUBLIC_KEY"))
                .requestTimeout(20000L)
                .build();

        // ── Step 2: Create service instance ──────────────────────────────
        AccountApiService accountApi = ServiceCreator.create(AccountApiService.class, config);

        // ── Step 3: Create a wallet account ──────────────────────────────
        CreateAccountRequest createReq = new CreateAccountRequest();
        createReq.setAccountName("my-first-wallet");
        createReq.setHiddenOnUI(false);

        CreateAccountResponse createResp = ServiceExecutor.execute(
                accountApi.createAccount(createReq));

        String accountKey = createResp.getAccountKey();
        System.out.println("Wallet created. accountKey: " + accountKey);

        // ── Step 4: Add coins to the wallet ──────────────────────────────
        // V2 supports up to 20 coins at once.
        // Adding USDT(ERC20) automatically adds ETHEREUM_ETH as well.
        CreateAccountCoinV2Request coinReq = new CreateAccountCoinV2Request();
        coinReq.setAccountKey(accountKey);
        coinReq.setCoinKeyList(Arrays.asList(
                "ETHEREUM_ETH",
                "USDT(ERC20)_ETHEREUM_USDT"
        ));

        List<CreateAccountCoinResponse> coinResp = ServiceExecutor.execute(
                accountApi.createAccountCoinV2(coinReq));

        System.out.println("Coins added:");
        for (CreateAccountCoinResponse coin : coinResp) {
            System.out.println("  " + coin.getCoinKey() + " -> " + coin.getAddress());
        }

        // ── Step 5: List wallet accounts ─────────────────────────────────
        ListAccountRequest listReq = new ListAccountRequest();
        listReq.setPageSize(10L);
        listReq.setPageNumber(1L);

        PageResult<AccountResponse> listResp = ServiceExecutor.execute(
                accountApi.listAccounts(listReq));

        System.out.println("Total wallets: " + listResp.getTotalElements());
        for (AccountResponse acct : listResp.getContent()) {
            System.out.println("  " + acct.getAccountKey() + "  " + acct.getAccountName());
        }
    }
}
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
| `1012 Signature verification failed` | Wrong `rsaPrivateKey` or not PKCS8 format | Re-run `openssl pkcs8 ...`; ensure raw base64, no headers |
| `1010 Parameter decryption failed` | Wrong `safeheronRsaPublicKey` | Copy the **Safeheron Platform Public Key** from Console (not your own public key) |
| `Illegal IP` / connection rejected | Server IP not in API Key IP whitelist | Add server IP in Console → API Keys → IP Whitelist |
| `NullPointerException` on config | Environment variable not set | Verify `System.getenv(...)` returns non-null values |
| `requestTimeout` compile error | Passed `int` instead of `Long` | Use `20000L` not `20000` |

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
