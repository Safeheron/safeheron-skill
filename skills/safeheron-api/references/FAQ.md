# Safeheron API — Frequently Asked Questions (FAQ)

This file covers real-world questions from developers and operators integrating the Safeheron API and SDK.

---

## API Co-Signer

**Q: What is the difference between the new and old Co-Signer deployment, specifically the "callback URL" in Web Console?**

A: The new version of API Co-Signer only requires you to generate **one RSA key pair**. The old version required **two RSA key pairs**. The "callback URL" and its associated public key in the new Web Console correspond to what was previously configured as `biz_callback` and `biz_pubkey` in the old config file. Generate the public key with:
```bash
openssl rsa -in api_private.pem -out api_public.pem -pubout
```

---

**Q: Configuring the Approval Callback Service — which keys are needed? How to export the Co-Signer public key?**

A: You need two keys:
1. **Co-Signer identity public key** — export with: `sudo ./cosigner export-public-key`
   Your callback service uses this to verify that callback requests truly come from Co-Signer.
2. **Callback private key** — the private key corresponding to the callback public key you uploaded in Console when registering the API Key.
   Your callback service uses this to decrypt the AES key in the callback payload.

---

**Q: Why is a .env config file still required if Secrets Manager is already used?**

A: Secrets Manager is recommended for managing sensitive data (private keys, etc.), but some configuration values cannot be managed through Secrets Manager and must still be set in the `.env` file.

---

**Q: After modifying the .env config file, how do I make the changes take effect?**

A: Stop and restart Co-Signer:
```bash
sudo ./cosigner stop
sudo ./cosigner start
```

---

**Q: What format is required when uploading the callback public key to Web Console?**

A: Use PEM format — the key file generated via OpenSSL command, starting with `-----BEGIN PUBLIC KEY-----`. When uploading to "API Key" configuration, the key should be on a single line (no line breaks). The Co-Signer identity public key exported via `export-public-key` is already in single-line format.

---

**Q: Can the callback service and Co-Signer be deployed on the same internal network, using an internal IP for the callback URL?**

A: Yes, as long as the machine running the callback service is reachable from Co-Signer at that internal IP address.

---

**Q: When is the callback URL required vs. optional?**

A:
- **Production teams**: Callback URL is **required**.
- **Test/sandbox teams**: Callback URL is **optional**.

---

**Q: Co-Signer failed to start, or approval tasks are stuck in "Waiting for API Co-Signer". How to diagnose?**

A: Check the logs:
```bash
./cosigner logs -f        # stream live logs
./cosigner logs -s        # export logs to file, send to Safeheron Support
```
The logs will contain specific error messages.

---

**Q: Co-Signer log shows: "The final private key fragment is not exists, please activate API Co-Signer first." — is this an error?**

A: This is **normal** — it means Co-Signer has started successfully. You need to proceed with the Co-Signer activation workflow in the App.

---

**Q: What is the polling interval for Co-Signer to check for pending approval tasks?**

A: v2.0+ (new version): **every 5 seconds**. v1.x (old version): **every 1 second**.

---

**Q: Which KMS providers are supported for Co-Signer deployment?**

A: AWS KMS and GCP KMS are supported. **Alibaba Cloud (Aliyun) KMS is NOT supported** — because Aliyun KMS requires the server to be on the same VPC, which is incompatible.

---

**Q: Co-Signer logs show "Illegal IP" after startup. Why?**

A: The Co-Signer host machine's IP address has not been added to the API Key's IP whitelist in Safeheron Console. Add the IP, then restart Co-Signer.

---

**Q: When using AWS with MySQL configured, Co-Signer was started with the `enable-mysql` parameter — which config takes precedence?**

A: When started with `--enable-mysql`, Co-Signer will use the MySQL configuration from AWS Secrets Manager (not the local `.env` file).

---

**Q: CLI `setup` or `start` command fails with: "Failed to log in to the Docker image repository. The token may have expired."**

A: Known causes:
1. **Built-in token expired** (less common) — re-download the CLI from Safeheron Web Console.
2. **`start` run before `setup`** — run `docker images` to confirm Docker is ready, then run `setup` first.
3. **Firewall blocking `registry.gitlab.com`** — whitelist this domain.
4. **OS image issue** — Known to occur on Debian 4.19.316 with error: `Cannot autolaunch D-Bus without X11 $DISPLAY`. Solution: switch to **Ubuntu 24.04**.

---

**Q: After updating the Callback URL in Web Console, does Co-Signer need to be restarted?**

A: **No restart needed.** The updated Callback URL takes effect within **5 minutes** automatically.

---

**Q: What is the callback timeout? I see error: "Timestamp out of range".**

A: Your callback service's response timestamp must be **within 5 seconds** of the Co-Signer server's current time. Ensure both servers' clocks are synchronized (NTP). The error log looks like: `Failed to execute Approval Callback, error: Timestamp out of range`.

---

**Q: After cancelling a transaction via Web/API, why does Co-Signer keep calling the Callback Service?**

A: In Co-Signer versions **before 2.x**, you must also manually update the Co-Signer database to cancel the stored audit task:
```sql
UPDATE `mpc_client_trans_audit_record`
SET audit_status = 2
WHERE trade_no = '${tx_key}';
```
In v2.x and later, this is handled automatically.

---

## API Integration

**Q: How do I add coins to a newly created wallet? What are the caveats?**

A: Use the V2 API (recommended) — supports adding up to **20 coins at once** via `coinKeyList`:
- Adding a **token** coinKey automatically adds its **mainnet** coin too.
- Adding a **mainnet** coinKey only adds that mainnet coin.
- Re-adding an already-added coinKey returns the same response (idempotent).
- If a coinKey was manually disabled in the UI after being added, the API **cannot** re-enable it.

See **Add Coin(s) to a Wallet Account** in `references/{lang}/COIN_API.md` for language-specific examples.

---

**Q: How to distinguish deposit (inflow) vs. withdrawal (outflow) vs. internal transfer by transaction type?**

A: Check `destinationAccountType` and `sourceAccountType`:

| Scenario | destinationAccountType | sourceAccountType |
|----------|----------------------|------------------|
| Withdrawal (outflow) | `ONE_TIME_ADDRESS` | `VAULT_ACCOUNT` |
| Internal transfer | `VAULT_ACCOUNT` | `VAULT_ACCOUNT` |
| Deposit (inflow) | — | `UNKNOWN` |

---

**Q: First time using MPC Sign, getting error 9028. Why?**

A: MPC Sign is a low-level cryptographic operation with high security requirements. A **special MPC Sign policy** must be enabled on your team first. Contact Safeheron Support via email using the official template:
https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/mpc-sign-policy

---

**Q: What are Safeheron's outbound (egress) IP addresses? (For configuring firewall whitelists to receive webhooks/callbacks)**

A: Safeheron's outbound IPs for webhooks and callbacks:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

---

**Q: Failed to configure a Webhook URL — it keeps showing an error.**

A: Ensure the Webhook URL is publicly accessible from the internet. For ngrok URLs, append `/webhook` to make it routable. Note: some public tunnel domains (ngrok, webhook.site, etc.) may be blocked by Akamai CDN.

---

**Q: What timezone does Safeheron's server use?**

A: **UTC+0 (UTC)**. All API timestamps are Unix milliseconds (timezone-agnostic).

---

**Q: A transaction is stuck in PENDING. How to differentiate an original transaction from its speed-up transaction?**

A: The original and speed-up are **independent transactions**. Distinguish them via:
- `replaceTxKey` — on the speed-up tx, references the original
- `replacedCustomerRefId` — on the speed-up tx, references the original's customerRefId
- `speedUpHistory` — on the original tx (from `/v1/transactions/one`), lists its speed-up transactions

The speed-up transaction will cause the original to fail.

---

**Q: What protocol is required for Webhook URL and Callback URL?**

A: HTTP or HTTPS. URL must start with `http://` or `https://`. Some specific domains may be blocked by Akamai (ngrok, webhook.site, other public tunnel tools).

---

**Q: Web3 API returns "account not found". Why?**

A: All Web3 API calls require a **Web3 wallet**. Using a regular vault account key (`accountType = VAULT_ACCOUNT`) causes this error.

---

**Q: After Auto Sweep (归集) completes, does Safeheron send a webhook notification?**

A: Yes. Sweep transactions, gas refill transactions, speed-up transactions, batch transfers — all are treated as regular transactions internally and all generate Webhook notifications.

---

**Q: How to query the balance of USDT/ETH on a specific MPC wallet address?**

A: Call `/v1/account/coin/list` and read the `balance` field in the response:
See **List Coins for a Wallet Account** in `references/{lang}/COIN_API.md` for language-specific examples.

---

**Q: How to query the aggregate balance of a specific coinKey across all wallets in the team?**

A: Call `/v1/coin/balance/snapshot` and read `coinBalance`:
See **Coin Balance Snapshot (Team-wide)** in `references/{lang}/COIN_API.md` for language-specific examples.

---

**Q: Is the new SDK backward compatible with the old SDK?**

A: Yes. SDK updates only **add** new methods — all existing methods are preserved. You can upgrade without breaking existing integrations.

---

**Q: After enabling custom Webhook block confirmation count, does it only apply to incoming transactions?**

A: No. Once a custom confirmation count is enabled for a chain, it applies to **both incoming and outgoing** transactions on that chain.

---

**Q: Calling an API with a duplicate customerRefId — what happens?**

A: Safeheron returns **error code 9001** ("Merchant unique business ID already exists"). This is also the idempotency mechanism: if you retry a timed-out call with the **same** customerRefId, Safeheron returns the original transaction rather than creating a duplicate.

---

**Q: What is the priority between `txFeeLevel` and `feeRateDto`? What if only `gasLimit` is set in `feeRateDto`?**

A: `txFeeLevel` takes **priority** over `feeRateDto` if both are provided. If only `feeRateDto` is used, you must provide at least **both `gasLimit` AND `feeRate`** — providing only `gasLimit` will cause the transaction creation to fail.

---

**Q: What IP address formats are supported for API Key IP whitelist?**

A: Both **IPv4** and **IPv6** are supported.

---

**Q: Does Safeheron filter webhook notifications by transaction amount? Will small-amount or dust-attack deposits trigger webhooks?**

A: **No filtering.** Any successful incoming transaction — regardless of amount, including dust attacks — triggers a Webhook notification.

---

**Q: If `hiddenOnUI=false` is set when creating a wallet via API, is the wallet counted as visible or hidden in APP stats?**

A: A wallet with `hiddenOnUI=false` is a **visible account**. Its assets are included in the visible account totals in APP/Web Console.

---

**Q: Do adding/modifying/deleting Webhook URL or RSA public key require admin approval?**

A: **No approval required.** All changes (add/modify/delete) to Webhook URL and RSA public key take effect **immediately**.

---

**Q: How is transaction fee estimated for different blockchains?**

A:

| Chain | Fee Formula | Special Notes |
|-------|-------------|--------------|
| Bitcoin (BTC) | `feeRate × byteSize` | No UTXOs = no balance → fee = 0 |
| Ethereum / EVM | `feeRate × gasLimit` | — |
| TRON | Pre-execution on-chain | `sourceAddress` required to check staking/energy; no LOW/MID/HIGH tiers |
| Solana | Fixed fee | 0.000005 SOL base fee |

---

**Q: What happens when the webhook URL is modified? When does webhook delivery stop?**

A: After URL modification, events are delivered to the **new URL**. Webhook delivery only **stops** when the webhook URL is **deleted** (not merely modified).

---

**Q: Speed-up (accelerate) transaction — how does it relate to the original transaction?**

A: The original and speed-up are **independent transactions**. The speed-up will invalidate the original (original → FAILED). The speed-up is also reported via Webhook. Query speed-up history via `speedUpHistory` field in `/v1/transactions/one`.

---

**Q: Can the DEPOSIT label on a wallet be removed? Can it be re-applied?**

A: Yes. Pass `accountTag = "NONE"` to remove the DEPOSIT label. It can be re-applied later by passing `accountTag = "DEPOSIT"` again.
See **Batch Label Wallet Accounts** in `references/{lang}/WALLET_API.md` for language-specific examples.

---

**Q: When `destinationAccountType` is `VAULT_ACCOUNT`, is `destinationAccountKey` required?**

A: Rules:
- `destinationAccountType = VAULT_ACCOUNT` → `destinationAccountKey` must be the **accountKey**
- `destinationAccountType = WHITELISTING_ACCOUNT` → `destinationAccountKey` must be the **whitelistKey**
- `destinationAccountType = ONE_TIME_ADDRESS` → `destinationAccountKey` is **NOT used** (set `destinationAddress` instead)

---

**Q: Webhook retry schedule — what happens when delivery fails?**

A: Retry schedule after first failure:
`30s → 1min → 5min → 1h → 12h → 24h` — total **7 attempts**. After the 7th failure, no further retries.

Call `POST /v1/webhook/resend/failed` to manually retry all failed events.

---

## Transactions

**Q: SOL (Solana) transfer fails on-chain. Why?**

A: Solana accounts have a **minimum rent-exempt balance** of approximately **0.002 SOL**. You cannot transfer the full balance — the maximum transfer amount is `(balance - 0.002 SOL)`.

---

**Q: Which test networks does Safeheron support in production? Where to get test tokens?**

A:

| Network | Token | Details |
|---------|-------|---------|
| Ethereum Sepolia | GSK | Contract: `0xF191b0720cb49DdAb6ECd72a65955a35b31fc944` |
| Ethereum Sepolia | USDC | Contract: `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238`, Faucet: https://faucet.circle.com/ |
| TRON Shasta | TLK | Contract: `TC2jGCcJUh33oAeQHYHBXu7KVVxrjitkHw` |
| Bitcoin Testnet | BTC | Faucet: https://coinfaucet.eu/en/btc-testnet4/ |

---

**Q: Which chains support speed-up (transaction acceleration)?**

A:
- **UTXO chains with speed-up support**: BTC, BCH, DASH, LTC, DOGE
- **Non-UTXO chains with speed-up support**: All EVM chains, FIL, Aptos, CFX
- **Chains WITHOUT speed-up**: NEAR, SUI, TRON, SOL, TON

---

**Q: What happens to an address flagged by AML? How to handle it?**

A: The recommended approach is to **change the address** and return the contaminated assets to the original sender to maintain compliance. The transaction in Safeheron will have `amlLock = "YES"`.

---

**Q: Under what circumstances are coins added automatically?**

A:
1. If a mainnet coin is enabled and a **token** of that chain is received as deposit → the token coin type is automatically added.
2. If you add a **token** coinKey via API → the corresponding mainnet coin is also automatically added.

---

**Q: Nonce rules for EVM chains**

A: Nonce is determined at **approval time** (when transaction enters SIGNING):
1. If `maxUseNonce = null` → only `chainNonce` is used.
2. `nonce < chainNonce` → error: "Nonce too low".
3. `nonce > maxUseNonce + 1` → error: "Nonce cannot exceed maxUseNonce+1".
4. `chainNonce ≤ nonce ≤ maxUseNonce` → confirmation prompt shown to user.

When multiple transactions are pending simultaneously, the one **approved first** gets the recommended nonce first.

---

**Q: TRON fee estimation shows negative values. Why?**

A: This occurs when the sender address has staking AND the conditions are: `bandwidth_available < 360` AND `energy_available > 360`. The fee calculation can yield a negative value in this specific edge case.

---

**Q: Does Aptos support token transfers?**

A: **Not currently supported.** Aptos token transfers require the recipient address to first call `register` for that token type. This requires significant system changes on Safeheron's side, so it is not yet supported.

---

**Q: TRON fee estimation — is there a difference between LOW/MEDIUM/HIGH levels?**

A: **No difference.** TRON fee is determined by **pre-execution on-chain** — whatever the chain returns is the fee. The LOW/MEDIUM/HIGH labels all yield the same value for TRON.

---

**Q: A wallet only has ETH and USDT added, but USDC was deposited — it doesn't show. Why?**

A: The USDC coin type was not added to the wallet. The balance will update once USDC is added via `createAccountCoinV2`. The deposit record may need manual reconciliation.

---

**Q: How to transfer funds to a contract address via API?**

A: Set `failOnContract = false` in the transaction request. The default is `true` (blocked). Note: transferring to contract addresses carries risk — only disable this flag when you explicitly know the destination is a valid contract.

Set `failOnContract = false` in the transaction request — see **Create a Transaction** in `references/{lang}/TRANSACTION_API.md` for language-specific examples.

---

**Q: UTXO sweep (归集) basic rules**

A: Manual/API sweep: threshold is 30 UTXOs. When UTXO count ≤ 30, sweep is not recommended; when > 30, sweep is recommended. Auto Sweep applies its own configured conditions.

---

## Permissions

**Q: What permissions are required for the following operations?**

A: The permission names below are the official labels shown in the Safeheron Console UI (Chinese names are displayed regardless of UI language setting):

| Operation | Required Permission |
|-----------|-------------------|
| Create wallet, modify wallet name | "管理团队钱包" (Manage Team Wallets) |
| Add Web3 custom network | "管理团队钱包" (production team only) |
| Initiate transaction, cancel transaction | "发起交易" (Initiate Transaction) |
| Manual webhook trigger in Web Console | "发起交易" (Initiate Transaction) |
| Modify API Key, Webhook, confirmation count | "API 管理" (API Management) |
| Download Co-Signer CLI and deployment docs | "API 管理" (API Management) |
| Manage policy engine (add/edit/delete/sort strategies) | "管理交易策略" (Manage Transaction Policies) |
| Auto Sweep policy management | "管理交易策略" (Manage Transaction Policies) |
| Whitelist / Connect app management | "管理白名单/Connect 应用" |
| Enable/edit custom approval flow | Owner and Admin only |
| View audit logs | Admin and Owner only |
| AML Checker | No special permission required |
| Enable gas refill switch | "钱包管理" (Wallet Management) |
| Prepay energy rental, configure alert threshold | "发起交易" (Initiate Transaction) |
| Full scan (全量扫描) button | "管理交易策略" (Manage Transaction Policies) |
| Enter Co-Signer application | No permission required |
| Add approval node participants | "成员管理" (Member Management) |

---

## Policy & Strategy

**Q: What is the maximum number of whitelists supported?**

A: Default maximum is **800 entries**. This can be increased internally by Safeheron Support (requires SQL change).

---

**Q: "Strategy range cannot overlap" error when configuring policies**

A: Policy amount ranges must be contiguous (no gaps, no overlaps). Example of correct configuration:
- Range 1: [0, 1,000) → Approval flow A
- Range 2: [1,000, 10,000) → Approval flow B
- Range 3: [10,000, ∞) → Approval flow C

Incorrect: Having `[0, 100,000,000)` and `[10,000,000, ∞)` — they overlap at 10,000,000.

---

**Q: Transaction records, audit logs — how far back can you query?**

A:
- Transaction records, energy rental records, prepay records: default **3 months**, max span **1 year**
- Audit logs: max **3 months**

---

**Q: Maximum number of policies supported per team?**

A: Default teams support **max 10 policies**. Special requests can be accommodated up to **30 policies** per team by contacting Safeheron Support.

---

## Web Console & App

**Q: How to find accountKey and coinKey in Web Console?**

A:
- **accountKey**: Web Console → Wallets → search wallet name → Copy accountKey
- **coinKey**: Web Console → any Wallet → hover mouse over the coin → coinKey appears in tooltip

---

**Q: Auto Sweep is not executing automatically. How to troubleshoot?**

A: Check these four conditions:
1. Target wallets have the **DEPOSIT** label set.
2. The Auto Sweep policy is **active**, and incoming transactions meet the configured threshold conditions.
3. If wallets already had token balances **before** the policy was configured, re-deposit or trigger a full scan.
4. Check if network **gas is too low** to cover the sweep transaction.

---


**Q: Can wallets be deleted?**

A: **No**. Wallets cannot be deleted. You can rename, hide (`hiddenOnUI=true`), or archive a wallet.

---

**Q: APP shows max how many wallets? How are they sorted?**

A: Max **1,000 wallets** displayed. Sorted by USD value descending. Wallets with no balance appear at the bottom. After 1,000 wallets, the first 1,000 by creation time are shown (not sorted by value then truncated).

---

**Q: How is manual webhook trigger rate limited?**

A: Manual webhook trigger rate limit: **10 times per minute**.

---

## Security

**Q: How should user permissions be assigned? (Principle of least privilege)**

A: Permission assignment must follow these principles:
1. **Least privilege** — only enable the permissions actually required for the current business role. Do not grant all permissions by default.
2. **Separation of duties** — the person who initiates a transaction and the person who approves it must be different accounts. Never allow self-initiate + self-approve.
3. **Periodic review** — regularly audit permission assignments. Revoke permissions for users who have left the team or changed roles to avoid stale permission risk.

---

**Q: What are the security requirements for Webhook integration?**

A:
1. **HTTPS is mandatory** — never expose an HTTP Webhook endpoint in production. All Webhook URLs must use HTTPS.
2. **IP whitelist via VPC security group or firewall** — only allow inbound traffic from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`. Block all other sources.
3. **Verify signature before processing** — validate the `sig` field in every Webhook payload using Safeheron's RSA public key before taking any action. Reject events with invalid signatures.
4. **Idempotent handler** — Safeheron retries up to 7 times; ensure duplicate events do not cause double-crediting or double-processing.
5. **Terminal state wins** — never downgrade a transaction status (e.g., `COMPLETED` → `CONFIRMING`). Out-of-order delivery is possible; if a terminal-state event was already applied, discard any later intermediate-state event for the same transaction.

---

**Q: What are the security requirements for Co-Signer deployment?**

A:
1. **Private network only** — the Co-Signer host must be deployed in an internal isolated network. It must not be directly exposed to the public internet.
2. **Approval Callback Service** — the callback service should only accept inbound traffic from the Co-Signer host IP, not from the public internet.
3. **Production API Key must configure Callback** — for any API Key used with Co-Signer in production, the Callback URL is **required**. Never skip Callback configuration in production.
4. **Never blindly approve** — the callback service must validate every transaction: `customerRefId` must match a real pending business order; amount must match; destination address must match. Reject any discrepancy.

---

**Q: What should I be careful about with API Key security?**

A:
1. **Minimum permissions** — configure each API Key with only the permissions strictly required for its use case. Do not enable all permissions.
2. **No plaintext private keys in code or config** — RSA private keys must never be written into source code or configuration files. Non-essential personnel must not have access to these keys.
3. **Minimal IP whitelist** — only add the server IPs that genuinely need API access. Remove stale or decommissioned IPs promptly.
4. **Production Co-Signer API Key must have Callback configured** — this is mandatory in production, not optional.
5. **Rotate keys on suspected compromise** — if a key may have been exposed, rotate it immediately.

---

**Q: What are the security best practices for configuring transaction policies?**

A:
1. **Tiered approval based on amount** — configure different approval flows for different amount ranges. Example: small amounts → Co-Signer auto-approve; large amounts → multi-person team approval.
2. **Catch-all blocking rule at the bottom** — always add a "block all" rule as the last entry in the policy stack to intercept any transactions that match no defined rule. This prevents unmatched transactions from slipping through.
3. **Test with small amounts after configuration** — after setting up policies, verify the approval flow works as expected by running a small-amount test transaction.
4. **Periodic security audit** — regularly review and audit policy rules to ensure they remain appropriate as business changes.
5. **Subscribe to `NO_MATCHING_TRANSACTION_POLICY` webhook** — use this event as an operational alert for transactions that hit no policy.

---

**Q: What should I be careful about when using the Safeheron App?**

A:
1. **Use a dedicated device** — avoid installing the Safeheron App on a personal everyday phone. A compromised, infected, or accidentally misused personal device could result in asset loss.
2. **Enable 2FA** — two-factor authentication must be enabled for every account that accesses the Safeheron App.
3. **No VPN on the App device** — VPN apps carry traffic interception risk. Do not install or enable a VPN on the device running the Safeheron App.
4. **Offline backup of private key and mnemonic** — back up offline (handwritten on paper). Never screenshot, never upload to cloud storage, never transmit via any instant messaging tool.
5. **Keep App updated** — regularly check for and install the latest version of the Safeheron App to ensure security patches are applied.
6. **Avoid public Wi-Fi** — do not perform asset operations on public Wi-Fi (airports, cafés, etc.) due to man-in-the-middle attack risk.

---

**Q: How should server permissions be assigned?**

A: Server permissions must be strictly controlled:
1. Apply the **principle of least privilege** — only grant users the minimum permissions required to perform their role. Never grant broad administrative access by default.
2. **Regularly audit and revoke** — periodically review all permission assignments. Immediately revoke access for users who no longer need it (e.g., role change, departure).

---

**Q: What password strength is required?**

A: All accounts and credentials involved in the deployment — including database passwords, cloud service account passwords, and API Key-related credentials — must use **strong randomly generated passwords**. Strong passwords must:
- Contain letters, digits, and special characters
- Avoid birthdates, names, or dictionary words
- Be rotated periodically

Additionally, **enable 2FA/MFA** on all services that support it to add a second layer of defense.

---

**Q: How should sensitive information be stored?**

A: All sensitive information — passwords, private keys, API keys, mnemonics — must be stored using a secure management tool such as a password manager (e.g., **1Password**). Rules:
1. **Never share in plaintext** — do not transmit or share sensitive information via social media, messaging apps, email, or any unencrypted channel.
2. **Never store in plain files** — do not place sensitive values in easily accessible files, `.env` files committed to git, or unencrypted note-taking apps.
3. **Use a secrets manager in production** — use AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault for server-side credential management.
4. **Offline backup for mnemonics and key shards** — handwritten on paper, stored in a physically secure location (e.g., safe). Never photographed, never in cloud storage.

---

## Support Contact Templates

**Q: How to apply for MPC Sign policy?**

A: https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/mpc-sign-policy

**Q: How to apply for Web3 wallet policy?**

A: https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/web3-policy

**Q: How to request admin count change or decision mode change?**

A: Send email to support@safeheron.com, CC all other admins. Email template:
```
Subject: Team Admin Change
Body:
Team ID:
New number of admins: X
New decision mode: X/X
Remove admins (if needed):
```
Required approvals follow the current decision mode threshold (e.g. 3/5 requires 3 admin replies).

**Q: How to reset Google Authenticator (GA)?**

A: https://support.safeheron.com/help-center/product-and-solution/support/what-if-i-forget-google-authentication
