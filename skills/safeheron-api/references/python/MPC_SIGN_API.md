# MPC Sign API Reference

## Imports

```python
import uuid
import time
from safeheron_api_sdk_python.api.mpc_sign_api import (
    MPCSignApi,
    CreateMPCSignTransactionRequest,
    OneMPCSignTransactionsRequest,
)
```

## Create API Instance

```python
mpc_sign_api = MPCSignApi(config)
```

---

## Create an MPC Sign Transaction

```python
param = CreateMPCSignTransactionRequest()
param.customerRefId = str(uuid.uuid4())
param.sourceAccountKey = account_key
param.signAlg = "Secp256k1"  # "Secp256k1" or "Ed25519"

# Add hash(es) to sign -- 32-byte hex, no "0x" prefix
param.dataList[0].data = hash_hex_without_prefix

resp = mpc_sign_api.create_mpc_sign_transactions(param)
tx_key = resp['txKey']
```

## Poll for MPC Sign Result

```python
query_param = OneMPCSignTransactionsRequest()
query_param.txKey = tx_key
# OR: query_param.customerRefId = customer_ref_id

for i in range(100):
    result = mpc_sign_api.one_mpc_sign_transactions(query_param)
    status = result['transactionStatus']
    sub_status = result.get('transactionSubStatus', '')
    print(f"status: {status}, sub: {sub_status}")

    if status in ('FAILED', 'REJECTED', 'CANCELLED'):
        raise RuntimeError("MPC sign failed/rejected/cancelled")

    if status == 'COMPLETED' and sub_status == 'CONFIRMED':
        sig = result['dataList'][0]['sig']
        # sig = R (64 hex chars) + S (64 hex chars) + V (2 hex chars)
        break

    time.sleep(5)
```

---

## Key Classes

| Class | Description |
|-------|-------------|
| `CreateMPCSignTransactionRequest` | Request to initiate MPC signing |
| `OneMPCSignTransactionsRequest` | Request to query one MPC sign tx |

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Transaction key |
| `transactionStatus` | str | Transaction status |
| `transactionSubStatus` | str | Transaction sub-status |
| `createTime` | int | Creation time, UNIX timestamp (ms) |
| `sourceAccountKey` | str | Source account key |
| `customerRefId` | str | Merchant unique business ID |
| `signAlg` | str | Signature algorithm |
| `dataList` | list | List of signed data items |

**dataList item Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `note` | str | Transaction note (max 180 chars) |
| `data` | str | Transaction data that was signed |
| `sig` | str | Signature: 32 bytes R + 32 bytes S + 1 byte V (hex concatenated) |

---

## Signature Format

The `sig` field is a concatenated hex string:
- Characters 0-63: `R`
- Characters 64-127: `S`
- Characters 128-129: `V` (hex, add 27 for legacy Ethereum)

```python
def parse_sig(sig_hex):
    sig_r = sig_hex[0:64]
    sig_s = sig_hex[64:128]
    sig_v = sig_hex[128:]
    v = int(sig_v, 16) + 27
    r = int(sig_r, 16)
    s = int(sig_s, 16)
    return v, r, s
```

---

## Full ERC-20 Transfer via MPC Sign (web3.py)

This is the pattern from the official SDK demo:

```python
import uuid
import time
from web3 import Web3
from eth_account._utils.legacy_transactions import (
    serializable_unsigned_transaction_from_dict,
    encode_transaction,
    UnsignedTransaction,
    Transaction,
)
from eth_account._utils.typed_transactions import TypedTransaction
from safeheron_api_sdk_python.api.mpc_sign_api import (
    MPCSignApi, CreateMPCSignTransactionRequest, OneMPCSignTransactionsRequest,
)


def to_eth_v(v_raw, chain_id=None):
    if chain_id is None:
        return v_raw + 27
    return v_raw + 35 + 2 * chain_id


def get_v_r_s_from_sig(sig_hex, chain_id=None):
    sig_bytes = bytes.fromhex(sig_hex)
    r = int.from_bytes(sig_bytes[0:32], 'big')
    s = int.from_bytes(sig_bytes[32:64], 'big')
    v = int(sig_bytes[64])
    v = to_eth_v(v, chain_id)
    return v, r, s


def raw_v_r_s_from_sig(sig_hex):
    sig_bytes = bytes.fromhex(sig_hex)
    r = int.from_bytes(sig_bytes[0:32], 'big')
    s = int.from_bytes(sig_bytes[32:64], 'big')
    v = int(sig_bytes[64])
    return v, r, s


def combine_unsigned_transaction_and_sig(tx_dict, sig_hex):
    unsigned_tx = serializable_unsigned_transaction_from_dict(tx_dict)
    if isinstance(unsigned_tx, UnsignedTransaction):
        v, r, s = get_v_r_s_from_sig(sig_hex, chain_id=None)
    elif isinstance(unsigned_tx, Transaction):
        v, r, s = get_v_r_s_from_sig(sig_hex, chain_id=unsigned_tx.v)
    elif isinstance(unsigned_tx, TypedTransaction):
        v, r, s = raw_v_r_s_from_sig(sig_hex)
    else:
        raise TypeError(f"Unknown Transaction type: {type(unsigned_tx)}")
    return encode_transaction(unsigned_tx, vrs=(v, r, s))


# 1. Build ERC-20 transfer transaction
py_web3 = Web3(Web3.HTTPProvider("https://your-rpc-url"))
nonce = py_web3.eth.get_transaction_count(sender_address)
amount_in_wei = amount * 10 ** decimals

transfer_function_signature = 'transfer(address,uint256)'
data = py_web3.to_hex(
    py_web3.keccak(text=transfer_function_signature)[:4]
    + py_web3.to_bytes(hexstr=recipient_address[2:].zfill(64))
    + amount_in_wei.to_bytes(32, byteorder='big')
)

latest_block = py_web3.eth.get_block('latest')
max_priority_fee_per_gas = py_web3.to_wei(2, "gwei")
base_fee_per_gas = latest_block['baseFeePerGas']
max_fee_per_gas = base_fee_per_gas * 2 + max_priority_fee_per_gas

transaction = {
    'nonce': nonce,
    'to': token_address,
    'data': data,
    'value': 0,
    'chainId': py_web3.eth.chain_id,
    'gas': gas_limit,
    'maxPriorityFeePerGas': max_priority_fee_per_gas,
    'maxFeePerGas': max_fee_per_gas,
}

# 2. Hash the encoded transaction
unsigned_tx = serializable_unsigned_transaction_from_dict(transaction)
tx_hash = unsigned_tx.hash()

# 3. Submit hash (without "0x") to Safeheron MPC Sign
mpc_sign_api = MPCSignApi(config)
param = CreateMPCSignTransactionRequest()
param.signAlg = "Secp256k1"
param.sourceAccountKey = account_key
param.customerRefId = str(uuid.uuid4())
param.dataList[0].data = tx_hash.hex()  # no "0x" prefix

resp = mpc_sign_api.create_mpc_sign_transactions(param)
tx_key = resp['txKey']

# 4. Poll until COMPLETED/CONFIRMED
query_param = OneMPCSignTransactionsRequest()
query_param.txKey = tx_key
for i in range(100):
    result = mpc_sign_api.one_mpc_sign_transactions(query_param)
    if result['transactionStatus'] == 'COMPLETED':
        break
    time.sleep(5)

sig = result['dataList'][0]['sig']

# 5. Assemble signed transaction and broadcast
signed_raw_tx = combine_unsigned_transaction_and_sig(transaction, sig)
on_chain_hash = py_web3.eth.send_raw_transaction(signed_raw_tx.hex()).hex()
print(f"tx_hash: {on_chain_hash}")
```

---

## Sign Algorithm Values

| Value | Curve | Use Case |
|-------|-------|----------|
| `"Secp256k1"` | secp256k1 | Ethereum, Bitcoin, BSC, Polygon |
| `"Ed25519"` | Ed25519 | Solana, Near, Aptos |

---

## MPC Sign Status Reference

| Status | Type | Description |
|--------|------|-------------|
| `SUBMITTED` | Initial | Task submitted and accepted by Safeheron |
| `SIGNING` | In-progress | MPC signing in progress after approval |
| `COMPLETED` | Final | Signature successfully completed |
| `FAILED` | Final | Task will no longer be processed |
| `REJECTED` | Final | Rejected by team member's approval |
| `CANCELLED` | Final | Cancelled by team member or system |

```
SUBMITTED -> SIGNING -> COMPLETED
                     |-- FAILED
                     |-- REJECTED
                     |-- CANCELLED
```

---

## List MPC Sign Transactions

Query MPC sign records by time range with cursor-based pagination.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `direct` | str | No | Page direction: `NEXT` (default) / `PREV` |
| `limit` | int | No | Items per page, max 500 |
| `fromId` | str | No | Cursor: omit on first page; pass the `txKey` of the last item from the previous page |
| `createTimeMin` | int (ms) | No | Start of creation time range (default: `createTimeMax` minus 24 hours) |
| `createTimeMax` | int (ms) | No | End of creation time range (default: current UTC time) |

```python
req = ListMPCSignTransactionsRequest()
req.limit = 50
req.createTimeMin = int(time.time() * 1000) - 86400000  # last 24h

result = mpc_sign_api.list_mpc_sign_transactions(req)
for tx in result:
    print(tx['txKey'], tx['transactionStatus'])
    for item in tx.get('dataList', []):
        print('  sig:', item['sig'])

# Next page: pass txKey of the last item as fromId
if result:
    req.fromId = result[-1]['txKey']
    next_page = mpc_sign_api.list_mpc_sign_transactions(req)
```

---

## Best Practices

- The MPCSign policy is also required before use -- contact Safeheron Support to enable.
- **Hash format** -- `data` must be exactly 32 bytes (64 hex chars) with **no `0x` prefix**. For Ethereum, use `unsigned_tx.hash().hex()`.
- **Choose the correct `signAlg`** -- mismatch between algorithm and chain causes signing failure.
- `customerRefId` must be unique (max 100 chars). On timeout, **retry with the same `customerRefId`** -- Safeheron returns the original request (idempotency). Error code `9001` means the refId already exists.
- **Handle all terminal statuses** when polling -- `FAILED`, `REJECTED`, `CANCELLED` should all raise exceptions, not just be ignored.
- **Approval policy applies** -- every MPC Sign request goes through the configured approval workflow before signing begins.
