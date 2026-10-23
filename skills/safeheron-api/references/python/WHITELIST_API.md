# Whitelist API Reference

## Imports

```python
from safeheron_api_sdk_python.api.whitelist_api import (
    WhitelistApi,
    CreateWhitelistRequest,
    CreateFromTransactionWhitelistRequest,
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
param.whitelistName = "Alice's ETH address"   # unique, max 30 chars
param.chainType = "EVM"                        # see chainType values below
param.address = "0xAliceEthAddress..."
# param.memo = "123456"                      # TON / XRP Ledger only (XRPL: numeric string 0-4294967295)
# param.hiddenOnUI = False                   # True = API-only, hidden in Console
# param.note = "Alice's cold wallet"         # optional note

resp = whitelist_api.create_whitelist(param)
whitelist_key = resp['whitelistKey']   # save this -- permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | str | Whitelist unique name, 30 characters max |
| `chainType` | Yes | str | Blockchain type (see table below) |
| `address` | Yes | str | Blockchain address to whitelist; format must meet the chain's requirements |
| `memo` | No | str | Memo/tag of the address (up to 20 chars; for **XRP Ledger** it must be a numeric string in 0–4294967295). Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `hiddenOnUI` | No | bool | If True, hidden from Web Console / App (API-only management and use) |
| `note` | No | str | Note |

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
| `Litecoin` | Litecoin mainnet |
| `Dogecoin` | Dogecoin mainnet |
| `Liquid` | Liquid mainnet |
| `XRP Ledger` | XRP Ledger mainnet (native XRP only, no other tokens yet) |

---

## Retrieve a Single Whitelist Entry

Look up by `whitelistKey` **or** by `address` (at least one is required; if both are given, `whitelistKey` wins):

```python
param = OneWhitelistRequest()
param.whitelistKey = whitelist_key
# or: param.address = "0xAliceEthAddress..."

resp = whitelist_api.one_whitelist(param)
print(f"Name: {resp['whitelistName']}")
print(f"Address: {resp['address']}")
print(f"Status: {resp['whitelistStatus']}")
print(f"Note: {resp.get('note')}")
```

---

## List Whitelist Entries

```python
param = ListWhitelistRequest()
# param.chainType = "EVM"  # optional filter: passing it returns only that chain; omit it to list all chains
param.limit = 20

entries = whitelist_api.list_whitelist(param)   # NOTE: only entries created in the last 24 hours (see below)
for entry in entries:
    print(f"{entry['whitelistKey']} {entry['whitelistName']} {entry['whitelistStatus']}")
```

> `chainType`: published docs mark it as required for List Whitelist Data, but the service treats it as an optional filter — passing it returns only that chain; omit it to list all chains.

> ⚠️ **Time window**: `createTimeMin` defaults to `createTimeMax` minus **24 hours** and `createTimeMax` defaults to now, so a call without time bounds returns only entries created in the last 24 hours. To list **all** entries, set `createTimeMin` to an early timestamp and page with `fromId`.

**List ALL whitelist entries (cursor paging):**

```python
PAGE_SIZE = 500  # max 500


def list_all_whitelists(whitelist_api, chain_type=None):
    result = []
    from_id = None
    while True:
        param = ListWhitelistRequest()
        param.createTimeMin = 1          # override the 24-hour default window
        param.direct = "NEXT"
        param.limit = PAGE_SIZE
        if from_id:
            param.fromId = from_id       # whitelistKey of the last record of the previous page
        if chain_type:
            param.chainType = chain_type # optional filter
        page = whitelist_api.list_whitelist(param) or []
        result.extend(page)
        if len(page) < PAGE_SIZE:
            return result
        from_id = page[-1]['whitelistKey']
```

**ListWhitelistRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `direct` | No | str | Page direction, `NEXT` by default |
| `limit` | No | int | Items per page, max 500 |
| `fromId` | No | str | `whitelistKey` of the last record of the previous page; omit for the first page |
| `chainType` | Yes (per API docs) | str | See chainType values above — the service treats it as an optional filter |
| `whitelistStatus` | No | str | Whitelist status filter |
| `createTimeMin` | No | int | Start of creation time, UNIX ms. **Default: `createTimeMax` minus 24 hours** |
| `createTimeMax` | No | int | End of creation time, UNIX ms. Default: current UTC time |

---

## Edit a Whitelist Entry

```python
param = EditWhitelistRequest()
param.whitelistKey = whitelist_key
param.whitelistName = "Updated name"           # required
param.address = "0xAliceEthAddress..."         # required (pass the current address if unchanged)
# param.memo = "123456"                        # TON / XRP Ledger only — overwrites the stored memo; pass the current one to keep it
# param.force = False                          # see field table
# param.note = "updated note"                  # overwrites the stored note — pass the current note to keep it

whitelist_api.edit_whitelist(param)
```

### EditWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistKey` | Yes | str | Unique identifier of the whitelist entry to edit |
| `whitelistName` | Yes | str | Whitelist unique name, 30 characters max |
| `address` | Yes | str | Public blockchain address and the address format needs to meet the requirements of the chain |
| `memo` | No | str | Memo/tag of the address (up to 20 chars; for **XRP Ledger** it must be a numeric string in 0–4294967295). Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `force` | No | bool | When the whitelist is involved in a transaction approval policy, modifications will result in the new whitelist being directly applied to the approval policy. False by default, meaning that when involved in a transaction approval policy, it will not be modified |
| `note` | No | str | Note |

---

## Delete a Whitelist Entry

```python
param = DeleteWhitelistRequest()
param.whitelistKey = whitelist_key

whitelist_api.delete_whitelist(param)
```

---

## Create Whitelist from an Existing Transaction

Whitelists the destination address of a transaction. The transaction must be a `COMPLETED` transfer from an asset wallet (not Web3 / MPC Sign) whose destination is a one-time address:

```python
param = CreateFromTransactionWhitelistRequest()
param.txKey = tx_key
param.whitelistName = "Recipient from TX-001"  # unique, max 30 chars
param.destinationAddress = "0xRecipient..."     # required: the destination address in the transaction record (case-sensitive)
# param.memo = "123456"                         # TON / XRP Ledger: must match the memo in the transaction record, if one was set
# param.hiddenOnUI = False
# param.note = "optional note"

resp = whitelist_api.create_from_transaction_whitelist(param)
whitelist_key = resp['whitelistKey']
```

### CreateFromTransactionWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | str | Whitelist unique name, 30 characters max |
| `txKey` | Yes | str | Transaction key |
| `destinationAddress` | Yes | str | The destination address in the transaction record; case-sensitive |
| `memo` | No | str | Memo (up to 20 chars; XRPL: numeric string 0–4294967295). For TON / XRP Ledger, if the destination memo was set in the transaction, a matching memo must be provided |
| `hiddenOnUI` | No | bool | If True, hidden from Web Console / App |
| `note` | No | str | Note |

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `whitelistKey` | str | Unique identifier |
| `whitelistName` | str | Display name |
| `chainType` | str | Blockchain type |
| `address` | str | Whitelisted address |
| `memo` | str | Memo of the address when creating the whitelist (TON / XRP Ledger) |
| `whitelistStatus` | str | `AUDIT` / `APPROVED` / `REJECTED` |
| `hiddenOnUI` | bool | Visibility in Web Console / App (True = invisible) |
| `note` | str | Note |
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
