# Whitelist API Reference

## Imports

```java
import com.safeheron.client.api.WhitelistApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;

import java.util.ArrayList;
import java.util.List;
```

## Create API Service

```java
WhitelistApiService whitelistApi = ServiceCreator.create(WhitelistApiService.class, safeheronConfig);
```

---

## Create a Whitelist Entry

```java
CreateWhitelistRequest req = new CreateWhitelistRequest();
req.setWhitelistName("Alice's ETH address");   // unique, max 30 chars
req.setChainType("EVM");                        // see chainType values below
req.setAddress("0xAliceEthAddress...");
// req.setMemo("12345");                       // TON / XRP Ledger only (max 20 chars; XRPL: numeric 0–4294967295)
// req.setHiddenOnUI(false);                   // true = API-only, hidden in Console
// req.setNote("Alice - Binance deposit");     // optional note

CreateWhitelistResponse resp = ServiceExecutor.execute(whitelistApi.createWhitelist(req));
String whitelistKey = resp.getWhitelistKey();   // save this — permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | String | Whitelist unique name, 30 characters max |
| `chainType` | Yes | String | Blockchain type (see table below) |
| `address` | Yes | String | Blockchain address to whitelist; the format must meet the chain's requirements |
| `memo` | No | String | Memo / tag (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295). Only valid for chainType `TON` and `XRP Ledger` |
| `hiddenOnUI` | No | Boolean | If true, hidden from Web Console / App (manageable via API only) |
| `note` | No | String | Note |

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
| `Litecoin` | Litecoin mainnet |
| `Dogecoin` | Dogecoin mainnet |
| `Liquid` | Liquid mainnet |
| `XRP Ledger` | XRP Ledger mainnet (native XRP only, no other tokens yet) |

---

## Retrieve a Single Whitelist Entry

```java
OneWhitelistRequest req = new OneWhitelistRequest();
req.setWhitelistKey(whitelistKey);   // or req.setAddress("0x...") — whitelistKey takes precedence if both are set

WhitelistResponse resp = ServiceExecutor.execute(whitelistApi.oneWhitelist(req));
System.out.println("Name: " + resp.getWhitelistName());
System.out.println("Address: " + resp.getAddress());
System.out.println("Status: " + resp.getWhitelistStatus());
```

---

## List Whitelist Entries

```java
ListWhitelistRequest req = new ListWhitelistRequest();
// Published docs mark chainType as required, but the service treats it as an optional filter —
// passing it returns only that chain; omit it to list all chains.
// req.setChainType("EVM");
// req.setWhitelistStatus("APPROVED"); // optional filter: AUDIT / APPROVED / REJECTED
// createTimeMin defaults to createTimeMax − 24 hours and createTimeMax defaults to now —
// without createTimeMin you only get whitelists created in the last 24 hours
req.setCreateTimeMin(1L);
req.setLimit(500L);                   // max 500 per page

List<WhitelistResponse> list = ServiceExecutor.execute(whitelistApi.listWhitelist(req));
for (WhitelistResponse entry : list) {
    System.out.println(entry.getWhitelistKey() + " " + entry.getWhitelistName()
                       + " " + entry.getWhitelistStatus());
}
```

> **Time window & paging:** `createTimeMin` defaults to `createTimeMax` minus 24 hours (`createTimeMax` defaults to now), so a request without time bounds returns only the last 24 hours. Results are cursor-paged: `direct` (`NEXT` by default) + `fromId` = `whitelistKey` of the last record of the previous page; `limit` max 500.

### List ALL Whitelist Entries (all chains, all time)

```java
public List<WhitelistResponse> listAllWhitelists(WhitelistApiService whitelistApi) {
    final long pageSize = 500L;
    List<WhitelistResponse> all = new ArrayList<>();
    String fromId = null;
    while (true) {
        ListWhitelistRequest req = new ListWhitelistRequest();
        req.setCreateTimeMin(1L);          // override the 24h default window
        req.setDirect("NEXT");
        req.setLimit(pageSize);
        req.setFromId(fromId);             // null for the first page
        List<WhitelistResponse> page = ServiceExecutor.execute(whitelistApi.listWhitelist(req));
        if (page == null || page.isEmpty()) {
            break;
        }
        all.addAll(page);
        if (page.size() < pageSize) {
            break;
        }
        fromId = page.get(page.size() - 1).getWhitelistKey();
    }
    return all;
}
```

---

## Edit a Whitelist Entry

```java
EditWhitelistRequest req = new EditWhitelistRequest();
req.setWhitelistKey(whitelistKey);
req.setWhitelistName("Updated name");   // required
req.setAddress("0xNewAddress");         // required
// req.setNote("updated note");         // optional — like memo, note overwrites the stored value;
//                                       // pass the current note to keep it

ServiceExecutor.execute(whitelistApi.editWhitelist(req));
```

### EditWhitelistRequest Fields

| Field | Required | Type    | Description |
|-------|----------|---------|-------------|
| `whitelistKey` | Yes | String  | Unique identifier of the whitelist entry to edit |
| `whitelistName` | Yes | String  | Whitelist unique name, 30 characters max |
| `address` | Yes | String  | Public blockchain address and the address format needs to meet the requirements of the chain |
| `memo` | No | String  | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the public blockchain address, also known as a comment or tag. Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `force` | No | Boolean | When the whitelist is involved in a transaction approval policy, modifications will result in the new whitelist being directly applied to the approval policy. False by default, meaning that when involved in a transaction approval policy, it will not be modified |
| `note` | No | String  | Note |
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
req.setTxKey(txKey);                                  // required
req.setWhitelistName("Recipient from TX-001");        // required, unique, max 30 chars
req.setDestinationAddress(destinationAddress);        // required — the destination address in the transaction record (case-sensitive)
// req.setMemo("12345");      // TON / XRP Ledger: if the transaction used a memo, the same memo must be provided
// req.setHiddenOnUI(false);
// req.setNote("optional note");

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
| `memo` | String | Memo of the address when creating the whitelist (TON / XRP Ledger) |
| `whitelistStatus` | String | `AUDIT` / `APPROVED` / `REJECTED` |
| `note` | String | Note |
| `createTime` | Long | Unix timestamp (ms) |
| `lastUpdateTime` | Long | Unix timestamp (ms) |

> The API also returns `hiddenOnUI`, but `WhitelistResponse` in SDK 1.0.15 does not have this field.

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
