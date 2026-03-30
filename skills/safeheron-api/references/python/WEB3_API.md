# Web3 API Reference

Web3 Wallet Accounts support arbitrary EVM signing: raw message hashes, arbitrary messages (personalSign), structured data (EIP-712), and raw EVM transactions.

> **Web3 API requires a Web3 wallet account.** Using a regular `VAULT_ACCOUNT` key will result in "account not found" errors.

---

## Imports

```python
import uuid
from safeheron_api_sdk_python.api.web3_api import (
    Web3Api,
    CreateWeb3EthSignRequest,
    CreateWeb3PersonalSignRequest,
    CreateWeb3EthSignTypedDataRequest,
    CreateWeb3EthSignTransactionRequest,
    OneWeb3SignRequest,
)
```

## Create API Instance

```python
web3_api = Web3Api(config)
```

---

## 1. ethSign -- Sign a Raw 32-byte Hash

Used to sign any arbitrary hash (e.g., Ethereum message hash prefix, custom protocol hash).

```python
param = CreateWeb3EthSignRequest()
param.customerRefId = str(uuid.uuid4())
param.accountKey = web3_account_key
param.messageHash.chainId = 1   # Ethereum mainnet (informational)
param.messageHash.hash = ["a1b2c3d4e5f6..."]  # 32-byte hex string(s)

resp = web3_api.createWeb3EthSign(param)
tx_key = resp['txKey']
```

---

## 2. personalSign -- Sign an Arbitrary Message

Adds Ethereum's `\x19Ethereum Signed Message:\n{len}` prefix automatically.

```python
param = CreateWeb3PersonalSignRequest()
param.customerRefId = str(uuid.uuid4())
param.accountKey = web3_account_key
param.message.chainId = 1
param.message.data = "Hello, Safeheron!"  # raw message text or hex

resp = web3_api.createWeb3PersonalSign(param)
tx_key = resp['txKey']
```

---

## 3. ethSignTypedData -- Sign EIP-712 Structured Data

Supports EIP-712 structured typed data signing (Metamask `eth_signTypedData`).

```python
param = CreateWeb3EthSignTypedDataRequest()
param.customerRefId = str(uuid.uuid4())
param.accountKey = web3_account_key
param.message.chainId = 1
param.message.version = "ETH_SIGNTYPEDDATA_V4"  # "v1", "v3", "v4", or "ETH_SIGNTYPEDDATA_V4"
param.message.data = '{"types":{...}, "domain":{...}, "message":{...}}'

resp = web3_api.createWeb3EthSignTypedData(param)
tx_key = resp['txKey']
```

---

## 4. ethSignTransaction -- Sign a Raw EVM Transaction

Signs (but does NOT broadcast) a raw EVM transaction.

```python
param = CreateWeb3EthSignTransactionRequest()
param.customerRefId = str(uuid.uuid4())
param.accountKey = web3_account_key
param.transaction.to = "0xRecipientAddress"
param.transaction.value = 0
param.transaction.chainId = 1
param.transaction.gasLimit = 21000
param.transaction.gasPrice = str(gas_price)
param.transaction.nonce = nonce
param.transaction.maxFeePerGas = str(max_fee_per_gas)
param.transaction.maxPriorityFeePerGas = str(max_priority_fee_per_gas)
param.transaction.data = ""  # contract call data, or "" for plain transfer

resp = web3_api.createWeb3EthSignTransaction(param)
tx_key = resp['txKey']
```

---

## Poll for Web3 Sign Result

All four sign types use the same polling method:

```python
import time

query_param = OneWeb3SignRequest()
query_param.txKey = tx_key

result = None
for i in range(60):
    result = web3_api.oneWeb3Sign(query_param)
    status = result['transactionStatus']
    if status in ('SIGN_COMPLETED', 'FAILED', 'REJECTED', 'CANCELLED'):
        break
    time.sleep(3)
```

---

## Retrieve Signature from Response

After `SIGN_COMPLETED`:

```python
# For ethSign:
sig = result['messageHash']['sigList'][0]['sig']

# For personalSign and ethSignTypedData:
sig = result['message']['sig']['sig']

# For ethSignTransaction:
signed_tx = result['transaction']['signedTransaction']  # hex-encoded signed tx
tx_hash = result['transaction']['txHash']
```

---

## Full ERC-20 Transfer via Web3 ethSign (web3.py)

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
from safeheron_api_sdk_python.api.web3_api import (
    Web3Api, CreateWeb3EthSignRequest, OneWeb3SignRequest,
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


def combine_unsigned_transaction_and_sig(transaction_dict, sig_hex):
    unsigned_tx = serializable_unsigned_transaction_from_dict(transaction_dict)
    if isinstance(unsigned_tx, UnsignedTransaction):
        v, r, s = get_v_r_s_from_sig(sig_hex, chain_id=None)
    elif isinstance(unsigned_tx, Transaction):
        v, r, s = get_v_r_s_from_sig(sig_hex, chain_id=unsigned_tx.v)
    elif isinstance(unsigned_tx, TypedTransaction):
        v, r, s = raw_v_r_s_from_sig(sig_hex)
    else:
        raise TypeError(f"Unknown Transaction type: {type(unsigned_tx)}")
    return encode_transaction(unsigned_tx, vrs=(v, r, s))


# 1. Build transaction
py_web3 = Web3(Web3.HTTPProvider("https://your-rpc-url"))
transaction = {
    'nonce': py_web3.eth.get_transaction_count(sender_address),
    'to': token_address,
    'data': data,
    'value': 0,
    'chainId': py_web3.eth.chain_id,
    'gas': gas_limit,
    'maxPriorityFeePerGas': max_priority_fee_per_gas,
    'maxFeePerGas': max_fee_per_gas,
}

# 2. Hash the transaction
unsigned_tx = serializable_unsigned_transaction_from_dict(transaction)
tx_hash = unsigned_tx.hash()

# 3. Submit to Safeheron Web3 ethSign
web3_api = Web3Api(config)
param = CreateWeb3EthSignRequest()
param.accountKey = web3_account_key
param.customerRefId = str(uuid.uuid4())
param.messageHash.chainId = transaction['chainId']
param.messageHash.hash = [tx_hash.hex()]

resp = web3_api.createWeb3EthSign(param)
tx_key = resp['txKey']

# 4. Poll for signature
query_param = OneWeb3SignRequest()
query_param.txKey = tx_key
for i in range(100):
    result = web3_api.oneWeb3Sign(query_param)
    if result['transactionStatus'] == 'SIGN_COMPLETED':
        break
    time.sleep(5)

sig = result['messageHash']['sigList'][0]['sig']

# 5. Assemble signed transaction and broadcast
signed_raw_tx = combine_unsigned_transaction_and_sig(transaction, sig)
on_chain_hash = py_web3.eth.send_raw_transaction(signed_raw_tx.hex()).hex()
print(f"tx_hash: {on_chain_hash}")
```

---

## Web3SignResponse Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron sign request key |
| `accountKey` | str | Web3 wallet account key |
| `sourceAddress` | str | Signing address |
| `transactionStatus` | str | `SUBMITTED`, `SIGNING`, `SIGN_COMPLETED`, `FAILED`, `REJECTED`, `CANCELLED` |
| `subjectType` | str | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `customerRefId` | str | Your reference ID |

---

## Web3 Sign Status Flow

```
SUBMITTED -> SIGNING -> SIGN_COMPLETED
                      +-- FAILED | REJECTED | CANCELLED
```

---

## Best Practices

- **Web3 API requires Web3 wallet**. Regular vault account keys cause "account not found" errors.
- Web3 wallets must have a Web3 signing policy configured -- contact Safeheron Support to enable.
- `customerRefId` must be unique (max 100 chars). Duplicate refId returns error code `9001`.
- All sign requests must also be approved via the configured approval policy before signing begins.
