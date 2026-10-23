# Whitelist API Reference

## Imports

```typescript
import { WhitelistApi, WhitelistResponse, SafeheronConfig } from '@safeheron/api-sdk';
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
  whitelistName: "Alice's ETH address",  // unique, max 30 chars
  chainType: 'EVM',                       // see chainType values below
  address: '0xAliceEthAddress...',
  // memo: '123456',                     // TON / XRP Ledger only (max 20 chars; XRPL: numeric string 0–4294967295)
  // hiddenOnUI: false,                  // true = API-only, hidden in Console
  // note: 'Alice cold wallet',          // optional note
});

const whitelistKey = resp.whitelistKey;  // save this -- permanent identifier
```

### CreateWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | string | Whitelist unique name, 30 characters max |
| `chainType` | Yes | string | Blockchain type (see table below) |
| `address` | Yes | string | Public blockchain address and the address format needs to meet the requirements of the chain |
| `memo` | No | string | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the public blockchain address, also known as a comment or tag. Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `hiddenOnUI` | No | boolean | Visibility status in Safeheron App and Web Console. `false` (default): visible; `true`: invisible — the whitelist can only be managed and used through the API |
| `note` | No | string | Note |

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
| `Litecoin` | Litecoin mainnet addresses that can receive Litecoin |
| `Dogecoin` | Dogecoin mainnet addresses that can receive Dogecoin |
| `Liquid` | Liquid mainnet addresses that can receive Liquid |
| `XRP Ledger` | XRP Ledger mainnet addresses, which only supports receiving native XRP Ledger tokens and does not support other tokens yet |

---

## Retrieve a Single Whitelist Entry

```typescript
// Look up by whitelistKey, or by address (if both are provided, whitelistKey takes precedence).
// SDK v1.0.32 declares both fields as required in TypeScript — pass '' for the one you don't use.
const resp = await whitelistApi.oneWhitelist({
  whitelistKey: whitelistKey,
  address: '',
});

console.log('Name:', resp.whitelistName);
console.log('Address:', resp.address);
console.log('Status:', resp.whitelistStatus);
console.log('Note:', resp.note);
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistKey` | No* | string | Whitelist unique identifier. It is required if address is not provided. If both are provided, the whitelistKey takes precedence |
| `address` | No* | string | Whitelist address. It is required if whitelistKey is not provided and please make sure the provided address is correct |

---

## List Whitelist Entries

> ⚠️ **Time window:** if `createTimeMin` is not provided it defaults to `createTimeMax` minus **24 hours** (and `createTimeMax` defaults to now). A call without time bounds therefore only returns whitelists **created in the last 24 hours** — not "all". To list everything, pass an early `createTimeMin` and page with `fromId`.

```typescript
const PAGE_SIZE = 500;  // API maximum

// Lists every whitelist entry: createTimeMin set to the epoch start, cursor paging with direct=NEXT + fromId
async function listAllWhitelists(): Promise<WhitelistResponse[]> {
  const all: WhitelistResponse[] = [];
  let fromId: string | undefined;
  while (true) {
    const page = await whitelistApi.listWhitelist({
      // chainType: 'EVM',  // optional filter — see note below; omit it to list whitelists of all chains
      createTimeMin: 1,     // without it only the last 24 hours are returned
      direct: 'NEXT',
      limit: PAGE_SIZE,
      fromId,               // undefined on the first page, then the whitelistKey of the last record
    });
    all.push(...page);
    if (page.length < PAGE_SIZE) {
      return all;
    }
    fromId = page[page.length - 1].whitelistKey;
  }
}

for (const entry of await listAllWhitelists()) {
  console.log(entry.whitelistKey, entry.whitelistName, entry.whitelistStatus, entry.note);
}
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `direct` | No | string | Query page direction, `NEXT` by default |
| `limit` | No | number | The number of items to retrieve at a time, default max value is 500 |
| `fromId` | No | string | whitelistKey of the first record. If the first page has no value, provide the whitelistKey of the last record from the previous result |
| `chainType` | Yes (per API docs) | string | Blockchain type (see chainType table above). Published docs mark chainType as required, but the service treats it as an optional filter — passing it returns only that chain; omit it to list all chains |
| `whitelistStatus` | No | string | `AUDIT` / `APPROVED` / `REJECTED` |
| `hiddenOnUI` | No | boolean | Filter by hiddenOnUI (default: all). Documented by the API, but not declared in SDK v1.0.32's `ListWhitelistRequest` |
| `createTimeMin` | No | number | Start time for creating a whitelist in UNIX timestamp (ms) (default: createTimeMax minus 24 hours) |
| `createTimeMax` | No | number | End time for creating a whitelist in UNIX timestamp (ms) (default: current UTC time) |

---

## Edit a Whitelist Entry

```typescript
// SDK v1.0.32 declares `force` and `note` as required in TypeScript — always pass them
const currentNote = 'Alice cold wallet';  // the note currently stored (e.g. from oneWhitelist) — re-send it to keep it
await whitelistApi.editWhitelist({
  whitelistKey: whitelistKey,
  whitelistName: 'Updated name',
  address: '0xNewAddress',
  force: false,   // true = also apply the change to approval policies that reference this whitelist
  note: currentNote,  // note (like memo) overwrites the stored value — pass the current note to keep it; '' may clear it
});
```

### EditWhitelistRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistKey` | Yes | string | Unique identifier of the whitelist entry to edit |
| `whitelistName` | Yes | string | Whitelist unique name, 30 characters max |
| `address` | Yes | string | Public blockchain address and the address format needs to meet the requirements of the chain |
| `memo` | No | string | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the public blockchain address, also known as a comment or tag. Only valid for chainType `TON` (TON mainnet) and `XRP Ledger` (XRP Ledger mainnet) |
| `force` | No (required in SDK v1.0.32 TS types) | boolean | When the whitelist is involved in a transaction approval policy, modifications will result in the new whitelist being directly applied to the approval policy. False by default, meaning that when involved in a transaction approval policy, it will not be modified |
| `note` | No (required in SDK v1.0.32 TS types) | string | Note |

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
  whitelistName: 'Recipient from TX-001',      // unique, max 30 chars
  destinationAddress: '0xRecipientAddress...', // required: the destination address in the tx record (case-sensitive)
  // memo: '123456',     // TON / XRP Ledger: must match the memo in the transaction record, if one was set
  // hiddenOnUI: false,
  // note: 'Recipient of TX-001',
});

const whitelistKey = resp.whitelistKey;
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `whitelistName` | Yes | string | Whitelist unique name, 30 characters max |
| `txKey` | Yes | string | Transaction key |
| `destinationAddress` | Yes | string | The destination address in the transaction record; case-sensitive |
| `memo` | No | string | The memo (up to 20 characters; for XRPL, the memo must be a numeric string in the range 0–4294967295) for the destination address. For TON and XRP Ledger, if a destination address memo was set initially, a memo matching the one in the transaction record must be provided |
| `hiddenOnUI` | No | boolean | Visibility status in Safeheron App and Web Console (`false` by default) |
| `note` | No | string | Note |

---

## WhitelistResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `whitelistKey` | string | Unique identifier |
| `whitelistName` | string | Display name |
| `chainType` | string | Blockchain type |
| `address` | string | Whitelisted address |
| `memo` | string | Memo of the address when creating a whitelist (TON / XRP Ledger) |
| `whitelistStatus` | string | `AUDIT` / `APPROVED` / `REJECTED` |
| `hiddenOnUI` | boolean | Visibility in App / Web Console (`true`: invisible). Returned by the API, but not declared in SDK v1.0.32's `WhitelistResponse` type |
| `note` | string | Note |
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
