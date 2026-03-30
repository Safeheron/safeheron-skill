# Coin Management API Reference

## Overview

Coin management covers two distinct API classes:
- `CoinApi` -- global coin/network info, address validation, balance snapshots
- `AccountApi` -- per-account coin operations (add coin, list coins on an account)

---

## Imports

```typescript
import { CoinApi, AccountApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instances

```typescript
const coinApi = new CoinApi(config);
const accountApi = new AccountApi(config);
```

---

## CoinApi Operations

### List All Supported Coins

```typescript
const coins = await coinApi.coinList();
for (const coin of coins) {
  console.log(`${coin.coinKey} / ${coin.coinFullName}`);
}
```

**CoinResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | string | Unique coin identifier (e.g. `ETHEREUM_ETH`, `BITCOIN_BTC`) |
| `coinFullName` | string | Full name (e.g. "Ethereum") |
| `coinName` | string | Symbol (e.g. "ETH") |
| `symbol` | string | Coin unit display name |
| `coinDecimal` | string | Decimal places |
| `showCoinDecimal` | string | Displayed decimal places on Console |
| `coinType` | string | Coin type classification |
| `feeCoinKey` | string | Coin used for gas fees |
| `feeUnit` | string | Fee unit name (Gwei, satoshis) |
| `feeDecimal` | string | Fee decimal |
| `gasLimit` | string | Default gas limit |
| `blockChain` | string | Blockchain name |
| `blockchainType` | string | EVM, UTXO, etc. |
| `network` | string | Mainnet / Testnet |
| `tokenIdentifier` | string | Contract address or NATIVE |
| `minTransferAmount` | string | Minimum transfer amount |
| `isMultipleAddress` | string | YES/NO -- supports multiple address groups |
| `isUtxo` | string | YES/NO -- UTXO-based chain |
| `isMemo` | string | YES/NO -- Requires memo/tag |
| `txRefUrl` | string | Block explorer TX URL template |
| `addressRefUrl` | string | Block explorer address URL |
| `logoUrl` | string | Coin logo URL |

---

### Validate a Coin Address

Use before adding to whitelist or sending funds:

```typescript
const resp = await coinApi.checkCoinAddress({
  coinKey: 'ETHEREUM_ETH',
  address: '0xAbCd...',
});

if (!resp.addressValid) {
  throw new Error(`Invalid address format: ${address}`);
}
if (!resp.amlValid) {
  throw new Error(`Address failed AML check: ${address}`);
}
```

**CheckCoinAddressRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `coinKey` | Yes | string | Coin identifier |
| `address` | Yes | string | Address to validate |

**CheckCoinAddressResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `addressValid` | boolean | Valid address format |
| `contract` | boolean | Contract address |
| `amlValid` | boolean | Subject to risk control limitations |

---

### Coin Balance Snapshot (Team-wide)

Query aggregate balance for specific coins across all accounts in the team:

```typescript
const result = await coinApi.coinBalanceSnapshot({
  gmt8Date: '2026-01-01',
});

for (const item of result) {
  console.log(`${item.coinKey}: ${item.coinBalance}`);
}
```

---

### Get Current Block Height

```typescript
const result = await coinApi.coinBlockHeight({
  coinKey: 'ETHEREUM_ETH',
});

for (const item of result) {
  console.log(`${item.coinKey} local block height: ${item.localBlockHeight}`);
}
```

---

### List Coins Under Maintenance

```typescript
const maintained = await coinApi.listCoinMaintain();
```

---

## AccountApi -- Per-Account Coin Operations

### Add Coin(s) to a Wallet Account (V2 -- Recommended)

Supports adding up to 20 coins in a single call. Adding a token automatically adds its parent mainnet coin.

```typescript
const res = await accountApi.createAccountCoinV2({
  accountKey: 'your-account-key',
  coinKeyList: ['USDT(ERC20)_ETHEREUM_USDT', 'ETH(SEPOLIA)_ETHEREUM_SEPOLIA'],
});

for (const coin of res.coinAddressList) {
  console.log(`Added coin: ${coin.coinKey}, addresses: ${JSON.stringify(coin.addressList)}`);
}
```

**Notes:**
- If `coinKey` was already added, the response returns the same data as before (idempotent).
- If a previously added coin was manually disabled on the UI, re-adding via API will NOT re-enable it.

---

### List Coins for a Wallet Account

```typescript
const coins = await accountApi.listAccountCoin({
  accountKey: 'your-account-key',
});

for (const coin of coins) {
  console.log(`${coin.coinKey} balance: ${coin.balance} address: ${coin.addressList[0]?.address}`);
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
- Always validate addresses with `checkCoinAddress` before sending funds or adding to whitelist.
- Team-level balance: use `coinBalanceSnapshot`. Per-account balance: use `listAccountCoin`.
