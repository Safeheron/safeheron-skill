# Whitelist API Reference

## Imports

```java
import com.safeheron.client.api.WhitelistApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
WhitelistApiService whitelistApi = ServiceCreator.create(WhitelistApiService.class, safeheronConfig);
```

---

## Create a Whitelist Entry

```java
CreateWhitelistRequest req = new CreateWhitelistRequest();
req.setWhitelistName("Alice's ETH address");   // max 20 chars
req.setChainType("EVM");                        // see chainType values below
req.setAddress("0xAliceEthAddress...");
// req.setMemo("optional memo");               // for TON networks (max 20 chars)
// req.setHiddenOnUI(false);                   // true = API-only, hidden in Console

CreateWhitelistResponse resp = ServiceExecutor.execute(whitelistApi.createWhitelist(req));
String whitelistKey = resp.getWhitelistKey();   // save this — permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | String | Display name (max 20 chars) |
| `chainType` | Yes | String | Blockchain type (see table below) |
| `address` | Yes | String | Blockchain address to whitelist |
| `memo` | No | String | Memo/tag for TON networks (max 20 chars) |
| `hiddenOnUI` | No | Boolean | If true, hidden from Web Console / App |

### Supported chainType Values

| `chainType` | Supported Networks |
|-------------|-------------------|
| `EVM` | Ethereum, BNB Chain, Polygon, Arbitrum, Base, and all EVM-compatible chains |
| `Bitcoin` | Bitcoin mainnet |
| `Bitcoin Cash` | Bitcoin Cash |
| `Dash` | Dash |
| `TRON` | TRON mainnet |
| `NEAR` | NEAR Protocol |
| `Filecoin` | Filecoin |
| `Sui` | Sui |
| `Aptos` | Aptos |
| `Solana` | Solana |
| `Bitcoin Testnet` | Bitcoin testnet |
| `TON` | TON mainnet |
| `TON_TESTNET` | TON testnet |

---

## Retrieve a Single Whitelist Entry

```java
OneWhitelistRequest req = new OneWhitelistRequest();
req.setWhitelistKey(whitelistKey);

WhitelistResponse resp = ServiceExecutor.execute(whitelistApi.oneWhitelist(req));
System.out.println("Name: " + resp.getWhitelistName());
System.out.println("Address: " + resp.getAddress());
System.out.println("Status: " + resp.getWhitelistStatus());
```

---

## List Whitelist Entries

```java
ListWhitelistRequest req = new ListWhitelistRequest();
req.setLimit(20L);

List<WhitelistResponse> list = ServiceExecutor.execute(whitelistApi.listWhitelist(req));
for (WhitelistResponse entry : list) {
    System.out.println(entry.getWhitelistKey() + " " + entry.getWhitelistName()
                       + " " + entry.getWhitelistStatus());
}
```

---

## Edit a Whitelist Entry

```java
EditWhitelistRequest req = new EditWhitelistRequest();
req.setWhitelistKey(whitelistKey);
req.setWhitelistName("Updated name");
// req.setAddress("0xNewAddress");  // optional

ServiceExecutor.execute(whitelistApi.editWhitelist(req));
```

---

## Delete a Whitelist Entry

```java
DeleteWhitelistRequest req = new DeleteWhitelistRequest();
req.setWhitelistKey(whitelistKey);

ServiceExecutor.execute(whitelistApi.deleteWhitelist(req));
```

---

## Create Whitelist from an Existing Transaction

Automatically creates a whitelist entry using the destination address of a completed transaction:

```java
CreateFromTransactionWhitelistRequest req = new CreateFromTransactionWhitelistRequest();
req.setTxKey(txKey);
req.setWhitelistName("Recipient from TX-001");

CreateWhitelistResponse resp = ServiceExecutor.execute(
        whitelistApi.createFromTransactionWhitelist(req));
String whitelistKey = resp.getWhitelistKey();
```

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `whitelistKey` | String | Unique identifier |
| `whitelistName` | String | Display name |
| `chainType` | String | Blockchain type |
| `address` | String | Whitelisted address |
| `memo` | String | Memo (TON networks) |
| `whitelistStatus` | String | `AUDIT` / `APPROVED` / `REJECTED` |
| `createTime` | Long | Unix timestamp (ms) |
| `lastUpdateTime` | Long | Unix timestamp (ms) |

### Whitelist Status Values

| Status | Description |
|--------|-------------|
| `AUDIT` | Pending approval (if policy requires human review) |
| `APPROVED` | Active — can be used as transaction destination |
| `REJECTED` | Denied — cannot be used |

---

## Using Whitelist as Transaction Destination

```java
CreateTransactionRequest txReq = new CreateTransactionRequest();
txReq.setDestinationAccountType("WHITELISTING_ACCOUNT");
txReq.setDestinationAccountKey(whitelistKey);  // pass whitelistKey here
txReq.setCoinKey("ETHEREUM_ETH");
txReq.setTxAmount("0.05");
// ... other fields
```

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create | `CreateWhitelistRequest` | `CreateWhitelistResponse` |
| Create from TX | `CreateFromTransactionWhitelistRequest` | `CreateWhitelistResponse` |
| Get one | `OneWhitelistRequest` | `WhitelistResponse` |
| List | `ListWhitelistRequest` | `List<WhitelistResponse>` |
| Edit | `EditWhitelistRequest` | `ResultResponse` |
| Delete | `DeleteWhitelistRequest` | `ResultResponse` |

---

## Best Practices

- Validate address with `CoinApiService.checkCoinAddress()` before creating a whitelist entry.
- Use descriptive `whitelistName` and set `chainType` carefully — wrong chainType causes address validation failure.
- In regulated environments, configure Safeheron Console policy to require human approval before whitelist additions take effect (`whitelistStatus = "AUDIT"` until approved).
- Subscribe to Whitelist webhook events (`WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`) to keep your local state in sync.
