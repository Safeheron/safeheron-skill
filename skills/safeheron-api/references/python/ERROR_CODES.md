# Common Error Codes & Troubleshooting

## API Error Codes

| Code | Message | Root Cause | Resolution |
|------|---------|-----------|------------|
| `1010` | Parameter decryption failed | Wrong **Safeheron platform public key** (`safeheronPublicKey`) in config | Verify and update `safeheronPublicKey` in your config dict -- copy the exact value from Safeheron Console |
| `1012` | Signature verification failed | Wrong **your RSA private key** (`privateKey`) in config, or private key not in PKCS8 format | Verify `privateKey` is PKCS8-encoded (strip BEGIN/END headers); ensure it matches the public key uploaded in Console |
| `9001` | Merchant unique business ID already exists | Duplicate `customerRefId` | Each call must use a unique `customerRefId`; use `str(uuid.uuid4())` |
| `9028` | No matching transaction policy | The transaction matched no transaction policy rule (not specific to MPC Sign); for MPC Sign, the special MPC Sign policy must also be enabled | Configure a matching transaction policy in Web Console; for MPC Sign, contact Safeheron Support to enable the MPC Sign policy |

---

## Reading the Error Code in Code

`safeheron_api_sdk_python` (1.1.25) does not define its own exception class. What a failed call raises:

| Situation | Raised | How to read it |
|-----------|--------|----------------|
| API returned an error code (e.g. 9001, 9028) | `Exception(response_dict)` | `e.args[0]` is a dict: `{'code': 9028, 'message': '...', ...}` |
| HTTP status not 2xx | `requests.HTTPError` (from `raise_for_status()`) | `e.response.status_code` |
| Timeout / connection error | `requests.Timeout` / `requests.ConnectionError` | -- (no API code) |
| Response signature check failed | `Exception("rsa verify: false")` | `e.args[0]` is a string |

```python
import requests


def safeheron_error_code(e):
    """Return the Safeheron API error code, or None if the exception is not an API error response."""
    detail = e.args[0] if e.args else None
    if isinstance(detail, dict) and 'code' in detail:
        return detail['code']
    return None


try:
    resp = transaction_api.create_transactions_v3(param)
except requests.RequestException:
    raise                      # network / HTTP error -- outcome UNKNOWN, see below
except Exception as e:
    code = safeheron_error_code(e)
    if code == 9028:
        alert_ops("No matching transaction policy -- configure the policy in Web Console")
    raise
```

> ⚠️ **Create-transaction calls: an unknown error is NOT a failure.** On a timeout, connection error, HTTP error, a response that fails `rsa verify`, or any exception without an API error code, the transaction may already have been created. Do **not** mark the withdrawal failed or create it again with a new `customerRefId`. Query it first by the same `customerRefId` (`OneTransactionsRequest.customerRefId` → `transaction_api.one_transactions()`), or retry with the **same** `customerRefId` (idempotent — a duplicate returns error 9001 instead of creating a second transaction). Only a definite API error code (e.g. 9028, insufficient balance) means "not created".

---

## Configuration Errors

### Error: `Signature verification failed (1012)`

**Symptom:** Every API call returns error code 1012.

**Checklist:**
1. Is `privateKey` in **PKCS8 format**? Generate with:
   ```bash
   openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
   ```
2. Did you strip the PEM header/footer lines? The value should be pure base64 (no `-----BEGIN...` lines).
   Alternatively, use `privateKeyPemFile` to point to the PEM file directly.
3. Does your private key match the **public key** uploaded in Safeheron Console?
4. Is the config using `privateKey` (not `safeheronPublicKey` by mistake)?

---

### Error: `Parameter decryption failed (1010)`

**Symptom:** API call returns error code 1010.

**Checklist:**
1. Is `safeheronPublicKey` set to the **Safeheron platform public key** (from Console), not your own public key?
2. Did you copy the full value including all base64 characters (watch for line breaks)?
3. Note: Python SDK uses `safeheronPublicKey` (NOT `safeheronRsaPublicKey`).

---

### Error: `Illegal IP`

**Symptom:** Co-Signer logs show "Illegal IP", or API returns IP-related error.

**Resolution:** Register your server's IP address in Safeheron Console -> API Key -> IP Whitelist.
Safeheron supports IPv4 and IPv6 formats. Safeheron's own outbound IPs (for webhooks/callbacks):
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

---

### Error: `9001` -- Duplicate customerRefId

**Symptom:** API returns 9001 on retry.

**Explanation:** `customerRefId` idempotency is a feature -- if you retry with the **same** `customerRefId`, Safeheron returns the original transaction. Error 9001 only occurs if you're intentionally passing a non-unique ID and expecting a new transaction.

**Resolution:**
- For new transactions: always generate a fresh `str(uuid.uuid4())`.
- For retries after timeout: reuse the same `customerRefId` to retrieve the original result.

---

### Error: `9028` -- No Matching Transaction Policy

**Symptom:** a request returns error 9028 ("No matching transaction policy") — e.g. creating a transaction or an MPC Sign.

**Resolution:** the request matched no transaction policy rule — add or adjust the transaction policy in Web Console (subscribe to the `NO_MATCHING_TRANSACTION_POLICY` webhook to be alerted). For **MPC Sign**, a special security policy must also be enabled — contact Safeheron Support via email:
https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le

---

## Python-Specific Issues

### Error: `KeyError` on config dict

**Symptom:** `KeyError: 'apiKey'` or similar when creating an API instance.

**Resolution:** Ensure your config dict contains all required keys:
```python
config = {
    'apiKey': '...',
    'privateKey': '...',       # OR 'privateKeyPemFile': '/path/to/key.pem'
    'safeheronPublicKey': '...',  # NOT 'safeheronRsaPublicKey'
    'baseUrl': 'https://api.safeheron.vip',
    'requestTimeout': 20000,
}
```

---

### Error: Amount precision issues

```python
# WRONG -- float precision loss
amount = 0.1 + 0.2  # = 0.30000000000000004
param.txAmount = str(amount)  # "0.30000000000000004"

# CORRECT -- use Decimal
from decimal import Decimal
amount = Decimal("0.1") + Decimal("0.2")  # = Decimal("0.3")
param.txAmount = str(amount)  # "0.3"
```

---

### Error: Web3 API with Vault Account key

**Symptom:** Web3 API returns "account not found".

**Cause:** Web3 APIs require a **Web3 wallet**. Using a regular vault account key causes this error.

**Resolution:** Create a Web3 wallet in Safeheron Console, then copy its `accountKey`.

---

### Error: MPC Sign hash with "0x" prefix

```python
# WRONG -- includes "0x" prefix
param.dataList[0].data = "0xa1b2c3d4..."

# CORRECT -- strip "0x"
param.dataList[0].data = tx_hash.hex()  # already without "0x" if using bytes.hex()
# or if hash starts with "0x":
param.dataList[0].data = hash_str[2:]  # strip "0x"
```

---

## Fee-Related Issues

### Transaction fails with fee error

- If using `feeRateDto`, both `gasLimit` AND `feeRate` are required -- providing only `gasLimit` is insufficient.
- `txFeeLevel` and `feeRateDto` cannot both be set; `txFeeLevel` takes precedence.

### TRON fee estimation returns 0

- For TRON fee estimation, `sourceAddress` is required to check on-chain staking/energy.
- If the address has no TRX staked, bandwidth/energy is 0 and estimated fee will be based on TRX burn.

---

## Webhook / Callback Issues

| Symptom | Cause | Resolution |
|---------|-------|------------|
| Webhook not received | URL not accessible from public internet | Test with `curl` from Safeheron IP range |
| Webhook stops after URL change | Old URL deleted | Only deletion stops webhooks; URL modification redirects them |
| Duplicate webhook events | Safeheron retries on non-200 | Implement idempotency by checking `txKey` |
| Callback `Timestamp out of range` | Server clock drift > 5s | Sync server time with NTP |
| Co-Signer callback not triggered | Transaction not in approval flow | Callback fires only when a transaction needs signing approval |

---

## Nonce Issues (EVM Chains)

Nonce is assigned when a transaction is **approved** (enters SIGNING):
1. If `maxUseNonce` is null, only `chainNonce` is used.
2. `nonce < chainNonce` -> error "Nonce too low".
3. `nonce > maxUseNonce + 1` -> error "Nonce cannot exceed maxUseNonce+1".

---

## Safeheron Server Timezone

Safeheron's servers operate in **UTC+0**. All timestamps in API responses/webhooks are Unix milliseconds (timezone-agnostic), but any log-based debugging should account for UTC.

---

## Full Error Code List

All error codes returned by the Safeheron API (from the official API documentation):

| Code | Description |
|------|-------------|
| `200` | Success |
| `415` | Unsupported Media Type |
| `429` | Too many requests |
| `500` | System error |
| `1001` | Unknown error, please contact Safeheron Support |
| `1004` | Duplicate request |
| `1005` | User doesn't have permission |
| `1008` | Invalid request |
| `1009` | Invalid API key |
| `1010` | Parameter decryption failed |
| `1011` | Invalid IP |
| `1012` | Signature verification failed |
| `1013` | Timestamp verification failed |
| `1014` | IP exceeds frequency limit |
| `1015` | Invalid user |
| `1016` | Invalid API key |
| `1017` | Parameter validation error |
| `1018` | Request record doesn't exist |
| `1019` | Parameter parsing exception |
| `1020` | Wrong request parameter |
| `1021` | Coin key doesn't exist |
| `1022` | RSA public key error |
| `1023` | You cannot request the current interface, please contact customer service to upgrade your account |
| `1024` | WebHook has expired or does not exist |
| `1026` | The API Key has been deactivated. You can check the reason for deactivating it on the Web Console and enable it again |
| `1027` | Compliance service is not enabled for this team. Contact your account manager to enable it |
| `1028` | Transfers are restricted due to an overdue invoice. Access resumes once payment is confirmed |
| `3009` | Frozen merchant |
| `3010` | Closed merchant |
| `3016` | Failed to request basic service |
| `3017` | Deactivated wallet |
| `3018` | Invalid merchant |
| `3020` | The transaction is not in 'pending approval' status |
| `3021` | Request transaction result exception |
| `3022` | Request transaction failed |
| `3023` | The current wallet does not allow this operation |
| `3024` | Transaction package data verification failed |
| `3025` | Signing failed |
| `3026` | Record cancelled |
| `3027` | Transaction approval record has been processed |
| `3028` | The current wallet does not allow this operation |
| `3029` | The hash entered is invalid |
| `3030` | The network corresponding to the chain ID does not exist |
| `3031` | Hash temporarily supports 1 element |
| `3032` | Data is required |
| `3033` | Chain ID is required |
| `3034` | Invalid data format |
| `3035` | Data length must be between 0 and 100000 |
| `3036` | Value is required |
| `3037` | Nonce is required |
| `3038` | Invalid value |
| `3039` | Invalid gas price |
| `3040` | Gas limit too high |
| `3041` | Invalid max priority fee per gas |
| `3042` | Invalid max fee per gas |
| `3043` | Gas limit is required |
| `3044` | Max priority fee per gas is lower than current value on network |
| `3045` | Max fee too low |
| `3046` | Gas price and fee per gas cannot both be empty |
| `3047` | Data length must be between 0 and 1000000 |
| `3048` | Nonce too low |
| `3049` | Nonce too high |
| `3050` | Balance verification is required for UTXOs |
| `3051` | This asset is not supported yet |
| `3052` | Version is required |
| `3053` | Version error |
| `3054` | This coin does not support speed-up |
| `3055` | Back up private key before creating wallets |
| `3056` | Fee rate must be greater than 0 |
| `3057` | Gas limit must be greater than 0 |
| `3058` | Max priority fee must be greater than 0 |
| `3059` | Max fee must be greater than 0 |
| `3060` | Hashs and data list cannot both be empty |
| `3061` | This algorithm is not supported yet |
| `3062` | This application is not supported yet |
| `3063` | Signature data too long |
| `3064` | The transaction amount cannot be 0 |
| `3065` | Invalid receiving address |
| `3066` | Gas fee cap too low |
| `3067` | The version of Co-Signer too low |
| `3068` | Transaction failed due to exceeding the maximum number of UTXO transactions. Please collect UTXOs and try again |
| `3069` | Duplicate approval data |
| `3070` | Expired security certificate |
| `3071` | Only support to label wallets created by API |
| `3072` | The existing wallet has been used to sweep the target account |
| `3073` | The non-UTXO type does not allow batch transactions |
| `3074` | Request parameter cannot be empty |
| `3075` | Token Account address not supported, please use Owner address |
| `3076` | Unsupported accelerated transactions |
| `3077` | Transactions via Connect are not supported |
| `3078` | The wallet has no DEPOSIT label |
| `3079` | No assets need to be swept in the wallet |
| `3080` | There is no corresponding auto-sweep policy in place |
| `3081` | The wallet address is flagged as malicious |
| `3082` | The wallet balance is less than the |
| `3083` | The sweeping amount does not meet the USD requirement |
| `3084` | There is already an ongoing sweeping task |
| `3085` | Account does not support sweeping |
| `3086` | This network does not support sweeping |
| `3087` | Sign input data must be even |
| `3088` | The time interval cannot exceed 1 hour |
| `3089` | Only supports pushing events that failed within the past 7 days |
| `3090` | The transaction amount cannot be less than 0.01 TON |
| `3091` | The transfer amount cannot be less than the minimum value for the CoinKey |
| `3092` | This service has not been activated yet. You can contact Support to apply for activation |
| `3093` | The requestId does not exist or the evaluation result has exceeded 5 minutes |
| `3094` | Unsupported networks, only support Bitcoin, Ethereum and Tron |
| `3095` | Wrong address format |
| `3096` | Request frequency exceeds limit, currently limited to 1 QPS (Web Console, Safeheron App and API share this quota) |
| `3097` | The specified network is not supported. See '/v1/compliance/kya/supportedNetworks' for available options |
| `3098` | The requested provider and network combination is not supported. See '/v1/compliance/kya/supportedNetworks' for valid combinations |
| `3099` | The address does not match the specified network or chainType |
| `3100` | The requested provider and chainType combination is not supported. See '/v1/compliance/kya/supportedNetworks' for valid combinations |
| `3101` | One of the specified providers is not enabled, or its API Key is missing |
| `3102` | The quota for one of the specified providers has been exhausted. Please contact Safeheron support |
| `3103` | Field 'network' is required when 'providers' contains MistTrack |
| `3104` | Field 'providers' must contain at least one unique value from: MistTrack, Elliptic, Chainalysis |
| `3105` | Fee Rate exceeds maximum limit |
| `9001` | Merchant unique business ID already exists |
| `9002` | System under maintenance, and asset transfer is suspended |
| `9003` | Wrong decimal for the amount sent |
| `9004` | Wrong decimal for fee rate |
| `9005` | Hidden account cannot change name |
| `9006` | Hidden account cannot withdraw assets |
| `9007` | Frozen source account |
| `9008` | Address already exists |
| `9009` | Wrong address format |
| `9011` | The whitelist does not exist |
| `9012` | Address doesn't exist |
| `9013` | The whitelist address status is illegal |
| `9015` | Frozen destination account |
| `9016` | The destination address may pose compliance risks. Please proceed with caution. If you fully understand the associated risks and still need to transfer funds to this address, set the failOnAml parameter to false and resubmit the transaction |
| `9017` | Insufficient transaction fee |
| `9018` | Transaction fee too high |
| `9019` | Transaction fee too low |
| `9020` | Source address and destination address cannot be the same |
| `9021` | The transaction to be replaced doesn't exist |
| `9022` | Only broadcasting transaction can be sped up |
| `9023` | This coin does not support speed-up |
| `9024` | Unauthorized user tried to initiate or approve a transaction |
| `9025` | The withdrawal amount is below the minimum limit |
| `9026` | Insufficient amount for withdrawal |
| `9027` | Matched with the policy, but the approver does not possess the approval role |
| `9028` | No matching transaction policy |
| `9029` | The destination address does not match the whitelisted address |
| `9030` | Non-cancellable status |
| `9031` | The estimated online transaction fee is greater than the maximum transaction fee |
| `9032` | This coin is not UTXO-based type |
| `9033` | The whitelist address already exists |
| `9034` | Address and ID are not matched |
| `9035` | This transaction record doesn't exist |
| `9036` | Account doesn't exist |
| `9037` | Account coin doesn't exist |
| `9038` | This coin is not supported yet |
| `9039` | The number of wallet accounts exceeds the limit |
| `9040` | The number of addresses exceeds the limit |
| `9041` | Wallet name already exists |
| `9042` | Do not meet sweeping requirements for UTXOs |
| `9043` | MPC Sign currently only supports one account |
| `9044` | Invalid hash |
| `9045` | The whitelist address is already associated with a transaction policy. Please unbind it before proceeding |
| `9047` | Gas limit is less than the minimum value |
| `9048` | Abnormal risk control detection |
| `9049` | The destination address is a contract address, which may pose security risks. Please proceed with caution. If you fully understand the associated risks and still need to transfer funds to this contract address, set the failOnContract parameter to false and resubmit the transaction |
| `9050` | Address format error |
| `9051` | Please wait patiently during transaction broadcasting |
| `9052` | The transaction output already used and cannot be accelerated |
| `9053` | Transaction already on the chain or nonce already used |
| `9054` | The whitelist addresses must be approved before any transfer is allowed |
| `9055` | Insufficient UTXO |
| `9056` | Duplicate speed-up |
| `9057` | To accelerate a transaction using RBF, the fee rate of the sped-up transaction must be higher than that of the original transaction |
| `9058` | GasBudget too low |
| `9059` | GasPrice too low |
| `9060` | Insufficient SUI object |
| `9061` | The handling fee is greater than the collected amount |
| `9062` | The whitelist address does not exist or has been deleted |
| `9063` | GasUnitPrice too low |
| `9064` | GasUnitPrice too high |
| `9065` | MaxGasAmount too low |
| `9066` | Replace a transaction on Aptos should have a higher gas unit price than that of the original transaction |
| `9067` | Replace a transaction on Aptos should have a max gas amount equal to that of the original transaction |
| `9068` | SequenceNumber too low |
| `9069` | SequenceNumber too high |
| `9070` | The accelerated transaction has been linked or sequenceNumber has been occupied |
| `9071` | The whitelist name already exists |
| `9072` | Unsupported chain types |
| `9073` | The number of whitelist exceeds the limit |
| `9074` | There are pending transactions related to the whitelist. Please make the modifications after the processing is completed |
| `9075` | During modification approval, modification is prohibited |
| `9076` | The system only supports whitelisting the destination address of a transaction if it is a one-time address |
| `9077` | For your security, the system only supports whitelisting the destination address of a completed transaction |
| `9078` | The address or memo data in the parameters does not match the transaction record |
| `9079` | This transaction type is not supported |
| `9080` | The custom network you are using has not been added yet. Please add it using the Safeheron Browser Extension and try again |
| `9081` | Only TON wallet addresses are supported for whitelisting. Jetton wallet addresses are not supported |
| `9082` | The destination address or memo does not match the whitelist |
| `9083` | Coins that are only supported on the Tron network |
| `9084` | Anonymous transfer through the MWEB feature is not supported. Please enter a different address |
| `9085` | Gas rate is too low |
| `9086` | Profile doesn't exist |
| `9087` | Profile has been disconnected or deleted |
| `9088` | The profile has been disconnected, you cannot transfer to the other party's profile |
| `9089` | If you haven't activated the Gas Service, you can do so in WebConsole under 'Tools → Gas Service' |
| `9090` | Gas record does not exist |
| `9091` | Confidential transactions are not supported on Safeheron. Please use a standard address |
| `9092` | You are sending to a new address. You must send at least some SOL to cover rent and create the account |
| `9093` | Insufficient network fee. Your account must retain at least some SOL as rent to remain active |
| `9094` | You can choose to zero out the balance, closing your SOL account, or retain at least some SOL to meet the minimum rent requirement, keeping the account active |
| `9095` | After deducting the handling fee, your account balance does not meet the minimum rent requirement and cannot be transferred out |
| `9096` | The number of Sui Objects does not meet the collection requirements, so there is no need to collect |
| `9097` | Transaction has been cancelled |
| `9098` | Transaction failed as the number of UTXO coins exceeds the maximum limit. Please first collect them and try again |
| `9099` | The destination address has been frozen by the token issuer. Funds sent to it cannot be transferred out. Please use another address |
| `9100` | The source address has been frozen by the token issuer and cannot send funds. Please use another wallet |
| `9101` | The memo must be an unsigned 32-bit integer |
| `9102` | The memo must be an integer between 0 and 4294967295 |
| `9103` | Insufficient balance after transfer. An XRP account must retain at least some XRP as reserve |
| `9104` | The destination address is not activated. The transfer amount must be at least some XRP |
| `9105` | The destination address is on the sanctions list. The transaction failed |
| `9106` | The source address is on the sanctions list. The transaction failed |
