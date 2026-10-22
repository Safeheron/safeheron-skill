# Transaction API Reference

## Imports

```java
import com.safeheron.client.api.TransactionApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
TransactionApiService transactionApi = ServiceCreator.create(TransactionApiService.class, safeheronConfig);
```

---

## Create a Transaction

### V2 (standard — returns txKey only)

```java
CreateTransactionRequest req = new CreateTransactionRequest();
req.setCustomerRefId(UUID.randomUUID().toString()); // your unique ID
req.setCoinKey("ETHEREUM_ETH");
req.setTxAmount("0.01");                // string, never float
req.setSourceAccountKey(accountKey);
req.setSourceAccountType("VAULT_ACCOUNT");
req.setDestinationAccountType("ONE_TIME_ADDRESS");
req.setDestinationAddress("0xRecipientAddress");
req.setTxFeeLevel("MIDDLE");           // "LOW" | "MIDDLE" | "HIGH"
// Optional: req.setNote("Payment for invoice #123");
// Optional: req.setCustomerExt1("order-id-456");

TxKeyResult resp = ServiceExecutor.execute(transactionApi.createTransactions(req));
String txKey = resp.getTxKey();  // save this — Safeheron transaction identifier
```

### V3 (extended — returns txKey + idempotency info)

```java
CreateTransactionV3Response resp = ServiceExecutor.execute(
        transactionApi.createTransactionsV3(req));  // same request class
String txKey = resp.getTxKey();
```

**CreateTransactionV3Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your unique business ID echoed back |
| `idempotentRequest` | Boolean | `true` if a duplicate `customerRefId` was detected — the original `txKey` is returned instead of creating a new transaction |

### CreateTransactionRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `customerRefId` | Yes | String | Your unique ID for idempotency (max 100 chars) |
| `coinKey` | Yes | String | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `txAmount` | Yes | String | Amount as string — never use float/double |
| `sourceAccountKey` | Yes | String | Sender wallet key |
| `sourceAccountType` | Yes | String | `VAULT_ACCOUNT` |
| `destinationAccountType` | Yes | String | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `destinationAddress` | Cond. | String | Required when `destinationAccountType=ONE_TIME_ADDRESS` |
| `destinationAccountKey` | Cond. | String | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `txFeeLevel` | No | String | `LOW` / `MIDDLE` / `HIGH` — Choose between transaction fees. If the transaction fee rate is preset, it will take priority |
| `feeRateDto` | No | Object | Transaction fee rate, either `TxFeeLevel` or `FeeRateDto`. `feeRate` unit: fee per byte for UTXO, gas price for EVM chains, fee limit for TRON (optional), gas price for SUI and drops for XRPL (the XRPL `drops` unit is documented for Create a Transaction **V3**) |
| `maxTxFeeRate` | No | String | Max acceptable fee rate |
| `note` | No | String | Transaction note (max 180 chars) |
| `customerExt1` | No | String | Custom field 1 (max 255 chars) |
| `customerExt2` | No | String | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | Boolean | If true, fee is deducted from `txAmount` |
| `memo` | No | String | Memo/tag of the destination address — only for **TON** and **XRP Ledger** (max 100 chars; for XRPL it must be a numeric string in 0–4294967295) |
| `failOnAml` | No | Boolean | Default true — block high-risk addresses via AML |
| `failOnContract` | No | Boolean | Block contract addresses as destination |
| `nonce` | No | Long | Custom nonce for EVM chains |
| `sequenceNumber` | No | Long | Customizable sequence number on Aptos and XRP Ledger, similar to the nonce in the EVM |
| `isRbf` | No | Boolean | Enable Replace-by-Fee for Bitcoin |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` — providing only `gasLimit` will cause the transaction to fail.

---

## Retrieve a Single Transaction

```java
OneTransactionsRequest req = new OneTransactionsRequest();
req.setTxKey(txKey);
// OR: req.setCustomerRefId(customerRefId);

OneTransactionsResponse resp = ServiceExecutor.execute(transactionApi.oneTransactions(req));
System.out.println("Status: " + resp.getTransactionStatus());
System.out.println("Sub-status: " + resp.getTransactionSubStatus());
System.out.println("TxHash: " + resp.getTxHash());
```

**OneTransactionsResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `txHash` | String | On-chain transaction hash |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `txFee` | String | Actual fee paid |
| `feeCoinKey` | String | Fee coin |
| `transactionStatus` | String | See status table below |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Sender wallet key |
| `sourceAddress` | String | Sender address |
| `destinationAddress` | String | Recipient address |
| `createTime` | Long | Unix timestamp (ms) |
| `completedTime` | Long | Unix timestamp (ms) of completion |
| `customerRefId` | String | Your reference ID |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | Whether the source address contains AML address(es): `YES` / `NO` |
| `amlList` | List\<Aml\> | AML assessment results |
| `blockHeight` | Long | Block height at confirmation |
| `nonce` | String | Transaction nonce |
| `speedUpHistory` | List | Speed-up transaction history |
| `replacedTxKey` | String | Original txKey if this is a speed-up |

---

## List Transactions

### V2 (Recommended — cursor-based)

```java
ListTransactionsV2Request req = new ListTransactionsV2Request();
req.setLimit(20L);
req.setTransactionStatus("COMPLETED");  // optional filter
req.setCoinKey("ETHEREUM_ETH");       // optional filter
// createTimeMin defaults to createTimeMax − 24h (createTimeMax defaults to now):
// without it you only get transactions created in the last 24 hours
// req.setCreateTimeMin(startMs);
// req.setCreateTimeMax(endMs);

List<TransactionsResponse> txList = ServiceExecutor.execute(transactionApi.listTransactionsV2(req));
for (TransactionsResponse tx : txList) {
    System.out.println(tx.getTxKey() + " " + tx.getTransactionStatus());
}
```

**ListTransactionsV2Request Fields** (extends `LimitSearch`):

Inherited from `LimitSearch`:

| Field | Type | Description |
|-------|------|-------------|
| `direct` | String | Page direction: `NEXT` (default) |
| `limit` | Long | Items per page, max 500 |
| `fromId` | String | txKey of last record from previous page; omit for first page |

Own fields:

| Field | Type | Description |
|-------|------|-------------|
| `sourceAccountKey` | String | Source account key |
| `sourceAccountType` | String | Source account type |
| `destinationAccountKey` | String | Destination account key |
| `destinationAccountType` | String | Destination account type |
| `accountKey` | String | Query all txns under this wallet (VAULT_ACCOUNT only; overrides source/dest filters) |
| `createTimeMin` | Long | Start time, UNIX timestamp (ms). Defaults to `createTimeMax` minus 24 hours |
| `createTimeMax` | Long | End time, UNIX timestamp (ms). Defaults to the current time |
| `txAmountMin` | String | Min transaction amount |
| `txAmountMax` | String | Max transaction amount |
| `coinKey` | String | Coin key, multiple separated by commas |
| `feeCoinKey` | String | Fee coin key, multiple separated by commas |
| `transactionStatus` | String | Transaction status |
| `transactionSubStatus` | String | Transaction sub-status |
| `completedTimeMin` | Long | Min completion time, UNIX timestamp (ms) |
| `completedTimeMax` | Long | Max completion time, UNIX timestamp (ms) |
| `customerRefId` | String | Merchant unique business ID |
| `realDestinationAccountType` | String | Type of actual destination account |
| `hideSmallAmountUsd` | String | Filter out transactions below this USD amount |
| `transactionDirection` | String | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER`; default: all |

### V1 (legacy — page-based)

```java
ListTransactionsV1Request req = new ListTransactionsV1Request();
req.setPageSize(10L);
req.setPageNumber(1L);

PageResult<TransactionsResponse> result = ServiceExecutor.execute(
        transactionApi.listTransactionsV1(req));
List<TransactionsResponse> txList = result.getContent();
```

---

## Estimate Transaction Fee

```java
TransactionsFeeRateRequest req = new TransactionsFeeRateRequest();
req.setCoinKey("ETHEREUM_ETH");
req.setValue("0.01");
req.setSourceAccountKey(accountKey);
req.setDestinationAddress("0xRecipient");

// For TRON: sourceAddress is required (determines staking/energy)
// req.setSourceAddress("TsenderAddress");

TransactionsFeeRateResponse resp = ServiceExecutor.execute(
        transactionApi.transactionFeeRate(req));
System.out.println("Low:    " + resp.getLowFeeRate().getFeeRate() + " (fee: " + resp.getLowFeeRate().getFee() + ")");
System.out.println("Middle: " + resp.getMiddleFeeRate().getFeeRate() + " (fee: " + resp.getMiddleFeeRate().getFee() + ")");
System.out.println("High:   " + resp.getHighFeeRate().getFeeRate() + " (fee: " + resp.getHighFeeRate().getFee() + ")");
```

**FeeRate Object Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `feeRate` | String | Fee rate: UTXO fee per byte, EVM gas price, TRON fee limit, SUI gas price and XRPL drops |
| `fee` | String | Estimated total fee amount |
| `gasLimit` | String | Gas limit (EVM chains only) |

**TransactionsFeeRateRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | String | Coin key |
| `txHash` | String | Original tx hash (for speed-up fee estimation) |
| `sourceAccountKey` | String | Source account key (required for UTXO coins) |
| `sourceAddress` | String | Source address (required for TRON; for EVM returns on-chain gas limit) |
| `destinationAddress` | String | Destination address (optional for TRON/FIL; for EVM returns on-chain gas limit) |
| `destinationAddressList` | `List<DestinationAddress>` | Destination address list |
| `value` | String | Transfer amount (improves accuracy for EVM, UTXO, SUI) |

> The API also accepts `memo` (TON / XRP Ledger; for XRPL a numeric string in 0–4294967295) for fee estimation, but `TransactionsFeeRateRequest` in SDK 1.0.15 does not have this field yet.

> **BTC fee** = feeRate × bytesSize. **ETH fee** = feeRate × gasLimit.
> **TRON fee**: `sourceAddress` must be provided to check on-chain staking/energy.

---

## Cancel a Transaction

```java
CancelTransactionRequest req = new CancelTransactionRequest();
req.setTxKey(txKey);
// req.setTxType("TRANSACTION");  // default, can be omitted

ServiceExecutor.execute(transactionApi.cancelTransactions(req));
```

**CancelTransactionRequest Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Transaction key |
| `txType` | String | Transaction type, `TRANSACTION` by default |

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

Replaces a stuck transaction with a higher fee version:

```java
RecreateTransactionRequest req = new RecreateTransactionRequest();
req.setTxKey(originalTxKey);     // required
req.setTxHash(originalTxHash);   // required — on-chain hash of the original transaction
req.setCoinKey(coinKey);         // required
req.setTxFeeLevel("HIGH");       // use higher fee (or set feeRateDto)

TxKeyResult resp = ServiceExecutor.execute(transactionApi.recreateTransactions(req));
String newTxKey = resp.getTxKey();
```

**RecreateTransactionRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKey` | Yes | String | Original transaction key to speed up |
| `txHash` | Yes | String | Original transaction hash |
| `coinKey` | Yes | String | Coin key |
| `txFeeLevel` | No | String | Fee level: `LOW` / `MIDDLE` / `HIGH` |
| `feeRateDto` | No | FeeRateDto | Custom fee rate (either `txFeeLevel` or `feeRateDto`) |

> Provide at least one of `txFeeLevel` / `feeRateDto` — the service rejects a speed-up request with neither (service-side validation).

> The original transaction and speed-up are independent. The speed-up will invalidate the original (original → FAILED). The speed-up transaction can be found via `speedUpHistory` in the original transaction, or `replacedTxKey` in the new one.

---

## Get Transaction Approval Details

```java
ApprovalDetailTransactionsRequest req = new ApprovalDetailTransactionsRequest();
req.setTxKeyList(Arrays.asList(txKey));

ApprovalDetailTransactionsResponse resp = ServiceExecutor.execute(
        transactionApi.approvalDetailTransactions(req));
```

---

## Transaction Status Reference

| Status | Description |
|--------|-------------|
| `SUBMITTED` | Received by Safeheron |
| `SIGNING` | MPC signing in progress — entered after approval by team members or API Co-Signer |
| `BROADCASTING` | Being broadcast to the blockchain network. For incoming transactions, only Bitcoin supports this state (zero confirmation) |
| `CONFIRMING` | On-chain, awaiting block confirmations |
| `COMPLETED` | Confirmed on-chain — assets successfully transferred |
| `CANCELLED` | Final state — transaction cancelled by team member or system |
| `FAILED` | Transaction failed |
| `REJECTED` | Rejected by a team member's approval, or rejected due to a Connect recipient's approval |

### Status Flow

```
SUBMITTED
    → SIGNING
    → BROADCASTING
    → CONFIRMING
    → COMPLETED
    └─ FAILED

    At SUBMITTED or SIGNING:
    → REJECTED (denied by approver)
    → CANCELLED (cancelled by team member)
```

### Transaction Substatus (`transactionSubStatus`)

`transactionSubStatus` gives the detailed sub-state / failure reason of a transaction (e.g. `CONFIRMED`, `PENDING_BLOCKCHAIN_CONFIRMATIONS`, or why a `FAILED` transaction failed).

| Substatus | Description |
|-----------|-------------|
| `CONFIRMED` | Confirmed |
| `UN_KNOW_ERROR` | Unexpected error |
| `REVIEW_REJECTED` | Rejected |
| `TRADING_STRATEGY_BLOCKING` | Transaction policy has been blocked |
| `MATCHING_RULE_NO_AUDITOR` | Matched with the policy, but the editor does not possess the approval role |
| `TX_INFO_ERROR` | Transaction failed |
| `NONCE_TOO_LOW` | Nonce too low |
| `INTRINSIC_GAS_TOO_LOW` | Gas limit too low, cannot perform transaction |
| `INSUFFICIENT_FUNDS` | Insufficient amount for withdrawal |
| `AMOUNT_TOO_SMALL` | The withdrawal amount is below the minimum limit |
| `TRANSACTION_UNDERPRICED` | Transaction fee too low, transaction is rejected by node |
| `MISSING_INPUTS` | Invalid input |
| `INVALID_SIGNATURE` | Invalid transaction signature |
| `INTERNAL_ERROR` | An internal error occurred while processing the transaction |
| `TIMEOUT` | The transaction request timed out |
| `PENDING_BLOCKCHAIN_CONFIRMATIONS` | Pending blockchain confirmation |
| `TAPOS_ERROR` | Tapos check error |
| `NODE_SERVER_BUSY` | Node service is busy |
| `DUP_TRANSACTION_ERROR` | The transaction is on-chain, and a duplicate submission has been made |
| `TRANSACTION_EXPIRATION_ERROR` | Transaction expired |
| `TRANSACTION_EXECUTION_FAILED` | Transaction execution failed |
| `BANDWITH_ERROR` | For Tron transactions: Insufficient bandwidth/energy or TRX. You can either increase the transaction fee or have other accounts delegate resources to this account before initiating the transaction |
| `ACCOUNT_DOES_NOT_EXIST` | Account doesn't exist |
| `NO_MATCHING_RULE_FOUND` | Transaction failed, configure transaction policy first |
| `WHITELIST_ADDRESS_STATUS_ILLEGAL` | The whitelisted address is not approved |
| `AML_SCREENING_REJECTED` | Rejected by AML check |
| `CONTRACT_VALIDATE_ERROR` | Transaction verification error |
| `THE_DEAL_WAS_ABANDONED` | Transaction abandoned |
| `MANUAL_TX_FAIL` | Manual processing |
| `CANCELLED_BY_USER` | User cancelled transaction |
| `RISK_CHECK_ILLEGAL` | Exceptions detected by risk control |
| `WALLET_NOT_ALLOWED` | The wallet doesn't support current operation |
| `USER_NO_PERMISSION` | User doesn't have permission |
| `EXCHANGE_EMAIL_CONFIRMING` | Pending email confirmation |
| `EXCHANGE_UN_KNOW_ERROR` | Unexpected error |
| `EXCHANGE_INVALID_INFO` | Invalid API key, IP, or operational permission |
| `EXCHANGE_AMOUNT_OVER_LIMIT` | The transfer amount exceeds the maximum |
| `EXCHANGE_BALANCE_LACK` | Insufficient balance |
| `RESET_AUTO_CANCEL` | Automatic cancellation |
| `REVIEW_OTHERS_REJECTED` | Rejected by the counterparty |
| `TON_BOUNCE_TRANSACTION` | The transaction is bounced |
| `TON_SEQUENCE_INVALID` | Invalid sequence |
| `UTXO_DUST` | Dust Transaction |
| `MPC_TIMEOUT` | The MPC signature timed out |
| `SOL_ERROR_PROCESSING_INSTRUCTION` | Transaction execution failed |
| `DEST_ADDRESS_TOKEN_BLACKLISTED` | The destination address has been frozen by the token issuer |
| `SOURCE_ADDRESS_TOKEN_BLACKLISTED` | The source address has been frozen by the token issuer |

### Transaction Subtype (`transactionType`)

| `transactionType` | Description |
|-------------------|-------------|
| `NORMAL` | Normal transaction |
| `UTXO_COLLECTION` | UTXO-sweeping transaction |
| `ABNORMAL` | Transfers to external destinations (not in the Safeheron platform) or transactions signed by the original private key recovered by key shards |
| `AUTO_SWEEP` | Auto-sweeping transaction |
| `AUTO_FUEL` | Auto-fueling transaction |
| `UTXO_BATCH_TRANSFER` | UTXO multi-address transfer |
| `BOUNCED_TRANSFER` | TON's bounced transaction |
| `GAS_SERVICE_DEPOSIT` | Gas Service Deposit |
| `GAS_SERVICE_FUEL` | Gas Service Fuel |

### Transaction Direction (source/destination type)

| Scenario | `destinationAccountType` | `sourceAccountType` |
|----------|------------------------|-------------------|
| Withdrawal (outflow) | `ONE_TIME_ADDRESS` | `VAULT_ACCOUNT` |
| Internal transfer | `VAULT_ACCOUNT` | `VAULT_ACCOUNT` |
| Deposit (inflow) | — | `UNKNOWN` |

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create V2 | `CreateTransactionRequest` | `TxKeyResult` |
| Create V3 | `CreateTransactionRequest` | `CreateTransactionV3Response` |
| Get one | `OneTransactionsRequest` | `OneTransactionsResponse` |
| List V2 | `ListTransactionsV2Request` | `List<TransactionsResponse>` |
| List V1 | `ListTransactionsV1Request` | `PageResult<TransactionsResponse>` |
| Estimate fee | `TransactionsFeeRateRequest` | `TransactionsFeeRateResponse` |
| Cancel | `CancelTransactionRequest` | `ResultResponse` |
| Speed up | `RecreateTransactionRequest` | `TxKeyResult` |
| Approval detail | `ApprovalDetailTransactionsRequest` | `ApprovalDetailTransactionsResponse` |

---

## Chain-Specific Transaction Behaviors

### Speed-Up (Transaction Acceleration) Support

Supported blockchains (per Safeheron API docs):

| Category | Supported Chains |
|----------|-----------------|
| EVM chains | ETH, ETC, BSC, MATIC, etc. |
| UTXO chains | Bitcoin, Dogecoin (RBF and CPFP), and other UTXO-based chains supporting CPFP |
| Others | FileCoin, Aptos, XRP Ledger |
| NOT supported | Any chain not listed above (e.g. NEAR, SUI, TRON, SOL, TON — not listed in the published docs as supported) |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred. Max transfer = balance - 0.002 SOL.
- **Aptos**: Token transfers are NOT supported (recipient must call `register` first — not yet implemented).
- **TRON**: Fee has no LOW/MID/HIGH tiers — pre-executed on-chain; all tiers return the same fee. `sourceAddress` required for fee estimation to check staking/energy.
- **TRON negative fee**: Can occur when `bandwidth_available < 360` AND `energy_available > 360`.
- **Contract addresses**: API transfers to contract addresses require `failOnContract = false` (default `true` blocks them).
- **destinationAccountKey rules**:
  - `destinationAccountType = VAULT_ACCOUNT` → pass `accountKey`
  - `destinationAccountType = WHITELISTING_ACCOUNT` → pass `whitelistKey`
  - `destinationAccountType = ONE_TIME_ADDRESS` → do NOT pass `destinationAccountKey`; use `destinationAddress`

---

## Best Practices

1. Always set `customerRefId` from your own DB record ID before calling create.
2. Store the returned `txKey` — it's Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call `/v1/transactions/one` periodically to re-request failed deliveries.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) — never use float/double.
6. On timeout, **retry with the same `customerRefId`** — Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
