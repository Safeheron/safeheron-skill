# Wallet Account API Reference

## Imports

```typescript
import { AccountApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instance

```typescript
const accountApi = new AccountApi(config);
```

---

## Create a Wallet Account

```typescript
const resp = await accountApi.createAccount({
  accountName: 'my-wallet-account',
  hiddenOnUI: false,  // true = API-only wallet, hidden in Console
});

const accountKey = resp.accountKey;  // save this -- permanent wallet identifier
```

**CreateAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `accountName` | Yes | string | Wallet display name |
| `customerRefId` | No | string | Merchant unique business ID (max 100 chars). Duplicate submissions with the same ID return the same wallet |
| `hiddenOnUI` | No | boolean | If true, wallet is hidden in Web Console / App |
| `autoFuel` | No | boolean | If true, Gas Station auto-tops up gas when a transaction is initiated. Default: false |
| `accountTag` | No | string | Tag applied at creation. Values: `DEPOSIT`, `NONE`. Required for Auto-Sweep |
| `coinKeyList` | No | Array\<string\> | Coin keys to add at creation (max 20) |

**CreateAccountResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `accountKey` | string | Unique wallet identifier (save permanently) |
| `pubKeys` | Array | Account public key list (signAlg, pubKey) |
| `coinAddressList` | Array | Coin address list per coin |

---

## List Wallet Accounts

```typescript
const result = await accountApi.listAccounts({
  pageSize: 10,
  pageNumber: 1,   // 1-indexed
});

const accounts = result.content;
const total = result.totalElements;

for (const acct of accounts) {
  console.log(acct.accountKey, acct.accountName);
}
```

**ListAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `pageSize` | No | number | Items per page (default 10, max 100) |
| `pageNumber` | No | number | Page number, 1-indexed |
| `hiddenOnUI` | No | boolean | Filter by visibility |
| `namePrefix` | No | string | Filter by account name prefix |
| `nameSuffix` | No | string | Filter by account name suffix |

**PageResult Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `pageNumber` | number | Current page number (1-indexed) |
| `pageSize` | number | Number of items per page |
| `totalElements` | number | Total number of records across all pages |
| `content` | `Array<T>` | Records on the current page |

**AccountResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `accountKey` | string | Unique wallet identifier |
| `accountName` | string | Wallet display name |
| `accountIndex` | number | Derivation path index |
| `accountTag` | string | Tag: `DEPOSIT`, `NONE`, etc. |
| `hiddenOnUI` | boolean | Hidden from UI |
| `accountType` | string | `VAULT_ACCOUNT` |

---

## Get a Single Wallet Account

```typescript
const resp = await accountApi.oneAccounts({
  accountKey: 'your-account-key',
});
```

> The method is `oneAccounts` (with 's'), not `oneAccount`.

---

## Batch Label Wallet Accounts

```typescript
await accountApi.batchUpdateAccountTag({
  accountKeyList: ['your-account-key'],
  accountTag: 'DEPOSIT',  // or "NONE" to remove label
});
```

**AccountTag Values:**

| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet -- eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

---

## Add Coin to a Wallet Account (V2 -- Recommended)

Add up to 20 coins in a single call. Adding a token auto-adds its mainnet coin.

```typescript
const res = await accountApi.createAccountCoinV2({
  accountKey: 'your-account-key',
  coinKeyList: [
    'ETHEREUM_ETH',
    'USDT(ERC20)_ETHEREUM_USDT',
    'BITCOIN_BTC',
  ],
});

for (const coin of res.coinAddressList) {
  console.log(`Added coin: ${coin.coinKey}, addresses: ${JSON.stringify(coin.addressList)}`);
}
```

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

Query balances and deposit addresses per coin for a specific account:

```typescript
const coins = await accountApi.listAccountCoin({
  accountKey: 'your-account-key',
});

for (const coin of coins) {
  console.log(`${coin.coinKey} balance: ${coin.balance} address: ${coin.addressList[0]?.address}`);
}
```

**AccountCoinResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | string | Coin identifier (e.g. `ETHEREUM_ETH`) |
| `coinFullName` | string | Full coin name (e.g. `Ethereum`) |
| `coinName` | string | Coin symbol (e.g. `ETH`) |
| `balance` | string | Account balance |
| `usdBalance` | string | Balance converted to USD |
| `addressList` | Array | Coin deposit address list |
| `feeCoinKey` | string | Fee coin identifier |

---

## API Method Summary

| Operation | Method | Request | Response |
|-----------|--------|---------|----------|
| Create account | `createAccount()` | `{ accountName, hiddenOnUI? }` | `{ accountKey, pubKeys, coinAddressList }` |
| List accounts | `listAccounts()` | `{ pageSize?, pageNumber? }` | `{ content, totalElements }` |
| Get one account | `oneAccounts()` | `{ accountKey }` | Account object |
| Batch update tag | `batchUpdateAccountTag()` | `{ accountKeyList, accountTag }` | void |
| Add coin V2 | `createAccountCoinV2()` | `{ accountKey, coinKeyList }` | `{ coinAddressList }` |
| List coins | `listAccountCoin()` | `{ accountKey }` | Array of coin objects |

---

## Best Practices

- `accountKey` is the permanent, immutable identifier for a wallet -- store it after creation.
- Web3 wallets use a separate set of APIs -- see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
