# Wallet Account API Reference

## Imports

```python
from safeheron_api_sdk_python.api.account_api import (
    AccountApi,
    CreateAccountRequest,
    ListAccountRequest,
    OneAccountRequest,
    BatchUpdateAccountTagRequest,
    CreateAccountCoinRequest,
    CreateAccountCoinRequestV2,
    ListAccountCoinRequest,
)
```

## Create API Instance

```python
account_api = AccountApi(config)
```

---

## Create a Wallet Account

```python
param = CreateAccountRequest()
param.accountName = "my-wallet-account"
param.hiddenOnUI = False  # True = API-only wallet, hidden in Console

resp = account_api.create_account(param)
account_key = resp['accountKey']  # save this -- permanent wallet identifier
pub_keys = resp.get('pubKeys', [])
coin_address_list = resp.get('coinAddressList', [])
```

**CreateAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `accountName` | Yes | str | Wallet display name |
| `customerRefId` | No | str | Merchant unique business ID (max 100 chars). Duplicate submissions with the same ID return the same wallet |
| `hiddenOnUI` | No | bool | If True, wallet is hidden in Web Console / App |
| `autoFuel` | No | bool | If True, Gas Station auto-tops up gas when a transaction is initiated. Default: False |
| `accountTag` | No | str | Tag applied at creation. Values: `DEPOSIT`, `NONE`. Required for Auto-Sweep |
| `coinKeyList` | No | list[str] | Coin keys to add at creation (max 20) |

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `accountKey` | str | Unique wallet identifier (save permanently) |
| `pubKeys` | list | Account public key list (signAlg, pubKey) |
| `coinAddressList` | list | Coin address list per coin |

---

## List Wallet Accounts

```python
param = ListAccountRequest()
param.pageSize = 10
param.pageNumber = 1   # 1-indexed

result = account_api.list_accounts(param)
accounts = result['content']
total = result['totalElements']

for acct in accounts:
    print(acct['accountKey'], acct['accountName'])
```

**ListAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `pageSize` | No | int | Items per page (default 10) |
| `pageNumber` | No | int | Page number, 1-indexed |

**Account Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `accountKey` | str | Unique wallet identifier |
| `accountName` | str | Wallet display name |
| `accountIndex` | int | Derivation path index |
| `accountTag` | str | Tag: `DEPOSIT`, `NONE`, etc. |
| `hiddenOnUI` | bool | Hidden from UI |
| `accountType` | str | `VAULT_ACCOUNT` |

---

## Get a Single Wallet Account

```python
param = OneAccountRequest()
param.accountKey = account_key

resp = account_api.one_accounts(param)
```

> The method is `one_accounts` (with 's'), not `one_account`.

---

## Batch Label Wallet Accounts

```python
param = BatchUpdateAccountTagRequest()
param.accountKeyList = ["your-account-key"]
param.accountTag = "NONE"  # "DEPOSIT" or "NONE"

account_api.batch_update_account_tag(param)
```

**AccountTag Values:**

| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet -- eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

> Cancelling `DEPOSIT` label: set `accountTag = "NONE"`. Can be re-applied later.

---

## Add Coin to a Wallet Account (V2 -- Recommended)

Add up to 20 coins in a single call. Adding a token auto-adds its mainnet coin.

```python
param = CreateAccountCoinRequestV2()
param.accountKey = account_key
param.coinKeyList = [
    "ETHEREUM_ETH",
    "USDT(ERC20)_ETHEREUM_USDT",
    "BITCOIN_BTC",
]

resp = account_api.create_account_coin_v2(param)
for coin in resp.get('coinAddressList', []):
    print(f"  Added coin: {coin['coinKey']} addressList: {coin.get('addressList', [])}")
```

> **V1 (single coin):** `CreateAccountCoinRequest` with `coinKey` set to a single coin string -- still works.

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

Query balances and deposit addresses per coin for a specific account:

```python
param = ListAccountCoinRequest()
param.accountKey = account_key

coins = account_api.list_account_coin(param)
for coin in coins:
    addr = coin['addressList'][0]['address'] if coin.get('addressList') else 'N/A'
    print(f"{coin['coinKey']:30s} balance: {coin['balance']:20s} address: {addr}")
```

**AccountCoinResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | str | Coin identifier (e.g. `ETHEREUM_ETH`) |
| `coinFullName` | str | Full coin name (e.g. `Ethereum`) |
| `coinName` | str | Coin symbol (e.g. `ETH`) |
| `balance` | str | Account balance |
| `usdBalance` | str | Balance converted to USD |
| `addressList` | list | Coin deposit address list |
| `feeCoinKey` | str | Fee coin identifier |
| `txRefUrl` | str | Transaction explorer URL template |

---

## Request / Response Summary

| Operation | Request Class | Method |
|-----------|--------------|--------|
| Create account | `CreateAccountRequest` | `create_account(param)` |
| List accounts | `ListAccountRequest` | `list_accounts(param)` |
| Get one account | `OneAccountRequest` | `one_accounts(param)` |
| Batch update tag | `BatchUpdateAccountTagRequest` | `batch_update_account_tag(param)` |
| Add coin V2 | `CreateAccountCoinRequestV2` | `create_account_coin_v2(param)` |
| Add coin V1 | `CreateAccountCoinRequest` | `create_account_coin(param)` |
| List coins | `ListAccountCoinRequest` | `list_account_coin(param)` |

---

## Best Practices

- `accountKey` is the permanent, immutable identifier for a wallet -- store it after creation.
- Web3 wallets use a separate set of APIs -- see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
