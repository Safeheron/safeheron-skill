# Web3 API Reference

Web3 Wallet Accounts support arbitrary EVM signing: raw message hashes, arbitrary messages (personalSign), structured data (EIP-712), and raw EVM transactions.

> **Web3 API requires a Web3 wallet account.** Using a regular `VAULT_ACCOUNT` key will result in "account not found" errors.

---

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
    "github.com/google/uuid"
)
```

## Create API Instance

```go
web3Api := api.Web3Api{Client: sc}
```

---

## 1. ethSign -- Sign a Raw 32-byte Hash

```go
customerRefId := uuid.New().String()
req := api.EthSignRequest{
    AccountKey:    web3AccountKey,
    CustomerRefId: customerRefId,
    MessageHash: struct {
        ChainId int64    `json:"chainId"`
        Hash    []string `json:"hash"`
    }{
        ChainId: 1, // Ethereum mainnet
        Hash:    []string{"a1b2c3d4e5f6..."}, // 32-byte hex string(s)
    },
}

var resp api.TxKeyResult
if err := web3Api.EthSign(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create web3 eth_sign: %w", err))
}
txKey := resp.TxKey
```

---

## 2. personalSign -- Sign an Arbitrary Message

Adds Ethereum's `\x19Ethereum Signed Message:\n{len}` prefix automatically.

```go
customerRefId := uuid.New().String()
req := api.PersonalSignRequest{
    AccountKey:    web3AccountKey,
    CustomerRefId: customerRefId,
    Message: struct {
        ChainId int64  `json:"chainId"`
        Data    string `json:"data"`
    }{
        ChainId: 1,
        Data:    "Hello, Safeheron!",
    },
}

var resp api.TxKeyResult
if err := web3Api.PersonalSign(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create web3 personal_sign: %w", err))
}
txKey := resp.TxKey
```

---

## 3. ethSignTypedData -- Sign EIP-712 Structured Data

```go
customerRefId := uuid.New().String()
req := api.EthSignTypedDataRequest{
    AccountKey:    web3AccountKey,
    CustomerRefId: customerRefId,
    Message: struct {
        ChainId int64  `json:"chainId"`
        Data    string `json:"data"`
        Version string `json:"version"`
    }{
        ChainId: 1,
        Data:    `{"types":{...}, "domain":{...}, "message":{...}}`,
        Version: "ETH_SIGNTYPEDDATA_V4",
    },
}

var resp api.TxKeyResult
if err := web3Api.EthSignTypedData(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create web3 eth_signTypedData: %w", err))
}
txKey := resp.TxKey
```

**EIP-712 Version Values:**

| Value | Standard |
|-------|----------|
| `ETH_SIGNTYPEDDATA_V1` | Legacy typed signing |
| `ETH_SIGNTYPEDDATA_V3` | EIP-712 draft |
| `ETH_SIGNTYPEDDATA_V4` | EIP-712 (MetaMask current default) |

---

## 4. ethSignTransaction -- Sign a Raw EVM Transaction

Signs (but does NOT broadcast) a raw EVM transaction:

```go
customerRefId := uuid.New().String()
req := api.EthSignTransactionRequest{
    AccountKey:    web3AccountKey,
    CustomerRefId: customerRefId,
    Transaction: struct {
        To                   string `json:"to"`
        Value                string `json:"value"`
        ChainId              int64  `json:"chainId"`
        GasPrice             string `json:"gasPrice,omitempty"`
        GasLimit             int32  `json:"gasLimit"`
        MaxPriorityFeePerGas string `json:"maxPriorityFeePerGas,omitempty"`
        MaxFeePerGas         string `json:"maxFeePerGas,omitempty"`
        Nonce                int64  `json:"nonce"`
        Data                 string `json:"data,omitempty"`
    }{
        To:                   "0xRecipientAddress",
        Value:                "1000000000000000", // in wei
        ChainId:              1,
        GasLimit:             21000,
        MaxFeePerGas:         "30000000000",
        MaxPriorityFeePerGas: "1500000000",
        Nonce:                42,
        Data:                 "",
    },
}

var resp api.TxKeyResult
if err := web3Api.EthSignTransaction(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create web3 eth_signTransaction: %w", err))
}
txKey := resp.TxKey
```

---

## Poll for Web3 Sign Result

All four sign types use the same polling method:

```go
func queryWeb3Sig(web3Api api.Web3Api, customerRefId string) api.Web3SignQueryResponse {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    req := api.Web3SignQueryRequest{
        CustomerRefId: customerRefId,
    }

    var resp api.Web3SignQueryResponse
    for range ticker.C {
        if err := web3Api.QueryWeb3Sig(req, &resp); err != nil {
            panic(fmt.Errorf("failed to query web3 sign result: %w", err))
        }

        switch resp.TransactionStatus {
        case "FAILED", "REJECTED":
            panic("web3 sign task was FAILED or REJECTED")
        case "SIGN_COMPLETED":
            return resp
        default:
            fmt.Println("Waiting for sign completion...")
        }
    }
    panic("timeout waiting for web3 sign result")
}
```

---

## Retrieve Signature from Response

After `SIGN_COMPLETED`:

```go
// For ethSign:
sig := resp.MessageHash.SigList[0].Sig

// For personalSign and ethSignTypedData:
sig := resp.Message.Sig.Sig

// For ethSignTransaction:
signedTx := resp.Transaction.SignedTransaction // hex-encoded signed tx
txHash   := resp.Transaction.TxHash
```

---

## Broadcast Signed Transaction (go-ethereum)

```go
import (
    "context"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// For ethSign: assemble with raw transaction
sigByte, _ := hexutil.Decode("0x" + sig)
signedTx, err := rawTransaction.WithSignature(signer, sigByte)
if err != nil {
    log.Fatal(err)
}
err = client.SendTransaction(context.Background(), signedTx)

// For ethSignTransaction: broadcast signed transaction hex directly
signedTxBytes, _ := hex.DecodeString(signedTransaction[2:])
tx := new(types.Transaction)
tx.UnmarshalBinary(signedTxBytes)
err = client.SendTransaction(context.Background(), tx)
```

---

## Web3 Sign Status Flow

```
SUBMITTED -> SIGNING -> SIGN_COMPLETED
                      |-- FAILED | REJECTED | CANCELLED
```

---

## Best Practices

- **Web3 API requires Web3 wallet**. Regular vault account keys cause "account not found" errors.
- Web3 wallets must have a Web3 signing policy configured -- contact Safeheron Support to enable.
- `CustomerRefId` must be unique (max 100 chars). Duplicate refId returns error code `9001`.
- All sign requests must also be approved via the configured approval policy before signing begins.
