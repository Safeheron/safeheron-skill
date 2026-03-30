# Wallet Account API Reference

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
accountApi := api.AccountApi{Client: sc}
```

---

## Create a Wallet Account

```go
req := api.CreateAccountRequest{
    AccountName: "my-wallet-account",
    // HiddenOnUI is *bool; to set it, use a pointer:
    //   hidden := true; req.HiddenOnUI = &hidden
}

var resp api.CreateAccountResponse
if err := accountApi.CreateAccount(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create wallet account: %w", err))
}
accountKey := resp.AccountKey // save this -- permanent wallet identifier
```

**CreateAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `AccountName` | Yes | `string` | Wallet display name |
| `CustomerRefId` | No | `string` | Merchant unique business ID (max 100 chars). Duplicate submissions with the same ID return the same wallet |
| `HiddenOnUI` | No | `*bool` | If true, wallet is hidden in Web Console / App |
| `AutoFuel` | No | `*bool` | If true, Gas Station auto-tops up gas when a transaction is initiated. Default: false |
| `AccountTag` | No | `string` | Tag applied at creation. Values: `DEPOSIT`, `NONE`. Required for Auto-Sweep |
| `CoinKeyList` | No | `[]string` | Coin keys to add at creation (max 20) |

**CreateAccountResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `AccountKey` | `string` | Unique wallet identifier (save permanently) |
| `PubKeys` | `[]struct{SignAlg, PubKey}` | Account public key list |
| `CoinAddressList` | `[]struct{...}` | Coin address list per coin |

---

## List Wallet Accounts

```go
req := api.ListAccountRequest{
    PageNumber: 1,
    PageSize:   10,
}

var res api.ListAccountResponse
if err := accountApi.ListAccounts(req, &res); err != nil {
    panic(fmt.Errorf("failed to list accounts: %w", err))
}

for _, acct := range res.Content {
    fmt.Printf("%s %s\n", acct.AccountKey, acct.AccountName)
}
```

**ListAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `PageSize` | No | `int` | Items per page (default 10) |
| `PageNumber` | No | `int` | Page number, 1-indexed |
| `HiddenOnUI` | No | `*bool` | Filter by hidden status |
| `AutoFuel` | No | `*bool` | Filter by auto gas refill setting |
| `Archived` | No | `*bool` | Filter by archived status |
| `NamePrefix` | No | `string` | Filter accounts whose name starts with this prefix |
| `NameSuffix` | No | `string` | Filter accounts whose name ends with this suffix |
| `CustomerRefId` | No | `string` | Filter by your custom reference ID |

**ListAccountResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `PageNumber` | `int32` | Current page number (1-indexed) |
| `PageSize` | `int32` | Number of items per page |
| `TotalElements` | `int64` | Total number of records across all pages |
| `Content` | `[]AccountResponse` | Records on the current page |

**AccountResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `AccountKey` | `string` | Unique wallet identifier |
| `AccountName` | `string` | Wallet display name |
| `CustomerRefId` | `string` | Your custom reference ID |
| `AccountIndex` | `int32` | Derivation path index |
| `AccountType` | `string` | `VAULT_ACCOUNT` |
| `AccountTag` | `string` | Tag: `DEPOSIT`, `NONE`, etc. |
| `HiddenOnUI` | `bool` | Hidden from UI |
| `AutoFuel` | `bool` | Auto gas refill enabled |
| `Archived` | `bool` | Whether the account is archived |
| `UsdBalance` | `string` | Total USD balance across all coins |
| `PubKeys` | `[]struct{SignAlg, PubKey string}` | MPC public key shards |

---

## Get a Single Wallet Account

```go
req := api.OneAccountRequest{
    AccountKey: accountKey,
}

var resp api.AccountResponse
if err := accountApi.OneAccounts(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get account: %w", err))
}
```

> The method is `OneAccounts` (with 's'), matching the API endpoint convention.

---

## Batch Label Wallet Accounts

```go
req := api.BatchUpdateAccountTagRequest{
    AccountKeyList: []string{"your-account-key"},
    AccountTag:     "DEPOSIT", // or "NONE" to remove
}

var resp api.ResultResponse
if err := accountApi.BatchUpdateAccountTag(req, &resp); err != nil {
    panic(fmt.Errorf("failed to update account tag: %w", err))
}
```

**AccountTag Values:**

| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet -- eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

---

## Add Coin to a Wallet Account

```go
req := api.AddCoinRequest{
    AccountKey: accountKey,
    CoinKey:    "ETH(SEPOLIA)_ETHEREUM_SEPOLIA",
}

var resp api.AddCoinResponse
if err := accountApi.AddCoin(req, &resp); err != nil {
    panic(fmt.Errorf("failed to add coin: %w", err))
}

fmt.Printf("Address: %s\n", resp[0].Address)
```

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

```go
req := api.ListAccountCoinRequest{
    AccountKey: accountKey,
}

var coins []api.AccountCoinResponse
if err := accountApi.ListAccountCoin(req, &coins); err != nil {
    panic(fmt.Errorf("failed to list account coins: %w", err))
}

for _, coin := range coins {
    fmt.Printf("%-30s balance: %-20s\n", coin.CoinKey, coin.Balance)
}
```

---

## Best Practices

- `PageSize` and `PageNumber` are `int` type in Go (not `Long` like Java).
- `accountKey` is the permanent, immutable identifier for a wallet -- store it after creation.
- Web3 wallets use a separate set of APIs -- see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
