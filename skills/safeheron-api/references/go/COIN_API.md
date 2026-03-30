# Coin Management API Reference

## Overview

Coin management covers two distinct API types:
- `CoinApi` -- global coin/network info, address validation, balance snapshots
- `AccountApi` -- per-account coin operations (add coin, list coins on an account)

---

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instances

```go
coinApi    := api.CoinApi{Client: sc}
accountApi := api.AccountApi{Client: sc}
```

---

## CoinApi Operations

### List All Supported Coins

```go
var coins api.CoinResponse
if err := coinApi.ListCoin(&coins); err != nil {
    panic(fmt.Errorf("failed to list coins: %w", err))
}

for _, coin := range coins {
    fmt.Printf("%s / %s\n", coin.CoinKey, coin.CoinName)
}
```

**CoinResponse Item Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `CoinKey` | `string` | Unique coin identifier (e.g. `ETHEREUM_ETH`, `BITCOIN_BTC`) |
| `CoinFullName` | `string` | Full name (e.g. "Ethereum") |
| `CoinName` | `string` | Symbol (e.g. "ETH") |
| `Symbol` | `string` | Coin unit display name |
| `CoinDecimal` | `int32` | Decimal places |
| `ShowCoinDecimal` | `int32` | Displayed decimal places on Console |
| `CoinType` | `string` | Coin type classification |
| `FeeCoinKey` | `string` | Coin used for gas fees |
| `FeeUnit` | `string` | Fee unit name (Gwei, satoshis) |
| `FeeDecimal` | `int32` | Fee decimal |
| `GasLimit` | `int32` | Default gas limit |
| `BlockChain` | `string` | Blockchain name |
| `BlockchainType` | `string` | EVM, UTXO, etc. |
| `Network` | `string` | Mainnet / Testnet |
| `TokenIdentifier` | `string` | Contract address or NATIVE |
| `MinTransferAmount` | `string` | Minimum transfer amount |
| `IsMultipleAddress` | `string` | YES/NO -- supports multiple address groups |
| `IsUtxo` | `string` | YES/NO -- UTXO-based chain |
| `IsMemo` | `string` | YES/NO -- Requires memo/tag |
| `TxRefUrl` | `string` | Block explorer TX URL template |
| `AddressRefUrl` | `string` | Block explorer address URL |
| `LogoUrl` | `string` | Coin logo URL |

---

### Validate a Coin Address

Use before adding to whitelist or sending funds:

```go
req := api.CheckCoinAddressRequest{
    CoinKey: "ETHEREUM_ETH",
    Address: "0xAbCd...",
}

var resp api.CheckCoinAddressResponse
if err := coinApi.CheckCoinAddress(req, &resp); err != nil {
    panic(fmt.Errorf("failed to check address: %w", err))
}

if !resp.AddressValid {
    panic("invalid address format")
}
if !resp.AmlValid {
    panic("address failed AML check")
}
```

---

### Coin Balance Snapshot (Team-wide)

Query aggregate balance for specific coins across all accounts in the team:

```go
req := api.CoinBalanceSnapshotRequest{
    Gmt8Date: "2026-01-01",
}

var result api.CoinBalanceSnapshotResponse
if err := coinApi.CoinBalanceSnapshot(req, &result); err != nil {
    panic(fmt.Errorf("failed to get balance snapshot: %w", err))
}

for _, item := range result {
    fmt.Printf("%s: %s\n", item.CoinKey, item.CoinBalance)
}
```

---

### Get Current Block Height

```go
req := api.CoinBlockHeightRequest{
    CoinKey: "ETHEREUM_ETH",
}

var result api.CoinBlockHeightResponse
if err := coinApi.CoinBlockHeight(req, &result); err != nil {
    panic(fmt.Errorf("failed to get block height: %w", err))
}

for _, item := range result {
    fmt.Printf("%s local block height: %d\n", item.CoinKey, item.LocalBlockHeight)
}
```

---

## AccountApi -- Per-Account Coin Operations

### Add Coin to a Wallet Account

```go
req := api.AddCoinRequest{
    AccountKey: "your-account-key",
    CoinKey:    "ETH(SEPOLIA)_ETHEREUM_SEPOLIA",
}

var resp api.AddCoinResponse
if err := accountApi.AddCoin(req, &resp); err != nil {
    panic(fmt.Errorf("failed to add coin: %w", err))
}

fmt.Printf("Address: %s\n", resp[0].Address)
```

**Notes:**
- If `CoinKey` was already added, the response returns the same data as before (idempotent).
- If a previously added coin was manually disabled on the UI, re-adding via API will NOT re-enable it.

---

### List Coins for a Wallet Account

```go
req := api.ListAccountCoinRequest{
    AccountKey: "your-account-key",
}

var coins api.AccountCoinResponse
if err := accountApi.ListAccountCoin(req, &coins); err != nil {
    panic(fmt.Errorf("failed to list coins: %w", err))
}

for _, coin := range coins {
    fmt.Printf("%s balance: %s\n", coin.CoinKey, coin.Balance)
}
```

---

## Automatic Coin Addition Rules

1. If a **mainnet coin** is enabled and a **token** of that chain is received as deposit, the token coin type is **automatically added**.
2. If you add a **token** coinKey via API, the corresponding **mainnet coin** is also automatically added.
3. If you add a **mainnet** coinKey, only that mainnet coin is added (no automatic token additions).

---

## Supported Test Networks

| Network | coinKey Example | Notes |
|---------|----------------|-------|
| Ethereum Sepolia | `ETH(SEPOLIA)_ETHEREUM_SEPOLIA` | Test faucet tokens available |
| Bitcoin Testnet | `BITCOIN_BTC_TESTNET` | Faucet: https://coinfaucet.eu/en/btc-testnet4/ |
| TRON Shasta | TRON testnet coinKey | TLK token available on Shasta |

Test token contracts (Ethereum Sepolia):
- GSK: `0xF191b0720cb49DdAb6ECd72a65955a35b31fc944`
- USDC: `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238` -- Faucet: https://faucet.circle.com/

---

## Best Practices

- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`, `TRON_TRX`.
- Always validate addresses with `CheckCoinAddress` before sending funds or adding to whitelist.
- Team-level balance: use `CoinBalanceSnapshot`. Per-account balance: use `ListAccountCoin`.
