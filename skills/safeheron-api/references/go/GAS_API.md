# Gas Station (GasApi) API Reference

## Overview

Safeheron's Gas Station (Auto Gas Refill) service automatically tops up wallet gas when balances fall below a configured threshold, enabling uninterrupted operations for deposit wallets.

The `GasApi` provides:
- Query current gas station balance
- Query auto gas refill records linked to transactions

---

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
gasApi := api.GasApi{Client: sc}
```

---

## Retrieve Gas Station Balance

```go
var resp api.GasStatusResponse
if err := gasApi.GasStatus(&resp); err != nil {
    panic(fmt.Errorf("failed to get gas status: %w", err))
}

for _, bal := range resp.GasBalance {
    fmt.Printf("%s: %s\n", bal.Symbol, bal.Amount)
}

for _, cfg := range resp.Configuration {
    fmt.Printf("Network: %s, AutoRefill: %v\n", cfg.Network, cfg.Enabled)
}
```

### GasStatusResponse Structure

**`GasBalance` (list):**

| Field | Type | Description |
|-------|------|-------------|
| `Symbol` | `string` | Coin symbol (e.g. ETH, BNB) |
| `Amount` | `string` | Current gas station balance |

**`Configuration` (list):**

| Field | Type | Description |
|-------|------|-------------|
| `Network` | `string` | Blockchain network name |
| `Enabled` | `bool` | Whether auto gas refill is enabled for this network |

---

## Retrieve Auto Gas Records for a Transaction

```go
req := api.GasTransactionsGetByTxKeyRequest{
    TxKey: "your-tx-key",
}

var resp api.GasTransactionsGetByTxKeyResponse
if err := gasApi.GasTransactionsGetByTxKey(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get gas records: %w", err))
}
```

> **Note:** This endpoint works with `TxKey` values from the V3 transaction API (`POST /v3/transactions/create`).

### GasTransactionsGetByTxKeyResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | `string` | The queried transaction key |
| `Symbol` | `string` | Transaction fee coin |
| `TotalAmount` | `string` | Total fee amount across all records |
| `DetailList` | `[]Detail` | Gas refill records |

**Detail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `GasServiceTxKey` | `string` | Energy rental transaction key |
| `Symbol` | `string` | Transaction fee coin |
| `Amount` | `string` | Amount |
| `Balance` | `string` | Balance after paying the fee |
| `Status` | `string` | SUCCESS / FAILURE_GAS_REFUNDED / FAILURE_GAS_CONSUMED |
| `ResourceType` | `string` | TRON only: ENERGY or BANDWIDTH |
| `Timestamp` | `string` | Gas deduction time (ms) |

---

## Webhook -- Gas Balance Warning

When the gas station balance drops below the configured threshold, Safeheron sends a webhook notification:

```
Event Type: GAS_BALANCE_WARNING
```

See [WEBHOOK.md](WEBHOOK.md) for full webhook handling setup.

---

## Auto Sweep & Gas Station Integration

Auto Sweep (automatic fund collection) and Gas Station work together:
- Deposit wallets tagged with `DEPOSIT` label are eligible for auto sweep.
- Gas Station automatically provides gas when needed for these wallets.
- All sweep/gas/regular/speed-up transactions are reported via Webhook identically.

Auto Sweep prerequisites:
1. Target wallets must have `DEPOSIT` label set at creation time.
2. Auto Sweep policy must be configured in Safeheron Console.
3. Incoming transaction must satisfy the sweep policy threshold.
4. Gas station must have sufficient balance.

To cancel DEPOSIT label:

```go
req := api.BatchUpdateAccountTagRequest{
    AccountKeyList: []string{"your-account-key"},
    AccountTag:     "NONE", // removes DEPOSIT label
}
var resp api.ResultResponse
if err := accountApi.BatchUpdateAccountTag(req, &resp); err != nil {
    panic(err)
}
```

---

## Best Practices

- **Gas Station balance is separate from wallet balances** -- top up via Safeheron Console.
- If auto-refill keeps failing, check: gas station balance, network congestion, and policy configuration.
- Supported networks for auto gas refill: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon.
- Gas API queries the team's Gas Station service, not individual wallet gas balances.
