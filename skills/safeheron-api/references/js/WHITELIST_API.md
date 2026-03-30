# Whitelist API Reference

## Imports

```typescript
import { WhitelistApi, SafeheronConfig } from '@safeheron/api-sdk';
import { SafeheronError } from '@safeheron/api-sdk';
```

## Create API Instance

```typescript
const whitelistApi = new WhitelistApi(config);
```

---

## Create a Whitelist Entry

```typescript
const resp = await whitelistApi.createWhitelist({
  whitelistName: "Alice's ETH address",  // max 20 chars
  chainType: 'EVM',                       // see chainType values below
  address: '0xAliceEthAddress...',
  // memo: 'optional memo',              // for TON networks (max 20 chars)
  // hiddenOnUI: false,                  // true = API-only, hidden in Console
});

const whitelistKey = resp.whitelistKey;  // save this -- permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | string | Display name (max 20 chars) |
| `chainType` | Yes | string | Blockchain type (see table below) |
| `address` | Yes | string | Blockchain address to whitelist |
| `memo` | No | string | Memo/tag for TON networks (max 20 chars) |
| `hiddenOnUI` | No | boolean | If true, hidden from Web Console / App |

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

```typescript
const resp = await whitelistApi.oneWhitelist({
  whitelistKey: whitelistKey,
  address: '',
});

console.log('Name:', resp.whitelistName);
console.log('Address:', resp.address);
console.log('Status:', resp.whitelistStatus);
```

---

## List Whitelist Entries

```typescript
const list = await whitelistApi.listWhitelist({
  limit: 20,
});

for (const entry of list) {
  console.log(entry.whitelistKey, entry.whitelistName, entry.whitelistStatus);
}
```

---

## Edit a Whitelist Entry

```typescript
await whitelistApi.editWhitelist({
  whitelistKey: whitelistKey,
  whitelistName: 'Updated name',
  // address: '0xNewAddress',  // optional
});
```

---

## Delete a Whitelist Entry

```typescript
await whitelistApi.deleteWhitelist({
  whitelistKey: whitelistKey,
});
```

---

## Create Whitelist from an Existing Transaction

Automatically creates a whitelist entry using the destination address of a completed transaction:

```typescript
const resp = await whitelistApi.createFromTransactionWhitelist({
  txKey: txKey,
  whitelistName: 'Recipient from TX-001',
});

const whitelistKey = resp.whitelistKey;
```

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `whitelistKey` | string | Unique identifier |
| `whitelistName` | string | Display name |
| `chainType` | string | Blockchain type |
| `address` | string | Whitelisted address |
| `memo` | string | Memo (TON networks) |
| `whitelistStatus` | string | `AUDIT` / `APPROVED` / `REJECTED` |
| `createTime` | number | Unix timestamp (ms) |
| `lastUpdateTime` | number | Unix timestamp (ms) |

### Whitelist Status Values

| Status | Description |
|--------|-------------|
| `AUDIT` | Pending approval (if policy requires human review) |
| `APPROVED` | Active -- can be used as transaction destination |
| `REJECTED` | Denied -- cannot be used |

---

## Using Whitelist as Transaction Destination

```typescript
import { TransactionApi } from '@safeheron/api-sdk';
import crypto from 'crypto';

const transactionApi = new TransactionApi(config);

await transactionApi.createTransactions({
  customerRefId: crypto.randomUUID(),
  destinationAccountType: 'WHITELISTING_ACCOUNT',
  destinationAccountKey: whitelistKey,  // pass whitelistKey here
  coinKey: 'ETHEREUM_ETH',
  txAmount: '0.05',
  sourceAccountKey: accountKey,
  sourceAccountType: 'VAULT_ACCOUNT',
  txFeeLevel: 'MIDDLE',
});
```

---

## Best Practices

- Validate address with `coinApi.checkCoinAddress()` before creating a whitelist entry.
- Use descriptive `whitelistName` and set `chainType` carefully -- wrong chainType causes address validation failure.
- In regulated environments, configure Safeheron Console policy to require human approval before whitelist additions take effect (`whitelistStatus = "AUDIT"` until approved).
- Subscribe to Whitelist webhook events (`WHITELIST_ADDED`, `WHITELIST_UPDATED`, `WHITELIST_REMOVED`) to keep your local state in sync.
