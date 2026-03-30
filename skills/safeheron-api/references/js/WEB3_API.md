# Web3 API Reference

Web3 Wallet Accounts support arbitrary EVM signing: raw message hashes, arbitrary messages (personalSign), structured data (EIP-712), and raw EVM transactions.

> **Web3 API requires a Web3 wallet account.** Using a regular `VAULT_ACCOUNT` key will result in "account not found" errors.

---

## Imports

```typescript
import { Web3Api, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
import crypto from 'crypto';
```

## Create API Instance

```typescript
const web3Api = new Web3Api(config);
```

---

## 1. ethSign -- Sign a Raw 32-byte Hash

Used to sign any arbitrary hash (e.g., Ethereum message hash prefix, custom protocol hash).

```typescript
const resp = await web3Api.createWeb3EthSign({
  customerRefId: crypto.randomUUID(),
  accountKey: web3AccountKey,
  messageHash: {
    chainId: 1,  // Ethereum mainnet (informational)
    hash: ['a1b2c3d4e5f6...'],  // 32-byte hex string(s)
  },
});

const txKey = resp.txKey;
```

---

## 2. personalSign -- Sign an Arbitrary Message

Adds Ethereum's `\x19Ethereum Signed Message:\n{len}` prefix automatically.

```typescript
const resp = await web3Api.createWeb3PersonalSign({
  customerRefId: crypto.randomUUID(),
  accountKey: web3AccountKey,
  message: {
    chainId: 1,
    data: 'Hello, Safeheron!',  // raw message text or hex
  },
});

const txKey = resp.txKey;
```

---

## 3. ethSignTypedData -- Sign EIP-712 Structured Data

Supports EIP-712 structured typed data signing (Metamask `eth_signTypedData`).

```typescript
const resp = await web3Api.createWeb3EthSignTypedData({
  customerRefId: crypto.randomUUID(),
  accountKey: web3AccountKey,
  message: {
    chainId: 1,
    data: JSON.stringify({
      types: { EIP712Domain: [...], Person: [...], Mail: [...] },
      primaryType: 'Mail',
      domain: { name: 'Ether Mail', version: '1', chainId: 1, verifyingContract: '0x...' },
      message: { from: { name: 'Cow', wallet: '0x...' }, to: { name: 'Bob', wallet: '0x...' }, contents: 'Hello!' },
    }),
    version: 'ETH_SIGNTYPEDDATA_V4',  // "ETH_SIGNTYPEDDATA_V1", "ETH_SIGNTYPEDDATA_V3", or "ETH_SIGNTYPEDDATA_V4"
  },
});

const txKey = resp.txKey;
```

**EIP-712 Version Values:**

| Value | Standard |
|-------|----------|
| `ETH_SIGNTYPEDDATA_V1` | Legacy typed signing |
| `ETH_SIGNTYPEDDATA_V3` | EIP-712 draft |
| `ETH_SIGNTYPEDDATA_V4` | EIP-712 (MetaMask current default) |

---

## 4. ethSignTransaction -- Sign a Raw EVM Transaction

Signs (but does NOT broadcast) a raw EVM transaction. Returns the signed transaction hex.

```typescript
const resp = await web3Api.createWeb3EthSignTransaction({
  customerRefId: crypto.randomUUID(),
  accountKey: web3AccountKey,
  transaction: {
    to: '0xContractOrRecipientAddress',
    value: '0',                         // amount in wei (string)
    chainId: 1,
    gasPrice: '30000000000',            // for legacy transactions
    gasLimit: 21000,
    maxPriorityFeePerGas: '1500000000', // for EIP-1559
    maxFeePerGas: '30000000000',        // for EIP-1559
    nonce: 42,
    data: '0x...',                      // contract call data, or '' for plain transfer
  },
});

const txKey = resp.txKey;
```

---

## Poll for Web3 Sign Result

All four sign types use the same polling method:

```typescript
async function pollWeb3Sign(web3Api: Web3Api, txKey: string): Promise<any> {
  for (let i = 0; i < 60; i++) {
    const result = await web3Api.oneWeb3Sign({ txKey });
    const status = result.transactionStatus;

    console.log(`Status: ${status}, Sub: ${result.transactionSubStatus}`);

    if (status === 'SIGN_COMPLETED' || status === 'FAILED' || status === 'REJECTED') {
      return result;
    }

    await new Promise(resolve => setTimeout(resolve, 5000));
  }
  throw new Error('Polling timeout');
}
```

---

## Retrieve Signature from Response

After `SIGN_COMPLETED`:

```typescript
// For ethSign:
const sig = result.messageHash.sigList[0].sig;

// For personalSign and ethSignTypedData:
const sig = result.message.sig.sig;

// For ethSignTransaction:
const signedTx = result.transaction.sig.sig;  // hex-encoded signature
```

---

## Convert Signature and Broadcast (ethSign Example)

```typescript
import { providers, utils } from 'ethers';
import { splitSignature } from '@ethersproject/bytes';

// provider must be an ethers JsonRpcProvider (or InfuraProvider, AlchemyProvider, etc.)
const provider = new providers.JsonRpcProvider(process.env.RPC_URL);

function convertSig(sig: string) {
  const r = sig.substring(0, 64);
  const s = sig.substring(64, 128);
  const v = sig.substring(128);
  return splitSignature({
    r: '0x' + r,
    s: '0x' + s,
    recoveryParam: parseInt(v, 16),
  });
}

// Serialize transaction with signature and broadcast
const signature = convertSig(sig);
const rawTransaction = utils.serializeTransaction(tx, signature);
const response = await provider.sendTransaction(rawTransaction);
console.log(`Transaction hash: ${response.hash}`);
```

---

## Cancel a Web3 Sign Request

```typescript
await web3Api.cancelWeb3Sign({ txKey });
```

---

## List Web3 Sign Requests

```typescript
const list = await web3Api.listWeb3Sign({
  limit: 20,
  // Optional: transactionStatus: ['SIGN_COMPLETED'],
  // Optional: subjectType: 'PERSONAL_SIGN',
});
```

---

## Web3SignResponse Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron sign request key |
| `accountKey` | string | Web3 wallet account key |
| `sourceAddress` | string | Signing address |
| `transactionStatus` | string | `SUBMITTED`, `SIGNING`, `SIGN_COMPLETED`, `FAILED`, `REJECTED`, `CANCELLED` |
| `transactionSubStatus` | string | Detailed sub-status |
| `subjectType` | string | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `customerRefId` | string | Your reference ID |
| `messageHash` | object | For ethSign -- contains `chainId`, `sigList` |
| `message` | object | For personalSign / ethSignTypedData -- contains `chainId`, `data`, `sig` |
| `transaction` | object | For ethSignTransaction -- contains all tx fields + `sig` |

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
- `customerRefId` must be unique (max 100 chars). Duplicate refId returns error code `9001`.
- All sign requests must also be approved via the configured approval policy before signing begins.
