# Web3 API Reference

Web3 Wallet Accounts support arbitrary EVM signing: raw message hashes, arbitrary messages (personalSign), structured data (EIP-712), and raw EVM transactions.

> ⚠️ **Web3 API requires a Web3 wallet account.** Using a regular `VAULT_ACCOUNT` key will result in "account not found" errors.

---

## Imports

```java
import com.safeheron.client.api.Web3ApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
Web3ApiService web3Api = ServiceCreator.create(Web3ApiService.class, safeheronConfig);
```

---

## 1. ethSign — Sign a Raw 32-byte Hash

Used to sign any arbitrary hash (e.g., Ethereum message hash prefix, custom protocol hash).

```java
CreateWeb3EthSignRequest req = new CreateWeb3EthSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);

CreateWeb3EthSignRequest.MessageHash messageHash = new CreateWeb3EthSignRequest.MessageHash();
messageHash.setChainId(1L);  // Ethereum mainnet (informational — not used in signing)
messageHash.setHash(Arrays.asList(
    "a1b2c3d4e5f6..."  // 32-byte hex string(s)
));
req.setMessageHash(messageHash);

TxKeyResult resp = ServiceExecutor.execute(web3Api.createWeb3EthSign(req));
String txKey = resp.getTxKey();
```

---

## 2. personalSign — Sign an Arbitrary Message

Adds Ethereum's `\x19Ethereum Signed Message:\n{len}` prefix automatically.

```java
CreateWeb3PersonalSignRequest req = new CreateWeb3PersonalSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);

CreateWeb3PersonalSignRequest.Message message = new CreateWeb3PersonalSignRequest.Message();
message.setChainId(1L);
message.setData("Hello, Safeheron!");  // raw message text or hex
req.setMessage(message);

TxKeyResult resp = ServiceExecutor.execute(web3Api.createWeb3PersonalSign(req));
String txKey = resp.getTxKey();
```

---

## 3. ethSignTypedData — Sign EIP-712 Structured Data

Supports EIP-712 structured typed data signing (Metamask `eth_signTypedData`).

```java
CreateWeb3EthSignTypedDataRequest req = new CreateWeb3EthSignTypedDataRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);

CreateWeb3EthSignTypedDataRequest.Message message = new CreateWeb3EthSignTypedDataRequest.Message();
message.setChainId(1L);
message.setVersion("v4");  // "v1", "v3", or "v4"
message.setData("{\"types\":{\"EIP712Domain\":[...]}, \"domain\":{...}, \"message\":{...}}");
req.setMessage(message);

TxKeyResult resp = ServiceExecutor.execute(web3Api.createWeb3EthSignTypedData(req));
String txKey = resp.getTxKey();
```

**EIP-712 Version Values:**

| Value | Standard |
|-------|----------|
| `v1` | Legacy typed signing |
| `v3` | EIP-712 draft |
| `v4` | EIP-712 (MetaMask current default) |

---

## 4. ethSignTransaction — Sign a Raw EVM Transaction

Signs (but does NOT broadcast) a raw EVM transaction. Returns the signed transaction hex.

```java
CreateWeb3EthSignTransactionRequest req = new CreateWeb3EthSignTransactionRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);

CreateWeb3EthSignTransactionRequest.Transaction tx =
    new CreateWeb3EthSignTransactionRequest.Transaction();
tx.setChainId(1L);                        // Ethereum mainnet
tx.setTo("0xRecipientAddress");
tx.setValue("1000000000000000");          // amount in wei (string)
tx.setNonce(42L);
tx.setGasLimit(21000L);
// EIP-1559 fee params (preferred over gasPrice):
tx.setMaxFeePerGas("30000000000");        // in wei (string)
tx.setMaxPriorityFeePerGas("1500000000");
// tx.setGasPrice("...");                 // use for legacy (type-0) transactions
tx.setData("");                           // contract call data, or "" for plain transfer
req.setTransaction(tx);

TxKeyResult resp = ServiceExecutor.execute(web3Api.createWeb3EthSignTransaction(req));
String txKey = resp.getTxKey();
```

---

## Poll for Web3 Sign Result

All four sign types use the same polling method:

```java
OneWeb3SignRequest pollReq = new OneWeb3SignRequest();
pollReq.setTxKey(txKey);
// OR: pollReq.setCustomerRefId(customerRefId);

Web3SignResponse result = null;
for (int i = 0; i < 60; i++) {
    result = ServiceExecutor.execute(web3Api.oneWeb3Sign(pollReq));
    String status = result.getTransactionStatus();
    if ("SIGN_COMPLETED".equalsIgnoreCase(status)
            || "FAILED".equalsIgnoreCase(status)
            || "REJECTED".equalsIgnoreCase(status)
            || "CANCELLED".equalsIgnoreCase(status)) {
        break;
    }
    Thread.sleep(3000);
}
```

---

## Retrieve Signature from Response

After `SIGN_COMPLETED`:

```java
// For ethSign:
String sig = result.getMessageHash().getSigList().get(0).getSig();

// For personalSign and ethSignTypedData:
String sig = result.getMessage().getSig().getSig();

// For ethSignTransaction:
String signedTx  = result.getTransaction().getSignedTransaction();  // hex-encoded signed tx
String txHash    = result.getTransaction().getTxHash();             // tx hash
```

---

## Cancel a Web3 Sign Request

```java
CancelWeb3SignRequest req = new CancelWeb3SignRequest();
req.setTxKey(txKey);

ServiceExecutor.execute(web3Api.cancelWeb3Sign(req));
```

**CancelWeb3SignRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Transaction key of the sign request to cancel |

---

## List Web3 Sign Requests

```java
ListWeb3SignRequest req = new ListWeb3SignRequest();
req.setLimit(20L);
// Optional filters:
// req.setTransactionStatus(Arrays.asList("SIGN_COMPLETED"));
// req.setSubjectType("PERSONAL_SIGN");

List<Web3SignResponse> list = ServiceExecutor.execute(web3Api.listWeb3Sign(req));
```

**ListWeb3SignRequest Fields** (extends `LimitSearch`):

Inherited from `LimitSearch`:

| Field | Type | Description |
|-------|------|-------------|
| `direct` | String | Page direction: `NEXT` (default) |
| `limit` | Long | Items per page, max 500 |
| `fromId` | String | txKey of last record from previous page; omit for first page |

Own fields:

| Field | Type | Description |
|-------|------|-------------|
| `subjectType` | String | Web3 sign type (e.g. `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION`) |
| `transactionStatus` | `List<String>` | Filter by status list |
| `accountKey` | String | Source account key |
| `createTimeMin` | Long | Start time, UNIX timestamp (ms); default: `createTimeMax` minus 24h |
| `createTimeMax` | Long | End time, UNIX timestamp (ms); default: current UTC time |

---

## Web3SignResponse Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron sign request key |
| `accountKey` | String | Web3 wallet account key |
| `sourceAddress` | String | Signing address |
| `transactionStatus` | String | `SUBMITTED`, `SIGNING`, `SIGN_COMPLETED`, `FAILED`, `REJECTED`, `CANCELLED` |
| `transactionSubStatus` | String | Detailed sub-status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `customerRefId` | String | Your reference ID |
| `createTime` | Long | Unix timestamp (ms) |
| `messageHash` | Object | For ethSign — contains `chainId`, `sigList` |
| `message` | Object | For personalSign / ethSignTypedData — contains `chainId`, `data`, `sig` |
| `transaction` | Object | For ethSignTransaction — contains all tx fields + `signedTransaction`, `txHash` |

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| ethSign | `CreateWeb3EthSignRequest` | `TxKeyResult` |
| personalSign | `CreateWeb3PersonalSignRequest` | `TxKeyResult` |
| ethSignTypedData | `CreateWeb3EthSignTypedDataRequest` | `TxKeyResult` |
| ethSignTransaction | `CreateWeb3EthSignTransactionRequest` | `TxKeyResult` |
| Get one sign | `OneWeb3SignRequest` | `Web3SignResponse` |
| List signs | `ListWeb3SignRequest` | `List<Web3SignResponse>` |
| Cancel sign | `CancelWeb3SignRequest` | `ResultResponse` |

---

## Web3 Sign Status Flow

```
SUBMITTED → SIGNING → SIGN_COMPLETED
                     └─ FAILED | REJECTED | CANCELLED
```

---

## Best Practices

- **Web3 API requires Web3 wallet**. Regular vault account keys cause "account not found" errors.
- Web3 wallets must have a Web3 signing policy configured — contact Safeheron Support to enable.
- `customerRefId` must be unique (max 100 chars). Duplicate refId returns error code `9001`.
- All sign requests must also be approved via the configured approval policy before signing begins.
