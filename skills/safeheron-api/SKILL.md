---
name: safeheron-api
description: >
  Use when working with Safeheron API to set up from scratch, manage wallet accounts,
  create transactions, handle MPC signing, interact with Web3 wallet operations,
  manage whitelists, process webhooks, use Gas Station, perform AML/compliance checks,
  use Tools API, or handle API Co-Signer callbacks — in Java, JavaScript/TypeScript,
  Go, or Python.
---

# Safeheron API Skill

A complete reference and integration guide for the Safeheron API and SDKs.

**When NOT to use:** For generic blockchain/Web3 questions unrelated to Safeheron's API or SDK; for other MPC custody providers.

## 🚀 New Here? Start with the Getting Started Guide

**→ `references/{lang}/GETTING_STARTED.md`** (replace `{lang}` with `java`, `js`, `go`, or `python`)

Covers all 5 onboarding steps:
1. Generate RSA key pair
2. Configure API account in Safeheron Console
3. Add SDK to project dependencies
4. Inject credentials
5. First API request

---

## 🔐 Security-First: Mandatory Checklist Before Writing Any Code

**These security principles are non-negotiable and must be reflected in all generated code. Use TaskCreate to create one task per item below before writing ANY code — including bulk "generate all features" requests. Items that are N/A must state a reason; they cannot be silently omitted. When user requests involve credentials, transfers, Webhooks, Co-Signer, or deployment, proactively explain the applicable requirements without waiting to be asked.**

### Credential & Key Management
- [ ] **Credential injection** — Always load API keys, RSA private keys, and platform public keys from a secrets manager. Never hardcode them in source code or config files.
- [ ] **Use the right secrets manager for the deployment target** — AWS KMS or GCP KMS for cloud deployments; HashiCorp Vault for self-hosted; a file outside the project directory (never committed to git) for local development only.

### Transfer & Whitelist Security
- [ ] **`customerRefId` first** — Generate and persist `customerRefId` to your DB **before** calling any Safeheron create API. On timeout, retry with the same ID. Error `9001` = already exists, query instead of recreating.
- [ ] **Whitelist addresses for formal transfers** — `ONE_TIME_ADDRESS` must only be used for genuinely temporary, one-off payments. All recurring or formal transfers (exchange hot wallets, partner addresses, internal accounts) require a whitelisted address.
- [ ] **AML check before every transfer** — Call the Tools API to screen the destination address before creating any outbound transaction. Intercept or alert on high-risk addresses.
- [ ] **Validate address format** — Call the coin address validation endpoint before adding an address to the whitelist or initiating a transfer. Never trust user-supplied address strings directly.
- [ ] **Amounts: String in API, decimal type in code** — Never use `float` or `double` for monetary values. API request fields use `String`; application-side calculations use an arbitrary-precision decimal type (Java: `BigDecimal`; Python: `Decimal`; JS/TS: keep as `string` or use `decimal.js`; Go: `string` or `shopspring/decimal`).
- [ ] **`failOnAml: true` by default** — Only disable when the business case is explicitly confirmed.
- [ ] **`failOnContract: true` by default** — Block contract address destinations unless explicitly required.

### Co-Signer Approval Callback
- [ ] **No blind approval** — The Co-Signer callback service must validate every transaction against the business system before returning `APPROVE`: verify that `customerRefId` matches a real pending order, that the amount matches, and that the destination address matches.
- [ ] **Co-Signer host in private network** — The Co-Signer must be deployed in a private isolated network with no public internet inbound access. The Approval Callback Service must only accept traffic from the Co-Signer host IP.
- [ ] **Production Co-Signer API Key must have Callback configured** — Never skip Callback URL configuration for a Co-Signer API Key in production.

### Webhook Security
- [ ] **Webhook signature verification** — Every webhook payload must have its `sig` verified using Safeheron's RSA public key before any processing. Reject events with invalid signatures immediately.
- [ ] **Webhook must use HTTPS** — Never expose an HTTP webhook endpoint in production.
- [ ] **Webhook handlers must be idempotent** — Safeheron retries up to 7 times. Never double-credit a deposit or double-process a withdrawal on duplicate events.
- [ ] **No status rollback; terminal state wins** — If the current DB status is already `COMPLETED`, `FAILED`,`CANCELLED`, or `REJECTED`, discard any later-arriving event for that transaction — even if it carries an intermediate status (out-of-order delivery is possible).
- [ ] **Webhook IP whitelist** — Only accept webhook traffic from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`
- [ ] **Webhook + REST API polling** — Always implement both: Webhook as primary, REST API polling as fallback. Call `/v1/transactions/one` periodically to re-request failed deliveries.

### Policy & Operational Alerts
- [ ] **Policy: least privilege** — API Keys, approval policies, and server permissions must be scoped to the minimum required.
- [ ] **Catch-all blocking policy** — Add a "block all" rule at the bottom of the policy stack to intercept transactions that match no defined rule.
- [ ] **Subscribe to operational alerts** — Always subscribe to `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`,`GAS_BALANCE_WARNING`, and `AML_KYT_ALERT` webhook events.
- [ ] **Minimum deposit filter** — Filter dust/address-pollution deposits by enforcing a minimum deposit amount in your system.

→ **`references/{lang}/SECURITY_BEST_PRACTICES.md`** — Detailed implementation guide with code examples (read this)
→ **[references/SECURITY_CHECKLIST.md](references/SECURITY_CHECKLIST.md)** — Complete pre-launch checklist
→ **`references/{lang}/POLICY_STRATEGY.md`** — Policy configuration guide with approval tiers
→ **`references/{lang}/BUSINESS_PATTERNS.md`** — Deposit/withdrawal/sweep patterns with security baked in

---

## ⚠️ SDK Initialization Pattern (Per Language)

| Language | Config | Get API instance | Invoke |
|----------|--------|-----------------|--------|
| Java | `SafeheronConfig` builder | `ServiceCreator.create(XxxApiService.class, config)` | `ServiceExecutor.execute(...)` sync |
| JS/TS | config object `{ baseUrl, apiKey, rsaPrivateKey, safeheronRsaPublicKey }` | `new XxxApi(config)` | `async/await` |
| Python | config dict `{ 'apiKey', 'privateKey', 'safeheronPublicKey', 'baseUrl' }` | `XxxApi(config)` | sync method call |
| Go | `safeheron.ApiConfig{ BaseUrl, ApiKey, RsaPrivateKey, SafeheronRsaPublicKey }` | `api.XxxApi{Client: sc}` | sync, returns `error` |

→ Full working examples: `references/{lang}/GETTING_STARTED.md`

---

## Quick Reference

- **API Base URL**: `https://api.safeheron.vip`
- **Auth Scheme**: RSA-4096 (signature) + AES-256-GCM (payload encryption)

- **SDK:**

| Language | Install                                            |
|----------|----------------------------------------------------|
| Java | `com.safeheron:api-sdk-java:1.0.12` (Maven)        |
| JS/TS | `npm install @safeheron/api-sdk`                   |
| Go | `go get github.com/Safeheron/safeheron-api-sdk-go` |
| Python | `pip install safeheron-api-sdk-python`             |

- **SDK Repos**: [Java](https://github.com/Safeheron/safeheron-api-sdk-java) | [JS/TS](https://github.com/Safeheron/safeheron-api-sdk-js) | [Go](https://github.com/Safeheron/safeheron-api-sdk-go) | [Python](https://github.com/Safeheron/safeheron-api-sdk-python)
- **API Docs**: https://docs.safeheron.com/api/en.html
- **Safeheron Egress IPs** (for webhook/callback firewall rules): `18.162.105.64`, `18.167.22.59`, `18.167.21.182`

---

## API Modules

| Module | Description |
|--------|-------------|
| Account | Wallet account CRUD |
| Coin | Coin / address management |
| Transaction | Create / query / speed-up transactions |
| MpcSign | MPC signing (requires Safeheron Support to enable) |
| Web3 | ethSign / personalSign / signTypedData / signTransaction |
| Whitelist | Whitelist address CRUD |
| Compliance | AML/KYT compliance reports |
| Gas | Gas Station status & auto top-up |
| Tools | AML address risk check |
| Webhook | Event subscription & handling |

For language-specific class names and call patterns, see `references/{lang}/WALLET_API.md` etc.

---

## Reference Files

> Replace `{lang}` with `java`, `js`, `go`, or `python`. If the user's language is not clear from context, ask before reading any `{lang}` file.

| Topic | File                                                                     |
|-------|--------------------------------------------------------------------------|
| RSA+AES auth flow (all languages) | [references/AUTH.md](references/AUTH.md)                             |
| Pre-launch security checklist (all languages) | [references/SECURITY_CHECKLIST.md](references/SECURITY_CHECKLIST.md) |
| FAQ (all languages) | [references/FAQ.md](references/FAQ.md)                |
| Getting Started (0 → first API call) | references/{lang}/GETTING_STARTED.md                                 |
| Security best practices (with code examples) | references/{lang}/SECURITY_BEST_PRACTICES.md                         |
| Policy configuration & approval tiers | references/{lang}/POLICY_STRATEGY.md                                 |
| Deposit / withdrawal / sweep patterns | references/{lang}/BUSINESS_PATTERNS.md                               |
| SDK installation & configuration | references/{lang}/SDK_SETUP.md                                           |
| Wallet account CRUD & coin management | references/{lang}/WALLET_API.md                                          |
| Coin list, address validation, balance snapshot | references/{lang}/COIN_API.md                                            |
| Transaction create, query, list, fee, cancel, speed-up | references/{lang}/TRANSACTION_API.md                                     |
| MPC Sign flow | references/{lang}/MPC_SIGN_API.md                                        |
| Web3 signing | references/{lang}/WEB3_API.md                                            |
| Whitelist CRUD | references/{lang}/WHITELIST_API.md                                       |
| Compliance / KYT report | references/{lang}/COMPLIANCE_API.md                                      |
| Gas Station status & auto top-up | references/{lang}/GAS_API.md                                             |
| Tools API — AML address checker | references/{lang}/TOOLS_API.md                                           |
| Webhook events, handler, re-push | references/{lang}/WEBHOOK.md                                             |
| API Co-Signer / Approval Callback + security deployment | references/{lang}/COSIGNER.md                                            |
| Error codes & troubleshooting | references/{lang}/ERROR_CODES.md                                         |

---

## Important Notes

- **IP Whitelisting** is mandatory — register your server IP in Safeheron Console first. API calls from unregistered IPs are rejected.
- **Idempotency** — generate `customerRefId` (UUID) and **save to DB before calling Safeheron**. On timeout, retry with the same ID. Error `9001` = duplicate refId (query existing instead of creating new).
- All monetary amounts are **strings** — never use float/double.
- **Web3 API** requires a Web3 wallet. Using a Vault account key causes "account not found".
- **MPC Sign** requires a special policy — contact Safeheron Support to enable (error `9028` = policy not configured).
- SDK is **backward compatible** — upgrading only adds new methods, never removes old ones.

