# Gas Station (GasService) API Reference

## Overview

Safeheron's Gas Station (Auto Gas Refill) service automatically tops up wallet gas when balances fall below a configured threshold, enabling uninterrupted operations for deposit wallets.

The `GasApiService` provides:
- Query current gas station balance
- Query auto gas refill records linked to transactions

---

## Imports

```java
import com.safeheron.client.api.GasApiService;
import com.safeheron.client.config.SafeheronConfig;
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
GasStatusResponse resp = ServiceExecutor.execute(gasApi.gasStatus());

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

### GasStatusResponse Structure

**`gasBalance` (List):**

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | String | Coin symbol (e.g. ETH, BNB) |
| `amount` | String | Current gas station balance |

**`configuration` (List):**

| Field | Type | Description |
|-------|------|-------------|
| `network` | String | Blockchain network name (Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon) |
| `enabled` | Boolean | Whether auto gas refill is enabled for this network |

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
| `totalAmount` | String |  Total fee amount across all records |
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

Each gas record includes status values such as:
- `SUCCESS` — Gas refill completed
- `FAILURE_GAS_CONSUMED` — Refill attempted but gas was used up
- `PENDING` — Refill in progress

---

## Webhook — Gas Balance Warning

When the gas station balance drops below the configured threshold, Safeheron sends a webhook notification:

```
Event Type: GAS_BALANCE_WARNING
```

See [WEBHOOK.md](WEBHOOK.md) for full webhook handling setup.

---

## Auto Sweep (归集) & Gas Station Integration

Auto Sweep (automatic fund collection) and Gas Station work together:
- Deposit wallets tagged with `DEPOSIT` label are eligible for auto sweep.
- Gas Station automatically provides gas when needed for these wallets.
- All sweep/gas/regular/speed-up transactions are reported via Webhook identically.

Auto Sweep prerequisites:
1. Target wallets must have `DEPOSIT` label set (see `UpdateAccountRequest.setAccountTag("DEPOSIT")`).
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
- Supported networks for auto gas refill: Ethereum, TRON, BNB Smart Chain, Arbitrum, Polygon.
- Gas API queries the team's Gas Station service, not individual wallet gas balances.
