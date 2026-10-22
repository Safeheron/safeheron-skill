# Whitelist API Reference

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
whitelistApi := api.WhitelistApi{Client: sc}
```

---

## Create a Whitelist Entry

```go
req := api.CreateWhitelistRequest{
    WhitelistName: "Alice's ETH address", // max 30 chars, unique
    ChainType:     "EVM",                 // see chainType values below
    Address:       "0xAliceEthAddress...",
    // Memo:       "optional memo",       // TON / XRP Ledger only (max 20 chars; XRPL: numeric string 0–4294967295)
    // HiddenOnUI: false,                 // true = API-only, hidden in Console
    // Note:       "optional note",
}

var resp api.CreateWhitelistResponse
if err := whitelistApi.CreateWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create whitelist: %w", err))
}
whitelistKey := resp.WhitelistKey // save this -- permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `WhitelistName` | Yes | `string` | Whitelist unique name, 30 characters max |
| `ChainType` | Yes | `string` | Blockchain type (see table below) |
| `Address` | Yes | `string` | Blockchain address to whitelist |
| `Memo` | No | `string` | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295), also known as a comment or tag. Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `HiddenOnUI` | No | `bool` | If true, hidden from Web Console / App (manageable via API only) |
| `Note` | No | `string` | Note (180 chars max per service-side validation; not stated in the published docs) |

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

Look up by `WhitelistKey` **or** by `Address` (one of them is required; if both are given, `WhitelistKey` takes precedence):

```go
req := api.OneWhitelistRequest{
    WhitelistKey: whitelistKey,
    // Address:   "0xAliceEthAddress...", // alternative lookup key
}

var resp api.WhitelistResponse
if err := whitelistApi.OneWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get whitelist: %w", err))
}
fmt.Printf("Name: %s, Address: %s, Status: %s\n",
    resp.WhitelistName, resp.Address, resp.WhitelistStatus)
```

---

## List Whitelist Entries

> **Time window and paging (published docs):** `CreateTimeMin` defaults to `CreateTimeMax` minus **24 hours** and `CreateTimeMax` defaults to now — a request without time bounds only returns whitelists created in the last 24 hours. `Limit` is at most 500. Paging is cursor-based: `Direct` (`NEXT` by default) + `FromId` = `whitelistKey` of the last record of the previous page.
>
> All `ListWhitelistRequest` fields are `omitempty` in Go SDK v1.0.15, so `CreateTimeMin: 0` is **not sent** (the 24h default applies) — use `1` for "from the beginning".

```go
// List ALL whitelist entries (all chains, all creation times)
func listAllWhitelists(whitelistApi api.WhitelistApi) ([]api.WhitelistResponse, error) {
    const pageSize = 500
    var all []api.WhitelistResponse
    fromId := ""
    for {
        req := api.ListWhitelistRequest{
            Direct:        "NEXT",
            Limit:         pageSize,
            FromId:        fromId,                 // "" on the first page (omitted)
            CreateTimeMin: 1,                      // not 0: 0 is dropped by omitempty → 24h default
            CreateTimeMax: time.Now().UnixMilli(),
            // ChainType: "EVM", // Published docs mark chainType as required, but the service treats it as an optional
            //                   // filter — passing it returns only that chain; omit it to list all chains.
        }
        var page []api.WhitelistResponse
        if err := whitelistApi.ListWhitelist(req, &page); err != nil {
            return nil, fmt.Errorf("failed to list whitelist: %w", err)
        }
        all = append(all, page...)
        if len(page) < pageSize {
            return all, nil
        }
        fromId = page[len(page)-1].WhitelistKey
    }
}
```

Imports used: `fmt`, `time`.

---

## Edit a Whitelist Entry

```go
req := api.EditWhitelistRequest{
    WhitelistKey:  whitelistKey,
    WhitelistName: "Updated name",           // required
    Address:       "0xAliceEthAddress...",   // required — pass the (new or unchanged) address
    // Memo:       "...",                    // TON / XRP Ledger only
    // Force:      true,                     // also apply to approval policies that reference this whitelist
    // Note:       "updated note",       // note (like memo) overwrites the stored value — pass the current note to keep it
}

var resp api.ResultResponse
if err := whitelistApi.EditWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to edit whitelist: %w", err))
}
```

### EditWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `WhitelistKey` | Yes | `string` | Unique identifier of the whitelist entry to edit |
| `WhitelistName` | Yes | `string` | Whitelist unique name, 30 characters max |
| `Address` | Yes | `string` | Public blockchain address and the address format needs to meet the requirements of the chain |
| `Memo` | No | `string` | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the public blockchain address, also known as a comment or tag. Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `Force` | No | `bool` | When the whitelist is involved in a transaction approval policy, modifications will result in the new whitelist being directly applied to the approval policy. False by default, meaning that when involved in a transaction approval policy, it will not be modified |
| `Note` | No | `string` | Note |

---

## Delete a Whitelist Entry

```go
req := api.DeleteWhitelistRequest{
    WhitelistKey: whitelistKey,
}

var resp api.ResultResponse
if err := whitelistApi.DeleteWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to delete whitelist: %w", err))
}
```

---

## Create Whitelist from an Existing Transaction

```go
req := api.CreateFromTransactionWhitelistRequest{
    TxKey:              txKey,
    WhitelistName:      "Recipient from TX-001", // required, max 30 chars
    DestinationAddress: "0xRecipient...",        // required — the destination address in the tx record (case-sensitive)
    // Memo:            "...",                   // TON / XRP Ledger: must match the memo used in the transaction
    // HiddenOnUI:      false,
    // Note:            "optional note",
}

var resp api.CreateWhitelistResponse
if err := whitelistApi.CreateFromTransactionWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create whitelist from tx: %w", err))
}
whitelistKey := resp.WhitelistKey
```

### CreateFromTransactionWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `WhitelistName` | Yes | `string` | Whitelist unique name, 30 characters max |
| `TxKey` | Yes | `string` | Transaction key |
| `DestinationAddress` | Yes | `string` | The destination address in the transaction record; case-sensitive |
| `Memo` | No | `string` | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the destination address. For TON and XRP Ledger, if a destination address memo was set initially, a memo matching the one in the transaction record must be provided |
| `HiddenOnUI` | No | `bool` | If true, hidden from Web Console / App |
| `Note` | No | `string` | Note |

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `WhitelistKey` | `string` | Unique identifier |
| `WhitelistName` | `string` | Display name |
| `ChainType` | `string` | Blockchain type |
| `Address` | `string` | Whitelisted address |
| `Memo` | `string` | Memo of the address when creating a whitelist |
| `WhitelistStatus` | `string` | `AUDIT` / `APPROVED` / `REJECTED` |
| `Note` | `string` | Note |
| `CreateTime` | `int64` | Unix timestamp (ms) |
| `LastUpdateTime` | `int64` | Unix timestamp (ms) |

> The API also returns `hiddenOnUI`, and List accepts a `hiddenOnUI` filter, but Go SDK v1.0.15 `WhitelistResponse` / `ListWhitelistRequest` do not declare them yet.

### Whitelist Status Values

| Status | Description |
|--------|-------------|
| `AUDIT` | Pending approval (if policy requires human review) |
| `APPROVED` | Active -- can be used as transaction destination |
| `REJECTED` | Denied -- cannot be used |

---

## Using Whitelist as Transaction Destination

```go
req := api.CreateTransactionsRequest{
    DestinationAccountType: "WHITELISTING_ACCOUNT",
    DestinationAccountKey:  whitelistKey, // pass whitelistKey here
    CoinKey:                "ETHEREUM_ETH",
    TxAmount:               "0.05",
    // ... other fields
}
```

---

## Best Practices

- Validate address with `CoinApi.CheckCoinAddress()` before creating a whitelist entry.
- Use descriptive `WhitelistName` and set `ChainType` carefully -- wrong chainType causes address validation failure.
- In regulated environments, configure Safeheron Console policy to require human approval before whitelist additions take effect (`WhitelistStatus = "AUDIT"` until approved).
- Subscribe to Whitelist webhook events (`WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`) to keep your local state in sync.
