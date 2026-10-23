# Gas Station (GasService) API Reference

## Overview

With the Gas Service enabled, Safeheron can automatically supplement the necessary Gas fees for transactions, enabling uninterrupted operations for deposit wallets. A wallet's `autoFuel` setting controls this: if set to true, the Gas Service will automatically supplement the Gas fee for the wallet when a transaction is initiated (see [WALLET_API.md](WALLET_API.md)). The alert threshold set in the Gas Service is used for the `GAS_BALANCE_WARNING` webhook (see below).

The `GasApiService` provides:
- Query current gas station balance
- Query auto gas refill records linked to transactions

---

## Imports

```java
import com.safeheron.client.api.GasApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.GasStatusRequest;
import com.safeheron.client.request.GasTransactionsGetByTxKeyRequest;
import com.safeheron.client.response.GasStatusResponse;
import com.safeheron.client.response.GasTransactionsGetByTxKeyResponse;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
GasApiService gasApi = ServiceCreator.create(GasApiService.class, safeheronConfig);
```

---

## Retrieve Gas Station Balance

Query the current gas balance and auto-refill configuration:

```java
GasStatusRequest req = new GasStatusRequest();
// req.setNetworkMode("MAINNET"); // optional: MAINNET or TESTNET

GasStatusResponse resp = ServiceExecutor.execute(gasApi.gasStatus(req));

// Gas balances per coin
for (GasStatusResponse.GasBalance bal : resp.getGasBalance()) {
    System.out.println(bal.getSymbol() + ": " + bal.getAmount());
}

// Per-network auto-refill config
for (GasStatusResponse.Configuration cfg : resp.getConfiguration()) {
    System.out.println("Network: " + cfg.getNetwork()
                       + " AutoRefill: " + cfg.getEnabled());
}
```

### GasStatusRequest Fields

| Field | Type | Description |
|-------|------|-------------|
| `networkMode` | String | Optional. Network mode: `MAINNET` (mainnet) / `TESTNET` (testnet) |

### GasStatusResponse Structure

**`gasBalance` (List):**

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | String | Coin symbol (e.g. ETH, BNB) |
| `amount` | String | Current gas station balance |

**`configuration` (List):**

| Field | Type | Description |
|-------|------|-------------|
| `network` | String | Network. Currently supported networks are: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia |
| `enabled` | Boolean | Whether automatic refueling is enabled |

---

## Retrieve Auto Gas Records for a Transaction

Query gas refill records associated with a specific transaction (created via the V3 transaction API):

```java
GasTransactionsGetByTxKeyRequest req = new GasTransactionsGetByTxKeyRequest();
req.setTxKey("your-tx-key");  // txKey from Create Transaction V3 (POST /v3/transactions/create)

GasTransactionsGetByTxKeyResponse resp = ServiceExecutor.execute(
        gasApi.gasTransactionsGetByTxKey(req));
```

> **Note:** This endpoint works with `txKey` values from the V3 transaction API (`POST /v3/transactions/create`). The `CreateTransactionV3Response` contains the `txKey`.

### GasTransactionsGetByTxKeyResponse Fields

| Field | Type |  Description |
|-------|------|-------------|
| `txKey` | String |  The queried transaction key |
| `symbol` | String |  Transaction fee coin |
| `totalAmount` | String |  Total fee amount. A single transaction may have multiple Gas records; the total fee paid is the sum of all records with SUCCESS and FAILURE_GAS_CONSUMED statuses. |
| `detailList` | `List<Detail>` |  Gas refill records associated with this transaction |

**GasTransactionsGetByTxKeyResponse.Detail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `gasServiceTxKey` | String | Energy rental transaction key |
| `symbol` | String | Transaction fee coin |
| `amount` | String | Amount |
| `balance` | String | Balance after paying the fee |
| `status` | String | SUCCESS / FAILURE_GAS_REFUNDED / FAILURE_GAS_CONSUMED |
| `resourceType` | String | TRON only: ENERGY or BANDWIDTH |
| `timestamp` | String | Gas deduction time (ms) |

Status values:
- `SUCCESS` — Gas refill completed
- `FAILURE_GAS_REFUNDED` — Gas failed, refunded
- `FAILURE_GAS_CONSUMED` — Gas failed, but fees were incurred

---

## Webhook — Gas Balance Warning

When Gas Service is enabled and an alert threshold is set, Safeheron sends a webhook notification when the Gas balance is detected below the threshold (during automatic fueling, when the threshold is first set or updated, or after a top-up that still leaves it below). It is triggered at most once every 30 minutes:

```
Event Type: GAS_BALANCE_WARNING
```

See [WEBHOOK.md](WEBHOOK.md) for full webhook handling setup.

---

## Auto Sweep (归集) & Gas Station Integration

Auto Sweep (automatic fund collection) and Gas Station work together:
- Deposit wallets tagged with `DEPOSIT` label are eligible for auto sweep.
- The Gas Service can supply the Gas these transactions need — see the `autoFuel` description in [WALLET_API.md](WALLET_API.md).
- All sweep/gas/regular/speed-up transactions are reported via Webhook identically.

Auto Sweep prerequisites:
1. Target wallets must have `DEPOSIT` label set (`BatchUpdateAccountTagRequest` + `accountApi.batchUpdateAccountTag`, see below).
2. Auto Sweep policy must be configured in Safeheron Console.
3. Incoming transaction must satisfy the sweep policy threshold.
4. Gas station must have sufficient balance.

To cancel DEPOSIT label:
```java
BatchUpdateAccountTagRequest req = new BatchUpdateAccountTagRequest();
req.setAccountKeyList(Arrays.asList("your-account-key"));
req.setAccountTag("NONE");  // removes DEPOSIT label
ServiceExecutor.execute(accountApi.batchUpdateAccountTag(req));
```

---

## Best Practices

- **Gas Station balance is separate from wallet balances** — top up via Safeheron Console.
- If auto-refill keeps failing, check: gas station balance, network congestion, and policy configuration.
- Supported networks for Gas Service: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon, Solana, Optimism, Base, Avalanche C-Chain, TRON(Shasta), Sepolia.
- Gas API queries the team's Gas Station service, not individual wallet gas balances.
