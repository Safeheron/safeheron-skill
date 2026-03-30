# Java SDK Setup & Configuration

## Maven
```xml
<dependency>
    <groupId>com.safeheron</groupId>
    <artifactId>api-sdk-java</artifactId>
    <version>1.0.10</version>
</dependency>
```

## Clone and Build Locally
```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-java.git
cd safeheron-api-sdk-java
mvn install -Dmaven.test.skip=true
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

## Loading Config Programmatically (as in official test code)
```java
Yaml yaml = new Yaml();
File file = new File("src/test/resources/demo/api/account/config.yaml");
InputStream inputStream = new FileInputStream(file);
Map<String, Object> config = yaml.load(inputStream);

SafeheronConfig safeheronConfig = SafeheronConfig.builder()
        .baseUrl(config.get("baseUrl").toString())
        .apiKey(config.get("apiKey").toString())
        .safeheronRsaPublicKey(config.get("safeheronPublicKey").toString())
        .rsaPrivateKey(config.get("privateKey").toString())
        .requestTimeout(Long.valueOf(config.get("requestTimeout").toString()))
        .build();
```

## ⚠️ SafeheronConfig Builder Field Names
| Field | Builder method | Notes |
|-------|---------------|-------|
| API Base URL | `.baseUrl(String)` | |
| API Key | `.apiKey(String)` | From Safeheron Console |
| Your RSA private key (base64) | `.rsaPrivateKey(String)` | PKCS8 encoded |
| Safeheron platform public key (base64) | `.safeheronRsaPublicKey(String)` | From Safeheron Console |
| Timeout | `.requestTimeout(Long)` | Milliseconds, must be Long |

---

## Obtaining API Service Instances
```java
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.api.TransactionApiService;
import com.safeheron.client.api.MPCSignApiService;
import com.safeheron.client.api.Web3ApiService;
import com.safeheron.client.api.WhitelistApiService;
import com.safeheron.client.utils.ServiceCreator;

AccountApiService     accountApi     = ServiceCreator.create(AccountApiService.class,     config);
TransactionApiService transactionApi = ServiceCreator.create(TransactionApiService.class, config);
MPCSignApiService     mpcSignApi     = ServiceCreator.create(MPCSignApiService.class,     config);
Web3ApiService        web3Api        = ServiceCreator.create(Web3ApiService.class,        config);
WhitelistApiService   whitelistApi   = ServiceCreator.create(WhitelistApiService.class,   config);
```

## Executing Calls
Every API call **must** be wrapped with `ServiceExecutor.execute(...)`:
```java
import com.safeheron.client.utils.ServiceExecutor;

// Correct
CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));

// WRONG — never call the interface method directly without ServiceExecutor
// CreateAccountResponse resp = accountApi.createAccount(req);  // ❌
```

---

## Spring Boot Integration
```java
@Configuration
public class SafeheronConfiguration {

    @Bean
    public SafeheronConfig safeheronConfig(
            @Value("${safeheron.baseUrl}") String baseUrl,
            @Value("${safeheron.apiKey}") String apiKey,
            @Value("${safeheron.rsaPrivateKey}") String rsaPrivateKey,
            @Value("${safeheron.safeheronRsaPublicKey}") String safeheronRsaPublicKey) {
        return SafeheronConfig.builder()
                .baseUrl(baseUrl)
                .apiKey(apiKey)
                .rsaPrivateKey(rsaPrivateKey)
                .safeheronRsaPublicKey(safeheronRsaPublicKey)
                .requestTimeout(20000L)
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
}
```

```yaml
# application.yml
safeheron:
  baseUrl: https://api.safeheron.vip
  apiKey: ${SAFEHERON_API_KEY}
  rsaPrivateKey: ${SAFEHERON_RSA_PRIVATE_KEY}
  safeheronRsaPublicKey: ${SAFEHERON_PLATFORM_PUBLIC_KEY}
```

---

## Key Generation (OpenSSL)
```bash
# Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Export public key
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Convert to PKCS8 (required for SDK's rsaPrivateKey field)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
```
Use the base64 content of `api_pkcs8.pem` (strip `-----BEGIN/END-----` headers) as `rsaPrivateKey`.
Upload `api_public.pem` contents (strip headers) as your RSA public key in Safeheron Console.
