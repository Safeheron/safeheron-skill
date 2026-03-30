# Coin Management API Reference

## Overview

Coin management covers two distinct API classes:
- `CoinApi` -- global coin/network info, address validation, balance snapshots
- `AccountApi` -- per-account coin operations (add coin, list coins on an account)

---

## Imports

```python
from safeheron_api_sdk_python.api.coin_api import (
    CoinApi,
    CheckCoinAddressRequest,
    CoinBalanceSnapshotRequest,
    CoinBlockHeightRequest,
)
from safeheron_api_sdk_python.api.account_api import (
    AccountApi,
    CreateAccountCoinRequestV2,
    CreateAccountCoinRequest,
    ListAccountCoinRequest,
)
```

## Create API Instances

```python
coin_api    = CoinApi(config)
account_api = AccountApi(config)
```

---

## CoinApi Operations

### List All Supported Coins

```python
coins = coin_api.list_coin()
for coin in coins:
    print(f"{coin['coinKey']} / {coin['coinFullName']}")
```

**CoinResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | str | Unique coin identifier (e.g. `ETHEREUM_ETH`, `BITCOIN_BTC`) |
| `coinFullName` | str | Full name (e.g. "Ethereum") |
| `coinName` | str | Symbol (e.g. "ETH") |
| `symbol` | str | Coin unit display name |
| `coinDecimal` | str | Decimal places |
| `showCoinDecimal` | str | Displayed decimal places on Console |
| `coinType` | str | Coin type classification |
| `feeCoinKey` | str | Coin used for gas fees |
| `feeUnit` | str | Fee unit name (Gwei, satoshis) |
| `feeDecimal` | str | Fee decimal |
| `gasLimit` | str | Default gas limit |
| `blockChain` | str | Blockchain name |
| `blockchainType` | str | EVM, UTXO, etc. |
| `network` | str | Mainnet / Testnet |
| `tokenIdentifier` | str | Contract address or NATIVE |
| `minTransferAmount` | str | Minimum transfer amount |
| `isMultipleAddress` | str | YES/NO -- supports multiple address groups |
| `isUtxo` | str | YES/NO -- UTXO-based chain |
| `isMemo` | str | YES/NO -- Requires memo/tag |
| `txRefUrl` | str | Block explorer TX URL template |
| `addressRefUrl` | str | Block explorer address URL |
| `logoUrl` | str | Coin logo URL |

---

### Validate a Coin Address

Use before adding to whitelist or sending funds:

```python
param = CheckCoinAddressRequest()
param.coinKey = "ETHEREUM_ETH"
param.address = "0xAbCd..."
param.checkContract = True
param.checkAml = True
param.checkAddressValid = True

resp = coin_api.check_coin_address(param)

if not resp.get('addressValid'):
    raise ValueError(f"Invalid address format: {param.address}")
if not resp.get('amlValid'):
    raise ValueError(f"Address failed AML check: {param.address}")
```

**CheckCoinAddressRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `coinKey` | Yes | str | Coin identifier |
| `address` | Yes | str | Address to validate |
| `checkContract` | No | bool | Check if address is a contract |
| `checkAml` | No | bool | Check AML risk |
| `checkAddressValid` | No | bool | Check address format validity |

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `addressValid` | bool | Valid address format |
| `contract` | bool | Contract address |
| `amlValid` | bool | Subject to risk control limitations |

---

### Coin Balance Snapshot (Team-wide)

Query aggregate balance for specific coins across all accounts in the team:

```python
param = CoinBalanceSnapshotRequest()
param.gmt8Date = "2026-01-01"

result = coin_api.coin_balance_snapshot(param)
for item in result:
    print(f"{item['coinKey']}: {item['coinBalance']}")
```

---

### Get Current Block Height

```python
param = CoinBlockHeightRequest()
param.coinKey = "ETHEREUM_ETH"

result = coin_api.coin_block_height(param)
for item in result:
    print(f"{item['coinKey']} local block height: {item['localBlockHeight']}")
```

---

### List Coins Under Maintenance

```python
maintained = coin_api.list_coin_maintain()
```

---

## AccountApi -- Per-Account Coin Operations

### Add Coin(s) to a Wallet Account (V2 -- Recommended)

Supports adding up to 20 coins in a single call. Adding a token automatically adds its parent mainnet coin.

```python
param = CreateAccountCoinRequestV2()
param.accountKey = "your-account-key"
param.coinKeyList = ["USDT(ERC20)_ETHEREUM_USDT", "ETH(SEPOLIA)_ETHEREUM_SEPOLIA"]

resp = account_api.create_account_coin_v2(param)
for coin in resp.get('coinAddressList', []):
    print(f"  Added coin: {coin['coinKey']} addressList: {coin.get('addressList', [])}")
```

> **V1 (legacy, single coin):** `CreateAccountCoinRequest` with `coinKey` set to a single string -- still supported.

**Notes:**
- If `coinKey` was already added, the response returns the same data as before (idempotent).
- If a previously added coin was manually disabled on the UI, re-adding via API will NOT re-enable it.

---

### List Coins for a Wallet Account

```python
param = ListAccountCoinRequest()
param.accountKey = "your-account-key"

coins = account_api.list_account_coin(param)
for coin in coins:
    addr = coin['addressList'][0]['address'] if coin.get('addressList') else 'N/A'
    print(f"{coin['coinKey']} balance: {coin['balance']} address: {addr}")
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
- Always validate addresses with `check_coin_address` before sending funds or adding to whitelist.
- Team-level balance: use `coin_balance_snapshot`. Per-account balance: use `list_account_coin`.
