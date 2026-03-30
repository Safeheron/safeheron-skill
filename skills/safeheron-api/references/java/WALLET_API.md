# Wallet Account API Reference

## Imports

```java
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
AccountApiService accountApi = ServiceCreator.create(AccountApiService.class, safeheronConfig);
```

---

## Create a Wallet Account

```java
CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("my-wallet-account");
req.setHiddenOnUI(false);  // true = API-only wallet, hidden in Console

CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));
// ✅ Correct getters — only these three exist on CreateAccountResponse
String accountKey = resp.getAccountKey();          // save this — permanent wallet identifier
List<CreateAccountResponse.PubKey> pubKeys = resp.getPubKeys();
List<CreateAccountResponse.CoinAddress> coins = resp.getCoinAddressList();
```

**CreateAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `accountName` | Yes | String | Wallet display name |
| `customerRefId` | No | String | Merchant unique business ID (max 100 chars). Duplicate submissions with the same ID return the same wallet |
| `hiddenOnUI` | No | Boolean | If true, wallet is hidden in Web Console / App |
| `autoFuel` | No | Boolean | If true, Gas Station auto-tops up gas when a transaction is initiated. Default: false |
| `accountTag` | No | String | Tag applied at creation. Values: `DEPOSIT`, `NONE`. Required for Auto-Sweep |
| `coinKeyList` | No | List\<String\> | Coin keys to add at creation (max 20) |

**CreateAccountResponse Fields:**

| Field | Type |  Description |
|-------|------|-------------|
| `accountKey` | String | Unique wallet identifier (save permanently) |
| `pubKeys` | `List<PubKey>` | Account public key list (signAlg, pubKey) |
| `coinAddressList` | `List<CoinAddress>` | Coin address list per coin |

---

## List Wallet Accounts

```java
ListAccountRequest req = new ListAccountRequest();
req.setPageSize(10L);   // Long, not int
req.setPageNumber(1L);  // Long, not int — 1-indexed

PageResult<AccountResponse> result = ServiceExecutor.execute(accountApi.listAccounts(req));
List<AccountResponse> accounts = result.getContent();
long total = result.getTotalElements();

for (AccountResponse acct : accounts) {
    System.out.println(acct.getAccountKey() + " " + acct.getAccountName());
}
```

**ListAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `pageSize` | No | Long | Items per page (default 10) |
| `pageNumber` | No | Long | Page number, 1-indexed |

**PageResult Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `pageNumber` | Long | Current page number (1-indexed) |
| `pageSize` | Long | Number of items per page |
| `totalElements` | Long | Total number of records across all pages |
| `content` | `List<T>` | Records on the current page |

**AccountResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `accountKey` | String | Unique wallet identifier |
| `accountName` | String | Wallet display name |
| `customerRefId` | String | Your custom reference ID |
| `accountIndex` | Long | Derivation path index |
| `accountType` | String | `VAULT_ACCOUNT` |
| `accountTag` | String | Tag: `DEPOSIT`, `NONE`, etc. |
| `hiddenOnUI` | Boolean | Hidden from UI |
| `autoFuel` | Boolean | Auto gas refill enabled |
| `archived` | Boolean | Whether the account is archived |
| `usdBalance` | String | Total USD balance across all coins |
| `pubKeys` | `List<PubKey>` | MPC public key shards |

**PubKey Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `signAlg` | String | Signature algorithm (e.g. `ECDSA`) |
| `pubKey` | String | Public key value |

---

## Get a Single Wallet Account

```java
OneAccountRequest req = new OneAccountRequest();
req.setAccountKey(accountKey);

AccountResponse resp = ServiceExecutor.execute(accountApi.oneAccounts(req));
```

> ⚠️ **The method is `oneAccounts` (with 's'), NOT `oneAccount`. Do not use `oneAccount` — it does not exist.**

---

## Batch Label Wallet Accounts

```java
BatchUpdateAccountTagRequest req = new BatchUpdateAccountTagRequest();
req.setAccountKeyList(Arrays.asList("your-account-key"));
req.setAccountTag("NONE");  // // optional: "DEPOSIT" or "NONE"
ServiceExecutor.execute(accountApi.batchUpdateAccountTag(req));
```

**AccountTag Values:**

| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet — eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

> Cancelling `DEPOSIT` label: set `accountTag = "NONE"`. Can be re-applied later.

---

## Add Coin to a Wallet Account (V2 — Recommended)

Add up to 20 coins in a single call. Adding a token auto-adds its mainnet coin.

```java
CreateAccountCoinV2Request req = new CreateAccountCoinV2Request();
req.setAccountKey(accountKey);
req.setCoinKeyList(Arrays.asList(
    "ETHEREUM_ETH",
    "USDT(ERC20)_ETHEREUM_USDT",
    "BITCOIN_BTC"
));

CreateAccountCoinV2Response res = ServiceExecutor.execute(accountApi.createAccountCoinV2(req));
for (CreateAccountCoinV2Response.CoinAddress coin : res.getCoinAddressList()) {
    System.out.println("  Added coin: " + coin.getCoinKey() + " addressList: " + coin.getAddressList());
}
```

> **V1 (single coin):** `CreateAccountCoinRequest` with `setCoinKey(String)` — still works, returns `List<CreateAccountCoinResponse>`.

**CreateAccountCoinV2Response Fields:**

| Field | Type |  Description |
|-------|------|-------------|
| `accountKey` | String | Wallet account key |
| `coinAddressList` | `List<CoinAddress>` | Per-coin address info |

**CreateAccountCoinV2Response.CoinAddress Fields:**

| Field | Type |  Description |
|-------|------|-------------|
| `coinKey` | String | Coin identifier |
| `addressGroupKey` | String | Address group key |
| `addressGroupName` | String | Address group name |
| `addressList` | `List<Address>` | Address list |

**CreateAccountCoinResponse Fields (V1):**

| Field | Type |  Description |
|-------|------|-------------|
| `address` | String | Coin receiving address |
| `addressType` | String | Address type |
| `derivePath` | String | BIP44 derivation path |

> ⚠️ **`CreateAccountCoinResponse` (V1) does NOT have `getCoinKey()`. Use `CreateAccountCoinV2Response` if you need coin key info.**

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

Query balances and deposit addresses per coin for a specific account:

```java
ListAccountCoinRequest req = new ListAccountCoinRequest();
req.setAccountKey(accountKey);

List<AccountCoinResponse> coins = ServiceExecutor.execute(accountApi.listAccountCoin(req));
for (AccountCoinResponse coin : coins) {
    System.out.printf("%-30s balance: %-20s address: %s%n",
        coin.getCoinKey(), coin.getBalance(), coin.getAddressList().get(0).getAddress());
}
```

> Also queryable via: `GET /v1/account/coin/list`

**AccountCoinResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | String | Coin identifier (e.g. `ETHEREUM_ETH`) |
| `coinFullName` | String | Full coin name (e.g. `Ethereum`) |
| `coinName` | String | Coin symbol (e.g. `ETH`) |
| `symbol` | String | Coin unit display name |
| `coinDecimal` | Long | Coin decimal places |
| `showCoinDecimal` | Long | Displayed decimal places on Console |
| `feeCoinKey` | String | Fee coin identifier (e.g. ETH for ERC-20 tokens) |
| `feeUnit` | String | Fee unit name (e.g. `Gwei`, `satoshis`) |
| `feeDecimal` | Long | Fee decimal places |
| `balance` | String | Account balance |
| `usdBalance` | String | Balance converted to USD |
| `addressList` | `List<AddressResult>` | Coin deposit address list |
| `isMultipleAddress` | String | Whether multiple address groups are supported (`Yes`/`No`) |
| `txRefUrl` | String | Transaction explorer URL template |
| `addressRefUrl` | String | Address explorer URL |
| `logoUrl` | String | Coin logo URL |

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create account | `CreateAccountRequest` | `CreateAccountResponse` |
| List accounts | `ListAccountRequest` | `PageResult<AccountResponse>` |
| Get one account | `OneAccountRequest` | `AccountResponse` |
| Change display in Console/App | `UpdateAccountShowStateRequest` | *(void / ResultResponse)* |
| Add coin V2 | `CreateAccountCoinV2Request` | `CreateAccountCoinV2Response` |
| Add coin V1 | `CreateAccountCoinRequest` | `List<CreateAccountCoinResponse>` |
| List coins | `ListAccountCoinRequest` | `List<AccountCoinResponse>` |

---

## Best Practices

- `pageSize` and `pageNumber` fields are **Long** type, not int.
- `accountKey` is the permanent, immutable identifier for a wallet — store it after creation.
- Web3 wallets use a separate set of APIs — see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
