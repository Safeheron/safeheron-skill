# MPC Sign API Reference

## Imports

```go
import (
    "fmt"
    "time"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
    "github.com/google/uuid"
)
```

## Create API Instance

```go
mpcSignApi := api.MpcSignApi{Client: sc}
```

---

## Create an MPC Sign Transaction

```go
customerRefId := uuid.New().String()
req := api.CreateMpcSignRequest{
    CustomerRefId:    customerRefId,
    SourceAccountKey: accountKey,
    SignAlg:          "Secp256k1", // "Secp256k1" or "Ed25519"
    DataList: []struct {
        Data string `json:"data,omitempty"`
        Note string `json:"note,omitempty"`
    }{
        {Data: hashHexWithoutPrefix}, // 32-byte hex, no "0x" prefix
    },
}

var resp api.CreateMpcSignResponse
if err := mpcSignApi.CreateMpcSign(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create MPC sign: %w", err))
}
txKey := resp.TxKey
```

---

## Poll for MPC Sign Result

```go
func retrieveSig(mpcSignApi api.MpcSignApi, customerRefId string) string {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    req := api.OneMPCSignTransactionsRequest{
        CustomerRefId: customerRefId,
    }

    var resp api.MPCSignTransactionsResponse
    for range ticker.C {
        if err := mpcSignApi.OneMPCSignTransactions(req, &resp); err != nil {
            panic(err)
        }

        fmt.Printf("MPC sign status: %s, sub: %s\n",
            resp.TransactionStatus, resp.TransactionSubStatus)

        if resp.TransactionStatus == "FAILED" || resp.TransactionStatus == "REJECTED" || resp.TransactionStatus == "CANCELLED" {
            panic("MPC sign task was FAILED, REJECTED, or CANCELLED")
        }

        if resp.TransactionStatus == "COMPLETED" && resp.TransactionSubStatus == "CONFIRMED" {
            return resp.DataList[0].Sig
            // sig = R (64 hex chars) + S (64 hex chars) + V (2 hex chars)
        }
    }
    panic("timeout waiting for MPC sign result")
}
```

---

## Key Types

| Type | Description |
|------|-------------|
| `CreateMpcSignRequest` | Request to initiate MPC signing |
| `CreateMpcSignResponse` | Response from create -- contains `TxKey` |
| `OneMPCSignTransactionsRequest` | Request to query one MPC sign tx |
| `MPCSignTransactionsResponse` | Query response -- contains status and signed data list |

**MPCSignTransactionsResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | `string` | Transaction key |
| `TransactionStatus` | `string` | Transaction status |
| `TransactionSubStatus` | `string` | Transaction sub-status |
| `CreateTime` | `int64` | Creation time, UNIX timestamp (ms) |
| `SourceAccountKey` | `string` | Source account key |
| `CustomerRefId` | `string` | Merchant unique business ID |
| `SignAlg` | `string` | Signature algorithm |
| `DataList` | `[]struct{Data, Sig, Note}` | List of signed data items |

---

## Signature Format

The `Sig` field is a concatenated hex string:
- Characters 0-63: `R`
- Characters 64-127: `S`
- Characters 128-129: `V` (hex, add 27 for legacy Ethereum)

```go
// Parse sig into R, S, V for go-ethereum
sigR := sig[0:64]
sigS := sig[64:128]
sigV := sig[128:]
v, _ := strconv.ParseInt(sigV, 16, 64)
v += 27 // for legacy Ethereum
```

---

## Full ERC-20 Transfer via MPC Sign (go-ethereum)

This is the exact pattern from the official SDK demo (`mpcsign_test.go`):

```go
import (
    "context"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
    "github.com/google/uuid"
    "golang.org/x/crypto/sha3"
)

// 1. Build ERC-20 transfer calldata
fnHash := sha3.NewLegacyKeccak256()
fnHash.Write([]byte("transfer(address,uint256)"))
methodId := fnHash.Sum(nil)[:4]
var data []byte
data = append(data, methodId...)
data = append(data, common.LeftPadBytes(common.HexToAddress(toAddress).Bytes(), 32)...)
data = append(data, common.LeftPadBytes(tokenAmount.Bytes(), 32)...)

// 2. Build EIP-1559 raw transaction
tx := types.NewTx(&types.DynamicFeeTx{
    ChainID:   chainID,
    Nonce:     nonce,
    To:        &toAddress,
    Value:     big.NewInt(0),
    Data:      data,
    Gas:       gasLimit,
    GasTipCap: maxPriorityFeePerGas,
    GasFeeCap: maxFeePerGas,
})

// 3. Hash the encoded transaction
signer := types.NewLondonSigner(chainID)
hash := signer.Hash(tx).Hex()

// 4. Submit hash (without "0x") to Safeheron MPC Sign
customerRefId := uuid.NewString()
createReq := api.CreateMpcSignRequest{
    CustomerRefId:    customerRefId,
    SourceAccountKey: accountKey,
    SignAlg:          "Secp256k1",
    DataList: []struct {
        Data string `json:"data,omitempty"`
        Note string `json:"note,omitempty"`
    }{
        {Data: hash[2:]}, // strip "0x"
    },
}
var createResp api.CreateMpcSignResponse
if err := mpcSignApi.CreateMpcSign(createReq, &createResp); err != nil {
    panic(err)
}

// 5. Poll until COMPLETED/CONFIRMED
sig := retrieveSig(mpcSignApi, customerRefId)

// 6. Assemble signed transaction and broadcast
sigByte, _ := hexutil.Decode("0x" + sig)
signedTx, err := tx.WithSignature(signer, sigByte)
if err != nil {
    log.Fatal(err)
}
err = client.SendTransaction(context.Background(), signedTx)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Transaction successful with hash: %s\n", signedTx.Hash().Hex())
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
| `Direct` | string | No | Page direction: `NEXT` (default) / `PREV` |
| `Limit` | int32 | No | Items per page, max 500 |
| `FromId` | string | No | Cursor: omit on first page; pass the `TxKey` of the last item from the previous page |
| `CreateTimeMin` | int64 (ms) | No | Start of creation time range (default: `CreateTimeMax` minus 24 hours) |
| `CreateTimeMax` | int64 (ms) | No | End of creation time range (default: current UTC time) |

```go
mpcSignApi := api.MpcSignApi{Client: sc}

req := api.ListMPCSignTransactionsRequest{
    Limit:         50,
    CreateTimeMin: time.Now().UnixMilli() - 86400000, // last 24h
}
var list []api.MPCSignTransactionsResponse
err := mpcSignApi.ListMPCSignTransactions(req, &list)
if err != nil {
    log.Fatal(err)
}
for _, tx := range list {
    fmt.Println(tx.TxKey, tx.TransactionStatus)
    for _, item := range tx.DataList {
        fmt.Println("  sig:", item.Sig)
    }
}

// Next page: pass TxKey of the last item as FromId
if len(list) > 0 {
    req.FromId = list[len(list)-1].TxKey
    var nextPage []api.MPCSignTransactionsResponse
    err = mpcSignApi.ListMPCSignTransactions(req, &nextPage)
}
```

---

## Best Practices

- The MPC Sign policy is required before use -- contact Safeheron Support to enable.
- **Hash format** -- `Data` must be exactly 32 bytes (64 hex chars) with **no `0x` prefix**. For Ethereum, use `signer.Hash(tx).Hex()` and strip the leading `"0x"` before submitting.
- **Choose the correct `SignAlg`** -- mismatch between algorithm and chain causes signing failure.
- `CustomerRefId` must be unique (max 100 chars). On timeout, **retry with the same `CustomerRefId`** -- Safeheron returns the original request (idempotency). Error code `9001` means the refId already exists.
- **Handle all terminal statuses** when polling -- `FAILED`, `REJECTED`, `CANCELLED` should all abort.
- **Approval policy applies** -- every MPC Sign request goes through the configured approval workflow before signing begins.
