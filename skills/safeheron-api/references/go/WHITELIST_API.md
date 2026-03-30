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
    WhitelistName: "Alice's ETH address", // max 20 chars
    ChainType:     "EVM",                 // see chainType values below
    Address:       "0xAliceEthAddress...",
    // Memo:       "optional memo",       // for TON networks (max 20 chars)
    // HiddenOnUI: false,                 // true = API-only, hidden in Console
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
| `WhitelistName` | Yes | `string` | Display name (max 20 chars) |
| `ChainType` | Yes | `string` | Blockchain type (see table below) |
| `Address` | Yes | `string` | Blockchain address to whitelist |
| `Memo` | No | `string` | Memo/tag for TON networks (max 20 chars) |
| `HiddenOnUI` | No | `bool` | If true, hidden from Web Console / App |

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

```go
req := api.OneWhitelistRequest{
    WhitelistKey: whitelistKey,
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

```go
req := api.ListWhitelistRequest{
    Limit: 20,
}

var list []api.WhitelistResponse
if err := whitelistApi.ListWhitelist(req, &list); err != nil {
    panic(fmt.Errorf("failed to list whitelist: %w", err))
}
for _, entry := range list {
    fmt.Printf("%s %s %s\n", entry.WhitelistKey, entry.WhitelistName, entry.WhitelistStatus)
}
```

---

## Edit a Whitelist Entry

```go
req := api.EditWhitelistRequest{
    WhitelistKey:  whitelistKey,
    WhitelistName: "Updated name",
}

var resp api.ResultResponse
if err := whitelistApi.EditWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to edit whitelist: %w", err))
}
```

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
    TxKey:         txKey,
    WhitelistName: "Recipient from TX-001",
}

var resp api.CreateWhitelistResponse
if err := whitelistApi.CreateFromTransactionWhitelist(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create whitelist from tx: %w", err))
}
whitelistKey := resp.WhitelistKey
```

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `WhitelistKey` | `string` | Unique identifier |
| `WhitelistName` | `string` | Display name |
| `ChainType` | `string` | Blockchain type |
| `Address` | `string` | Whitelisted address |
| `Memo` | `string` | Memo (TON networks) |
| `WhitelistStatus` | `string` | `AUDIT` / `APPROVED` / `REJECTED` |
| `CreateTime` | `int64` | Unix timestamp (ms) |
| `LastUpdateTime` | `int64` | Unix timestamp (ms) |

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
