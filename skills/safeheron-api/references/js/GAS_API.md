# Gas Station (GasApi) API Reference

## Overview

Safeheron's Gas Service (Auto Gas Refill): when initiating a transaction, Gas Service automatically supplements the Gas fee for wallets where `autoFuel` is `true`, enabling uninterrupted operations for deposit wallets. When Gas Service is enabled and an alert threshold is set, Safeheron sends the `GAS_BALANCE_WARNING` webhook when the Gas balance is below the threshold (see below).

The `GasApi` provides:
- Query current gas station balance
- Query auto gas refill records linked to transactions

---

## Imports

```typescript
import { GasApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instance

```typescript
const gasApi = new GasApi(config);
```

---

## Retrieve Gas Station Balance

Query the current gas balance and auto-refill configuration:

```typescript
const resp = await gasApi.gasStatus({
  networkMode: 'MAINNET',  // MAINNET or TESTNET (optional in the API; required by the SDK v1.0.32 TS type)
});

// Gas balances per coin
for (const bal of resp.gasBalance) {
  console.log(`${bal.symbol}: ${bal.amount}`);
}

// Per-network auto-refill config
for (const cfg of resp.configuration) {
  console.log(`Network: ${cfg.network}, AutoRefill: ${cfg.enabled}`);
}
```

### GasStatusRequest Fields

| Field | Type | Description |
|-------|------|-------------|
| `networkMode` | string | Network mode: `MAINNET` (mainnet) / `TESTNET` (testnet). Optional in the API, but declared required in SDK v1.0.32's TS type |

### GasStatusResponse Structure

**`gasBalance` (Array):**

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | string | Coin symbol (e.g. ETH, BNB) |
| `amount` | string | Current gas station balance |

**`configuration` (Array):**

| Field | Type | Description |
|-------|------|-------------|
| `network` | string | Network. Currently supported networks are: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia |
| `enabled` | boolean | Whether auto gas refill is enabled for this network |

---

## Retrieve Auto Gas Records for a Transaction

Query gas refill records associated with a specific transaction (created via the V3 transaction API):

```typescript
const resp = await gasApi.gasTransactionsGetByTxKey({
  txKey: 'your-tx-key',  // txKey from Create Transaction V3
});

console.log('Total amount:', resp.totalAmount);
for (const detail of resp.detailList) {
  console.log(`Gas tx: ${detail.gasServiceTxKey}, amount: ${detail.amount}, status: ${detail.status}`);
}
```

### GasTransactionsGetByTxKeyResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | The queried transaction key |
| `symbol` | string | Transaction fee coin |
| `totalAmount` | string | Total fee amount. A single transaction may have multiple Gas records; the total fee paid is the sum of all records with SUCCESS and FAILURE_GAS_CONSUMED statuses. |
| `detailList` | Array | Gas refill records associated with this transaction |

**Detail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `gasServiceTxKey` | string | Energy rental transaction key |
| `symbol` | string | Transaction fee coin |
| `amount` | string | Amount |
| `balance` | string | Balance after paying the fee |
| `status` | string | SUCCESS / FAILURE_GAS_REFUNDED / FAILURE_GAS_CONSUMED |
| `resourceType` | string | TRON only: ENERGY or BANDWIDTH |
| `timestamp` | string | Gas deduction time (ms) |

---

## Webhook -- Gas Balance Warning

When Gas Service is enabled and an alert threshold is set, Safeheron sends a webhook notification when the Gas balance is below the threshold (e.g. first detected during automatic fueling, when the threshold is set or updated, or after a top-up that still leaves it below). It is sent at most once every 30 minutes:

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
1. Target wallets must have `DEPOSIT` label set (see `batchUpdateAccountTag`).
2. Auto Sweep policy must be configured in Safeheron Console.
3. Incoming transaction must satisfy the sweep policy threshold.
4. Gas station must have sufficient balance.

To cancel DEPOSIT label:

```typescript
await accountApi.batchUpdateAccountTag({
  accountKeyList: ['your-account-key'],
  accountTag: 'NONE',  // removes DEPOSIT label
});
```

---

## Best Practices

- **Gas Station balance is separate from wallet balances** -- top up via Safeheron Console.
- If auto-refill keeps failing, check: gas station balance, network congestion, and policy configuration.
- Supported networks for auto gas refill: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia.
- `autoFuel` controls whether Gas Service supplements the Gas fee when a transaction is initiated from the wallet — set it per wallet at creation, or in batch with `accountApi.batchUpdateAccountAutofuel` (see [WALLET_API.md](WALLET_API.md#batch-set-auto-fuel)).
- Gas API queries the team's Gas Station service, not individual wallet gas balances.
