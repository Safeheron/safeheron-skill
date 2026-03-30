# MPC Sign API Reference

## Imports

```typescript
import { MPCSignApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
import crypto from 'crypto';
```

## Create API Instance

```typescript
const mpcSignApi = new MPCSignApi(config);
```

---

## Create an MPC Sign Transaction

```typescript
const resp = await mpcSignApi.createMPCSignTransactions({
  customerRefId: crypto.randomUUID(),
  sourceAccountKey: accountKey,
  signAlg: 'Secp256k1',  // "Secp256k1" or "Ed25519"
  dataList: [{
    // 32-byte hex string without '0x' prefix
    data: hashHexWithoutPrefix,
  }],
});

const txKey = resp.txKey;
```

---

## Poll for MPC Sign Result

```typescript
async function pollMpcSign(mpcSignApi: MPCSignApi, txKey: string): Promise<string> {
  for (let i = 0; i < 100; i++) {
    const resp = await mpcSignApi.oneMPCSignTransactions({ txKey });

    console.log(`Status: ${resp.transactionStatus}, Sub: ${resp.transactionSubStatus}`);

    if (resp.transactionStatus === 'FAILED' || resp.transactionStatus === 'REJECTED' || resp.transactionStatus === 'CANCELLED') {
      throw new Error('MPC sign failed/rejected/cancelled');
    }

    if (resp.transactionStatus === 'COMPLETED' && resp.transactionSubStatus === 'CONFIRMED') {
      return resp.dataList[0].sig;
      // sig = R (64 hex chars) + S (64 hex chars) + V (2 hex chars)
    }

    await new Promise(resolve => setTimeout(resolve, 5000));
  }

  throw new Error("Polling timeout -- can't get sig.");
}
```

---

## MPCSignTransactionsResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Transaction key |
| `transactionStatus` | string | Transaction status |
| `transactionSubStatus` | string | Transaction sub-status |
| `createTime` | number | Creation time, UNIX timestamp (ms) |
| `sourceAccountKey` | string | Source account key |
| `customerRefId` | string | Merchant unique business ID |
| `signAlg` | string | Signature algorithm |
| `dataList` | Array | List of signed data items |

**dataList Item Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `note` | string | Transaction note (max 180 chars) |
| `data` | string | Transaction data that was signed |
| `sig` | string | Signature: 32 bytes R + 32 bytes S + 1 byte V (hex concatenated) |

---

## Signature Format

The `sig` field is a concatenated hex string:
- Characters 0--63: `R`
- Characters 64--127: `S`
- Characters 128--129: `V` (hex, add 27 for legacy Ethereum)

```typescript
import { splitSignature } from '@ethersproject/bytes';

function convertSig(mpcSig: string) {
  const r = mpcSig.substring(0, 64);
  const s = mpcSig.substring(64, 128);
  const v = mpcSig.substring(128);
  return splitSignature({
    r: '0x' + r,
    s: '0x' + s,
    recoveryParam: parseInt(v, 16),
  });
}
```

---

## Full ERC-20 Transfer via MPC Sign (ethers.js)

This is the pattern from the official SDK demo:

```typescript
import { providers, utils, Contract, UnsignedTransaction, BigNumber } from 'ethers';
import { splitSignature } from '@ethersproject/bytes';
import { MPCSignApi } from '@safeheron/api-sdk';
import crypto from 'crypto';

const provider = new providers.InfuraProvider('sepolia');
const ERC20_CONTRACT_ADDRESS = '0x...';
const ERC20_ABI = [/* standard ERC20 ABI */];
const ERC20 = new Contract(ERC20_CONTRACT_ADDRESS, ERC20_ABI, provider);

async function transferERC20(
  mpcSignApi: MPCSignApi,
  accountKey: string,
  fromAddress: string,
  toAddress: string,
) {
  // 1. Encode ERC-20 transfer function
  const decimals = await ERC20.decimals();
  const amount = utils.parseUnits('1', decimals);
  const data = ERC20.interface.encodeFunctionData('transfer', [toAddress, amount]);

  // 2. Get chain info
  const chainId = (await provider.getNetwork()).chainId;
  const nonce = await provider.getTransactionCount(fromAddress);
  const gasLimit = await provider.estimateGas({
    from: fromAddress,
    to: ERC20_CONTRACT_ADDRESS,
    value: 0,
    data,
  });

  // 3. Estimate fees
  const maxPriorityFeePerGas = utils.parseUnits('2', 'gwei');
  const latestBlock = await provider.getBlock('latest');
  const suggestBaseFee = latestBlock.baseFeePerGas?.mul(2);
  const maxFeePerGas = suggestBaseFee?.add(maxPriorityFeePerGas);

  // 4. Build unsigned transaction
  const tx: UnsignedTransaction = {
    to: ERC20_CONTRACT_ADDRESS,
    value: 0,
    data,
    nonce,
    chainId,
    type: 2,
    maxPriorityFeePerGas,
    maxFeePerGas: maxFeePerGas!,
    gasLimit,
  };

  // 5. Hash the encoded transaction
  const serialized = utils.serializeTransaction(tx);
  const hash = utils.keccak256(serialized);

  // 6. Submit hash (without "0x") to Safeheron MPC Sign
  const createResp = await mpcSignApi.createMPCSignTransactions({
    customerRefId: crypto.randomUUID(),
    sourceAccountKey: accountKey,
    signAlg: 'Secp256k1',
    dataList: [{ data: hash.substring(2) }],  // strip "0x"
  });

  console.log(`MPC sign created, txKey: ${createResp.txKey}`);

  // 7. Poll until COMPLETED/CONFIRMED
  const mpcSig = await pollMpcSign(mpcSignApi, createResp.txKey);
  console.log(`Got MPC signature: ${mpcSig}`);

  // 8. Convert signature and broadcast
  const signature = convertSig(mpcSig);
  const rawTransaction = utils.serializeTransaction(tx, signature);
  const response = await provider.sendTransaction(rawTransaction);
  console.log(`Transaction hash: ${response.hash}`);
}
```

---

## Sign a Message (Ethereum / TRON)

```typescript
import { keccak256, toUtf8Bytes, concat } from 'ethers/lib/utils';

// Ethereum personal sign
async function ethSignMessage(mpcSignApi: MPCSignApi, accountKey: string) {
  const message = 'here is message to be signed';
  const messagePrefix = '\x19Ethereum Signed Message:\n';
  const messageByte = toUtf8Bytes(message);
  const messageHash = keccak256(concat([
    toUtf8Bytes(messagePrefix),
    toUtf8Bytes(String(messageByte.length)),
    messageByte,
  ]));

  const resp = await mpcSignApi.createMPCSignTransactions({
    customerRefId: crypto.randomUUID(),
    sourceAccountKey: accountKey,
    signAlg: 'Secp256k1',
    dataList: [{ data: messageHash.substring(2) }],
  });

  const sig = await pollMpcSign(mpcSignApi, resp.txKey);
  console.log(`Signature: 0x${sig}`);
}

// TRON sign (uses '\x19TRON Signed Message:\n' prefix)
async function tronSignMessage(mpcSignApi: MPCSignApi, accountKey: string) {
  const message = 'here is message to be signed';
  const messagePrefix = '\x19TRON Signed Message:\n';
  const messageByte = toUtf8Bytes(message);
  const messageHash = keccak256(concat([
    toUtf8Bytes(messagePrefix),
    toUtf8Bytes(String(messageByte.length)),
    messageByte,
  ]));

  const resp = await mpcSignApi.createMPCSignTransactions({
    customerRefId: crypto.randomUUID(),
    sourceAccountKey: accountKey,
    signAlg: 'Secp256k1',
    dataList: [{ data: messageHash.substring(2) }],
  });

  const sig = await pollMpcSign(mpcSignApi, resp.txKey);
  console.log(`Signature: 0x${sig}`);
}
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
| `direct` | string | No | Page direction: `NEXT` (default) / `PREV` |
| `limit` | number | No | Items per page, max 500 |
| `fromId` | string | No | Cursor: omit on first page; pass the `txKey` of the last item from the previous page |
| `createTimeMin` | number (ms) | No | Start of creation time range (default: `createTimeMax` minus 24 hours) |
| `createTimeMax` | number (ms) | No | End of creation time range (default: current UTC time) |

```typescript
const list = await mpcSignApi.listMPCSignTransactions({
  limit: 50,
  createTimeMin: Date.now() - 86400000, // last 24h
});

for (const tx of list) {
  console.log(tx.txKey, tx.transactionStatus);
  for (const item of tx.dataList) {
    console.log('  sig:', item.sig);
  }
}

// Next page: pass txKey of the last item as fromId
if (list.length > 0) {
  const nextPage = await mpcSignApi.listMPCSignTransactions({
    limit: 50,
    fromId: list[list.length - 1].txKey,
  });
}
```

---

## Best Practices

- The MPCSign policy is also required before use -- contact Safeheron Support to enable.
- **Hash format** -- `data` must be exactly 32 bytes (64 hex chars) with **no `0x` prefix**.
- **Choose the correct `signAlg`** -- mismatch between algorithm and chain causes signing failure.
- `customerRefId` must be unique (max 100 chars). On timeout, **retry with the same `customerRefId`** -- Safeheron returns the original request (idempotency). Error code `9001` means the refId already exists.
- **Handle all terminal statuses** when polling -- `FAILED`, `REJECTED`, `CANCELLED` should all throw/abort.
- **Approval policy applies** -- every MPC Sign request goes through the configured approval workflow before signing begins.
