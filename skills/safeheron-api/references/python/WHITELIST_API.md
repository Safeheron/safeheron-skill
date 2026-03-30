# Whitelist API Reference

## Imports

```python
from safeheron_api_sdk_python.api.whitelist_api import (
    WhitelistApi,
    CreateWhitelistRequest,
    OneWhitelistRequest,
    ListWhitelistRequest,
    EditWhitelistRequest,
    DeleteWhitelistRequest,
)
```

## Create API Instance

```python
whitelist_api = WhitelistApi(config)
```

---

## Create a Whitelist Entry

```python
param = CreateWhitelistRequest()
param.whitelistName = "Alice's ETH address"   # max 20 chars
param.chainType = "EVM"                        # see chainType values below
param.address = "0xAliceEthAddress..."
# param.memo = "optional memo"               # for TON networks (max 20 chars)
# param.hiddenOnUI = False                   # True = API-only, hidden in Console

resp = whitelist_api.create_whitelist(param)
whitelist_key = resp['whitelistKey']   # save this -- permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | str | Display name (max 20 chars) |
| `chainType` | Yes | str | Blockchain type (see table below) |
| `address` | Yes | str | Blockchain address to whitelist |
| `memo` | No | str | Memo/tag for TON networks (max 20 chars) |
| `hiddenOnUI` | No | bool | If True, hidden from Web Console / App |

### Supported chainType Values

| `chainType` | Supported Networks |
|-------------|-------------------|
| `EVM` | Ethereum, BNB Chain, Polygon, Arbitrum, Base, and all EVM-compatible chains |
| `Bitcoin` | Bitcoin mainnet |
| `Bitcoin Cash` | Bitcoin Cash |
| `Dash` | Dash |
| `TRON` | TRON mainnet |
| `NEAR` | NEAR Protocol |
| `Filecoin` | Filecoin |
| `Sui` | Sui |
| `Aptos` | Aptos |
| `Solana` | Solana |
| `Bitcoin Testnet` | Bitcoin testnet |
| `TON` | TON mainnet |
| `TON_TESTNET` | TON testnet |

---

## Retrieve a Single Whitelist Entry

```python
param = OneWhitelistRequest()
param.whitelistKey = whitelist_key

resp = whitelist_api.one_whitelist(param)
print(f"Name: {resp['whitelistName']}")
print(f"Address: {resp['address']}")
print(f"Status: {resp['whitelistStatus']}")
```

---

## List Whitelist Entries

```python
param = ListWhitelistRequest()
param.limit = 20

entries = whitelist_api.list_whitelist(param)
for entry in entries:
    print(f"{entry['whitelistKey']} {entry['whitelistName']} {entry['whitelistStatus']}")
```

---

## Edit a Whitelist Entry

```python
param = EditWhitelistRequest()
param.whitelistKey = whitelist_key
param.whitelistName = "Updated name"

whitelist_api.edit_whitelist(param)
```

---

## Delete a Whitelist Entry

```python
param = DeleteWhitelistRequest()
param.whitelistKey = whitelist_key

whitelist_api.delete_whitelist(param)
```

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `whitelistKey` | str | Unique identifier |
| `whitelistName` | str | Display name |
| `chainType` | str | Blockchain type |
| `address` | str | Whitelisted address |
| `memo` | str | Memo (TON networks) |
| `whitelistStatus` | str | `AUDIT` / `APPROVED` / `REJECTED` |
| `createTime` | int | Unix timestamp (ms) |
| `lastUpdateTime` | int | Unix timestamp (ms) |

### Whitelist Status Values

| Status | Description |
|--------|-------------|
| `AUDIT` | Pending approval (if policy requires human review) |
| `APPROVED` | Active -- can be used as transaction destination |
| `REJECTED` | Denied -- cannot be used |

---

## Using Whitelist as Transaction Destination

```python
from safeheron_api_sdk_python.api.transaction_api import TransactionApi, CreateTransactionRequest

param = CreateTransactionRequest()
param.destinationAccountType = "WHITELISTING_ACCOUNT"
param.destinationAccountKey = whitelist_key  # pass whitelistKey here
param.coinKey = "ETHEREUM_ETH"
param.txAmount = "0.05"
# ... other fields
```

---

## Best Practices

- Validate address with `coin_api.check_coin_address()` before creating a whitelist entry.
- Use descriptive `whitelistName` and set `chainType` carefully -- wrong chainType causes address validation failure.
- In regulated environments, configure Safeheron Console policy to require human approval before whitelist additions take effect (`whitelistStatus = "AUDIT"` until approved).
- Subscribe to Whitelist webhook events (`WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`) to keep your local state in sync.
