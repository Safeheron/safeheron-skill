# MPC Sign API Reference

## Imports
```java
import com.safeheron.client.api.MPCSignApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.CreateMPCSignTransactionRequest;
import com.safeheron.client.request.OneMPCSignTransactionsRequest;
import com.safeheron.client.response.MPCSignTransactionsResponse;
import com.safeheron.client.response.TxKeyResult;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service
```java
MPCSignApiService mpcSignApi = ServiceCreator.create(MPCSignApiService.class, safeheronConfig);
```

---

## Create an MPC Sign Transaction
```java
CreateMPCSignTransactionRequest request = new CreateMPCSignTransactionRequest();
request.setCustomerRefId(UUID.randomUUID().toString());
request.setSourceAccountKey(accountKey);
request.setSignAlg("Secp256k1");  // "Secp256k1" or "Ed25519"

// Add hash(es) to sign — use the inner Date class
CreateMPCSignTransactionRequest.Date hashItem = new CreateMPCSignTransactionRequest.Date();
hashItem.setData(hashHexWithoutPrefix);  // 32-byte hex, no "0x" prefix
request.setDataList(Arrays.asList(hashItem));

TxKeyResult response = ServiceExecutor.execute(mpcSignApi.createMPCSignTransactions(request));
String txKey = response.getTxKey();
```

## Poll for MPC Sign Result
```java
OneMPCSignTransactionsRequest pollRequest = new OneMPCSignTransactionsRequest();
pollRequest.setCustomerRefId(customerRefId);   // OR setTxKey(txKey)

for (int i = 0; i < 100; i++) {
    MPCSignTransactionsResponse resp = ServiceExecutor.execute(
            mpcSignApi.oneMPCSignTransactions(pollRequest));

    String status = resp.getTransactionStatus();
    String subStatus = resp.getTransactionSubStatus();
    System.out.println("status: " + status + ", sub: " + subStatus);

    if ("FAILED".equalsIgnoreCase(status) || "REJECTED".equalsIgnoreCase(status) || "CANCELLED".equalsIgnoreCase(status)) {
        throw new RuntimeException("MPC sign failed/rejected/cancelled");
    }
    if ("COMPLETED".equalsIgnoreCase(status) && "CONFIRMED".equalsIgnoreCase(subStatus)) {
        String sig = resp.getDataList().get(0).getSig();
        // sig = R (64 hex chars) + S (64 hex chars) + V (2 hex chars)
        break;
    }
    Thread.sleep(5000);
}
```

---

## Key Classes
| Class | Description |
|-------|-------------|
| `CreateMPCSignTransactionRequest` | Request to initiate MPC signing |
| `CreateMPCSignTransactionRequest.Date` | Inner class for each hash item (field: `data`) |
| `TxKeyResult` | Response from create — contains `txKey` |
| `OneMPCSignTransactionsRequest` | Request to query one MPC sign tx |
| `MPCSignTransactionsResponse` | Query response — contains status and signed data list |

**MPCSignTransactionsResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Transaction key |
| `transactionStatus` | String | Transaction status |
| `transactionSubStatus` | String | Transaction sub-status |
| `createTime` | Long | Creation time, UNIX timestamp (ms) |
| `sourceAccountKey` | String | Source account key |
| `auditUserKey` | String | Final approver key |
| `auditUserName` | String | Final approver username |
| `createdByUserKey` | String | Creator key |
| `createdByUserName` | String | Creator username |
| `customerRefId` | String | Merchant unique business ID |
| `customerExt1` | String | Merchant extended field |
| `customerExt2` | String | Merchant extended field |
| `signAlg` | String | Signature algorithm |
| `dataList` | `List<Date>` | List of signed data items |

**MPCSignTransactionsResponse.Date Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `note` | String | Transaction note (max 180 chars) |
| `data` | String | Transaction data that was signed |
| `sig` | String | Signature: 32 bytes R + 32 bytes S + 1 byte V (hex concatenated) |

---

## Signature Format
The `sig` field from `resp.getDataList().get(0).getSig()` is a concatenated hex string:
- Characters 0–63: `R`
- Characters 64–127: `S`
- Characters 128–129: `V` (hex, add 27 for legacy Ethereum)

```java
private Sign.SignatureData convertSig(String sig) {
    String sigR = sig.substring(0, 64);
    String sigS = sig.substring(64, 128);
    String sigV = sig.substring(128);
    Integer v = Integer.parseInt(sigV, 16) + 27;
    return new Sign.SignatureData(
            v.byteValue(),
            Numeric.hexStringToByteArray(sigR),
            Numeric.hexStringToByteArray(sigS));
}
```

---

## Full ERC-20 Transfer via MPC Sign (Web3j)

This is the exact pattern from the official SDK test (`MpcSignTest.java`):

```java
// 1. Build ERC-20 transfer function
Function function = new Function(
    "transfer",
    Arrays.asList(new Address(toAddress), new Uint256(tokenValue.toBigInteger())),
    Arrays.asList(new TypeReference<Type>() {}));
String data = FunctionEncoder.encode(function);

// 2. Build EIP-1559 raw transaction (web3j)
RawTransaction rawTx = RawTransaction.createTransaction(
    chainId.longValue(), nonce, gasLimit, contractAddress,
    BigInteger.ZERO, data, maxPriorityFeePerGas, maxFeePerGas);

// 3. Hash the encoded transaction
byte[] encodedRawTx = TransactionEncoder.encode(rawTx);
String hash = Numeric.toHexString(Hash.sha3(encodedRawTx));

// 4. Submit hash (without "0x") to Safeheron MPC Sign
String customerRefId = UUID.randomUUID().toString();
CreateMPCSignTransactionRequest req = new CreateMPCSignTransactionRequest();
req.setCustomerRefId(customerRefId);
req.setSourceAccountKey(accountKey);
req.setSignAlg("Secp256k1");
CreateMPCSignTransactionRequest.Date hashItem = new CreateMPCSignTransactionRequest.Date();
hashItem.setData(hash.substring(2));  // strip "0x"
req.setDataList(Arrays.asList(hashItem));
TxKeyResult txKeyResult = ServiceExecutor.execute(mpcSignApi.createMPCSignTransactions(req));

// 5. Poll until COMPLETED/CONFIRMED
OneMPCSignTransactionsRequest pollReq = new OneMPCSignTransactionsRequest();
pollReq.setCustomerRefId(customerRefId);
MPCSignTransactionsResponse pollResp = null;
for (int i = 0; i < 100; i++) {
    pollResp = ServiceExecutor.execute(mpcSignApi.oneMPCSignTransactions(pollReq));
    if ("COMPLETED".equalsIgnoreCase(pollResp.getTransactionStatus())
            && "CONFIRMED".equalsIgnoreCase(pollResp.getTransactionSubStatus())) break;
    Thread.sleep(5000);
}
String sig = pollResp.getDataList().get(0).getSig();

// 6. Assemble signed transaction and broadcast
byte[] signedMessage = TransactionEncoder.encode(rawTx, convertSig(sig));
String ethTxHash = web3j.ethSendRawTransaction(Numeric.toHexString(signedMessage))
        .send().getTransactionHash();
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
| `direct` | String | No | Page direction: `NEXT` (default) / `PREV` |
| `limit` | Long | No | Items per page, max 500 |
| `fromId` | String | No | Cursor: omit on first page; pass the `txKey` of the last item from the previous page |
| `createTimeMin` | Long (ms) | No | Start of creation time range (default: `createTimeMax` minus 24 hours) |
| `createTimeMax` | Long (ms) | No | End of creation time range (default: current UTC time) |

```java
MPCSignApiService mpcSignApi = ServiceCreator.create(MPCSignApiService.class, config);

ListMPCSignTransactionsRequest req = new ListMPCSignTransactionsRequest();
req.setLimit(50L);
req.setCreateTimeMin(System.currentTimeMillis() - 86400000L); // last 24h

List<MPCSignTransactionsResponse> list = ServiceExecutor.execute(mpcSignApi.listMPCSignTransactions(req));
for (MPCSignTransactionsResponse tx : list) {
    System.out.println(tx.getTxKey() + " " + tx.getTransactionStatus());
    for (MPCSignTransactionsResponse.Date item : tx.getDataList()) {
        System.out.println("  sig: " + item.getSig());
    }
}

// Next page: pass txKey of the last item as fromId
if (!list.isEmpty()) {
    req.setFromId(list.get(list.size() - 1).getTxKey());
    List<MPCSignTransactionsResponse> nextPage = ServiceExecutor.execute(mpcSignApi.listMPCSignTransactions(req));
}
```

---

## Best Practices

- The MPCSign policy is also required before use — contact Safeheron Support to enable.
- **Hash format** — `data` must be exactly 32 bytes (64 hex chars) with **no `0x` prefix**. For Ethereum, use `Hash.sha3(TransactionEncoder.encode(rawTx))` and strip the leading `"0x"` before submitting.
- **Choose the correct `signAlg`** — mismatch between algorithm and chain causes signing failure.
- `customerRefId` must be unique (max 100 chars). On timeout, **retry with the same `customerRefId`** — Safeheron returns the original request (idempotency). Error code `9001` means the refId already exists.
- **Handle all terminal statuses** when polling — `FAILED`, `REJECTED`, `CANCELLED` should all throw/abort, not just be ignored.
- **Approval policy applies** — every MPC Sign request goes through the configured approval workflow before signing begins; factor the approval wait time into your polling timeout.
