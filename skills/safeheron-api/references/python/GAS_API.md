# Gas Station (GasApi) API Reference

## Overview

When initiating a transaction, Gas Service automatically supplements the Gas fee for wallets where `autoFuel` is `true` (set per wallet — see [WALLET_API.md](WALLET_API.md)). When Gas Service is enabled and an alert threshold is set, the system triggers the `GAS_BALANCE_WARNING` webhook when the Gas balance is below the threshold (see below).

Supported networks: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia.

The `GasApi` provides:
- Query current gas station balance
- Query auto gas refill records linked to transactions

---

## Imports

```python
from safeheron_api_sdk_python.api.gas_api import GasApi, GasStatusRequest, GasTransactionsGetByTxKeyRequest
```

## Create API Instance

```python
gas_api = GasApi(config)
```

---

## Retrieve Gas Station Balance

Query the current gas balance and auto-refill configuration:

```python
param = GasStatusRequest()
# param.networkMode = 'MAINNET'  # optional: 'MAINNET' or 'TESTNET'

resp = gas_api.gas_status(param)

# Gas balances per coin
for bal in resp.get('gasBalance', []):
    print(f"{bal['symbol']}: {bal['amount']}")

# Per-network auto-refill config
for cfg in resp.get('configuration', []):
    print(f"Network: {cfg['network']} AutoRefill: {cfg['enabled']}")
```

### GasStatusRequest Fields

| Field | Type | Description |
|-------|------|-------------|
| `networkMode` | str | Optional. Network mode: `MAINNET` (mainnet) / `TESTNET` (testnet) |

### GasStatusResponse Structure

**`gasBalance` (list):**

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | str | Coin symbol (e.g. ETH, BNB) |
| `amount` | str | Current gas station balance |

**`configuration` (list):**

| Field | Type | Description |
|-------|------|-------------|
| `network` | str | Network. Currently supported: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia |
| `enabled` | bool | Whether automatic refueling is enabled |

---

## Retrieve Auto Gas Records for a Transaction

Query gas refill records associated with a specific transaction:

```python
param = GasTransactionsGetByTxKeyRequest()
param.txKey = 'your-tx-key'
resp = gas_api.gas_transactions_get_by_tx_key(param)
```

> **Note:** This endpoint works with `txKey` values from the V3 transaction API. The deprecated `gas_transactions_ge_b_tx_key` method (typo) has been replaced by `gas_transactions_get_by_tx_key`.

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | The queried transaction key |
| `symbol` | str | Transaction fee coin |
| `totalAmount` | str | Total fee amount. A single transaction may have multiple Gas records; the total fee paid is the sum of all records with SUCCESS and FAILURE_GAS_CONSUMED statuses. |
| `detailList` | list | Gas refill records associated with this transaction |

**Detail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `gasServiceTxKey` | str | Energy rental transaction key |
| `symbol` | str | Transaction fee coin |
| `amount` | str | Amount |
| `balance` | str | Balance after paying the fee |
| `status` | str | SUCCESS / FAILURE_GAS_REFUNDED / FAILURE_GAS_CONSUMED |
| `resourceType` | str | TRON only: ENERGY or BANDWIDTH |
| `timestamp` | str | Gas deduction time (ms) |

---

## Webhook -- Gas Balance Warning

When Gas Service is enabled and an alert threshold is set, Safeheron sends this webhook when the Gas balance is below the threshold — e.g. first detected below it during automatic fueling, when the threshold is first set or updated while the balance is already below it, or after a top-up that still leaves it below. It is sent at most once every 30 minutes:

```
Event Type: GAS_BALANCE_WARNING
```

See [WEBHOOK.md](WEBHOOK.md) for full webhook handling setup.

---

## Auto Sweep & Gas Station Integration

Auto Sweep (automatic fund collection) and Gas Station work together:
- Deposit wallets tagged with `DEPOSIT` label are eligible for auto sweep.
- Gas Service supplements the Gas fee for wallets whose `autoFuel` is `true` (`batch_update_account_autofuel`, see [WALLET_API.md](WALLET_API.md)).
- All sweep/gas/regular/speed-up transactions are reported via Webhook identically.

Auto Sweep prerequisites:
1. Target wallets must have `DEPOSIT` label set.
2. Auto Sweep policy must be configured in Safeheron Console.
3. Incoming transaction must satisfy the sweep policy threshold.
4. Gas station must have sufficient balance.

To cancel DEPOSIT label:

```python
from safeheron_api_sdk_python.api.account_api import AccountApi, BatchUpdateAccountTagRequest

param = BatchUpdateAccountTagRequest()
param.accountKeyList = ["your-account-key"]
param.accountTag = "NONE"  # removes DEPOSIT label
account_api.batch_update_account_tag(param)
```

---

## Best Practices

- **Gas Station balance is separate from wallet balances** -- top up via Safeheron Console.
- If auto-refill keeps failing, check: gas station balance, network congestion, and policy configuration.
- Supported networks: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia.
- Gas API queries the team's Gas Station service, not individual wallet gas balances.
