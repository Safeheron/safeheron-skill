# Transaction API Reference

## Imports

```python
import uuid
from safeheron_api_sdk_python.api.transaction_api import (
    TransactionApi,
    CreateTransactionRequest,
    ApprovalDetailTransactionsRequest,
    OneTransactionsRequest,
    ListTransactionsV2Request,
    ListTransactionsV1Request,
    TransactionsFeeRateRequest,
    CancelTransactionRequest,
    RecreateTransactionRequest,
)
```

## Create API Instance

```python
transaction_api = TransactionApi(config)
```

---

## Create a Transaction

### V2 (standard — returns txKey only)

```python
param = CreateTransactionRequest()
param.customerRefId = str(uuid.uuid4())   # your unique ID
param.coinKey = "ETHEREUM_ETH"
param.txAmount = "0.01"                    # string, never float
param.sourceAccountKey = account_key
param.sourceAccountType = "VAULT_ACCOUNT"
param.destinationAccountType = "ONE_TIME_ADDRESS"
param.destinationAddress = "0xRecipientAddress"
param.txFeeLevel = "MIDDLE"               # "LOW" | "MIDDLE" | "HIGH"
# Optional: param.note = "Payment for invoice #123"
# Optional: param.customerExt1 = "order-id-456"

resp = transaction_api.create_transactions(param)
tx_key = resp['txKey']  # save this -- Safeheron transaction identifier
```

### V3 (extended — returns txKey + idempotency info)

```python
resp = transaction_api.create_transactions_v3(param)
tx_key = resp['txKey']
is_duplicate = resp.get('idempotentRequest', False)  # True = duplicate customerRefId, original txKey returned
```

**CreateTransactionV3Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `customerRefId` | str | Your unique business ID echoed back |
| `idempotentRequest` | bool | `True` if a duplicate `customerRefId` was detected — the original `txKey` is returned instead of creating a new transaction |

### CreateTransactionRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `customerRefId` | Yes | str | Your unique ID for idempotency (max 100 chars) |
| `coinKey` | Yes | str | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `txAmount` | Yes | str | Amount as string -- never use float |
| `sourceAccountKey` | Yes | str | Sender wallet key |
| `sourceAccountType` | Yes | str | `VAULT_ACCOUNT` |
| `destinationAccountType` | Yes | str | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `destinationAddress` | Cond. | str | Required when `destinationAccountType=ONE_TIME_ADDRESS` |
| `destinationAccountKey` | Cond. | str | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `txFeeLevel` | No | str | `LOW` / `MIDDLE` / `HIGH` — Choose between transaction fees. If the transaction fee rate is preset, it will take priority |
| `feeRateDto` | No | dict | Transaction fee rate, either `TxFeeLevel` or `FeeRateDto` (V3: `feeRate` is in drops for XRPL) |
| `maxTxFeeRate` | No | str | Max acceptable fee rate |
| `note` | No | str | Transaction note (max 180 chars) |
| `customerExt1` | No | str | Custom field 1 (max 255 chars) |
| `customerExt2` | No | str | Custom field 2 (max 255 chars) |
| `treatAsGrossAmount` | No | bool | If True, fee is deducted from `txAmount` |
| `memo` | No | str | Memo/tag (TON and XRP Ledger only -- max 100 chars; for XRPL it must be a numeric string in the range 0–4294967295) |
| `failOnAml` | No | bool | Default True -- block high-risk addresses via AML |
| `failOnContract` | No | bool | Block contract addresses as destination |
| `nonce` | No | int | Custom nonce for EVM chains |
| `isRbf` | No | bool | Enable Replace-by-Fee for Bitcoin |

> **Fee priority:** `txFeeLevel` takes precedence over `feeRateDto`. If using `feeRateDto`, you must provide at least `gasLimit` AND `feeRate` -- providing only `gasLimit` will cause the transaction to fail.

---

## Retrieve a Single Transaction

```python
param = OneTransactionsRequest()
param.txKey = tx_key
# OR: param.customerRefId = customer_ref_id

resp = transaction_api.one_transactions(param)
print(f"Status: {resp['transactionStatus']}")
print(f"Sub-status: {resp['transactionSubStatus']}")
print(f"TxHash: {resp.get('txHash', 'N/A')}")
```

**Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `txHash` | str | On-chain transaction hash |
| `coinKey` | str | Coin identifier |
| `txAmount` | str | Transaction amount |
| `txFee` | str | Actual fee paid |
| `transactionStatus` | str | See status table below |
| `transactionSubStatus` | str | Sub-status |
| `sourceAccountKey` | str | Sender wallet key |
| `sourceAddress` | str | Sender address |
| `destinationAddress` | str | Recipient address |
| `createTime` | int | Unix timestamp (ms) |
| `completedTime` | int | Unix timestamp (ms) of completion |
| `customerRefId` | str | Your reference ID |
| `amlLock` | str | Whether the source address contains AML address(es): `YES` = contains / `NO` = does not contain |

---

## List Transactions

### V2 (Recommended -- cursor-based)

```python
param = ListTransactionsV2Request()
param.limit = 20
param.transactionStatus = "COMPLETED"  # optional filter
param.coinKey = "ETHEREUM_ETH"        # optional filter
# param.createTimeMin = start_ms      # default: createTimeMax minus 24 hours (see note below)
# param.createTimeMax = end_ms        # default: now

tx_list = transaction_api.list_transactions_v2(param)
for tx in tx_list:
    print(f"{tx['txKey']} {tx['transactionStatus']}")
```

**ListTransactionsV2Request Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `direct` | str | Page direction: `NEXT` (default) |
| `limit` | int | Items per page, max 500 |
| `fromId` | str | txKey of last record from previous page; omit for first page |
| `sourceAccountKey` | str | Source account key |
| `coinKey` | str | Coin key, multiple separated by commas |
| `transactionStatus` | str | Transaction status |
| `createTimeMin` | int | Start time, UNIX timestamp (ms). **Default: `createTimeMax` minus 24 hours** |
| `createTimeMax` | int | End time, UNIX timestamp (ms). Default: current UTC time |
| `transactionDirection` | str | `INFLOW` / `OUTFLOW` / `INTERNAL_TRANSFER`; default: all |

> ⚠️ Without `createTimeMin`, V2 only returns transactions created in the **last 24 hours**. For history or reconciliation, set an explicit `createTimeMin` and page with `fromId` (txKey of the last record) until a page returns fewer than `limit` items.

### V1 (legacy -- page-based)

```python
param = ListTransactionsV1Request()
param.pageSize = 10
param.pageNumber = 1

result = transaction_api.list_transactions_v1(param)
tx_list = result['content']
```

---

## Estimate Transaction Fee

```python
param = TransactionsFeeRateRequest()
param.coinKey = "ETHEREUM_ETH"
param.value = "0.01"
param.sourceAccountKey = account_key
param.destinationAddress = "0xRecipient"
# SDK 1.1.25 defaults destinationAddressList to the class object itself, which makes json.dumps fail
# ("Object of type type is not JSON serializable") — always overwrite it (None, or a list of dicts for UTXO multi-destination)
param.destinationAddressList = None

resp = transaction_api.transaction_fee_rate(param)
print(f"Low:    {resp['lowFeeRate']['feeRate']} (fee: {resp['lowFeeRate']['fee']})")
print(f"Middle: {resp['middleFeeRate']['feeRate']} (fee: {resp['middleFeeRate']['fee']})")
print(f"High:   {resp['highFeeRate']['feeRate']} (fee: {resp['highFeeRate']['fee']})")
```

**FeeRate Object Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `feeRate` | String | Fee rate: UTXO fee per byte, EVM gas price, TRON fee limit, SUI gas price, XRPL drops |
| `fee` | String | Estimated total fee amount |
| `gasLimit` | String | Gas limit (EVM chains only) |

> **BTC fee** = feeRate x bytesSize. **ETH fee** = feeRate x gasLimit.
> **TRON fee**: `sourceAddress` must be provided to check on-chain staking/energy.
> **TON / XRP Ledger memo**: the Estimate Transaction Fee API accepts a `memo` parameter (same rules as `CreateTransactionRequest.memo`), but `TransactionsFeeRateRequest` in Python SDK 1.1.25 does not declare it yet. Since the SDK serializes the request's `__dict__`, you can set it dynamically: `param.memo = "104729"`.

---

## Cancel a Transaction

```python
param = CancelTransactionRequest()
param.txKey = tx_key

transaction_api.cancel_transactions(param)
```

> Only transactions in `SUBMITTED` or `SIGNING` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

Replaces a stuck transaction with a higher fee version:

```python
param = RecreateTransactionRequest()
param.txKey = original_tx_key
param.txHash = original_tx_hash   # on-chain hash of the transaction being replaced
param.coinKey = coin_key          # coin of the original transaction
param.txFeeLevel = "HIGH"         # use higher fee

resp = transaction_api.recreate_transactions(param)
new_tx_key = resp['txKey']
```

> Provide at least one of `txFeeLevel` / `feeRateDto` — the service rejects the request if both are empty (service-side validation).
>
> The original transaction and speed-up are independent. The speed-up will invalidate the original (original -> FAILED).

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
| `FAILED` | Final state — no assets were transferred; see `transactionSubStatus` (table below) for the reason |
| `REJECTED` | Rejected by a team member's approval, or by a Connect recipient's approval |

### Status Flow

```
SUBMITTED
    -> SIGNING
    -> BROADCASTING
    -> CONFIRMING
    -> COMPLETED
    +-- FAILED

    At SUBMITTED or SIGNING:
    -> REJECTED (denied by approver)
    -> CANCELLED (cancelled by team member)
```

### Transaction Substatus (`transactionSubStatus`)

`transactionSubStatus` gives the detailed sub-state / failure reason of a transaction:

| Code | Description |
|------|-------------|
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

| Code | Description |
|------|-------------|
| `NORMAL` | Normal transaction |
| `UTXO_COLLECTION` | UTXO-sweeping transaction |
| `ABNORMAL` | Transfers to external destinations (not in the Safeheron platform) or transactions signed by the original private key recovered by key shards |
| `AUTO_SWEEP` | Auto-sweeping transaction |
| `AUTO_FUEL` | Auto-fueling transaction |
| `UTXO_BATCH_TRANSFER` | UTXO multi-address transfer |
| `BOUNCED_TRANSFER` | TON's bounced transaction |
| `GAS_SERVICE_DEPOSIT` | Gas Service Deposit |
| `GAS_SERVICE_FUEL` | Gas Service Fuel |

---

## Get Transaction Approval Details

```python
param = ApprovalDetailTransactionsRequest()
param.txKeyList = [tx_key]  # max 20 keys

resp = transaction_api.approval_detail_transactions(param)
for detail in resp['approvalDetailList']:
    print(detail['txKey'], detail['approvalStatus'], detail['policyName'])
```

**ApprovalDetailTransactionsRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKeyList` | Yes | list[str] | List of transaction keys to query (max 20) |

**ApprovalDetail Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `approvalStatus` | str | Current approval status (see below) |
| `policyName` | str | Name of the matched approval policy |
| `approvalProgress` | object | Approval progress details |

**`approvalStatus` Values:**

| Value | Description |
|-------|-------------|
| `PENDING_APPROVAL` | Awaiting approver action |
| `APPROVED` | Approved — transaction proceeds to signing |
| `REJECTED` | Denied by an approver |
| `CANCELLED` | Cancelled before completion |
| `BLOCKED_BY_POLICY` | Blocked by a policy rule |
| `FAILED` | Approval process failed |

> **Note:** Results exclude transactions using old policies, non-existent txKeys, and incoming fund transactions.

---

## Chain-Specific Transaction Behaviors

### Speed-Up Support

| Category | Supported Chains |
|----------|-----------------|
| EVM chains | ETH, ETC, BSC, MATIC, etc. |
| UTXO chains | Bitcoin, Dogecoin (RBF and CPFP), and other UTXO chains supporting CPFP |
| Other | Filecoin, Aptos, XRP Ledger |
| NOT supported (not listed in the published docs as supported) | NEAR, SUI, TRON, SOL, TON |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred.
- **TRON**: Fee has no LOW/MID/HIGH tiers -- pre-executed on-chain; all tiers return the same fee. `sourceAddress` required for fee estimation.
- **Contract addresses**: API transfers to contract addresses require `failOnContract = False` (default `True` blocks them).
- **destinationAccountKey rules**:
  - `destinationAccountType = VAULT_ACCOUNT` -> pass `accountKey`
  - `destinationAccountType = WHITELISTING_ACCOUNT` -> pass `whitelistKey`
  - `destinationAccountType = ONE_TIME_ADDRESS` -> do NOT pass `destinationAccountKey`; use `destinationAddress`

---

## Best Practices

1. Always set `customerRefId` from your own DB record ID before calling create.
2. Store the returned `txKey` -- it's Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call one_transactions periodically as fallback.
4. Credit/debit accounts only on `COMPLETED` status.
5. All amounts are **strings** (e.g. `"0.05"`) -- never use float.
6. On timeout, **retry with the same `customerRefId`** -- Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.
