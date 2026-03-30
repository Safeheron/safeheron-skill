# Pre-Launch Security Checklist

This checklist maps directly to Safeheron's official 上线前检查清单. Complete every item before go-live.

---

## 1. Private Key Shard Backup

Safeheron uses 3-of-3 MPC: one local shard (team owner) + two cloud shards (admin A, admin B). **Safeheron never stores the local shard — if it is lost, assets are permanently inaccessible.**

| # | Check Item |
|---|-----------|
| 1 | Team owner has backed up the **local private key shard** (mnemonic handwritten, stored offline in a safe — never on internet-connected devices) |
| 2 | Team owner has designated a **trusted second backup person** for the local shard (along with their Safeheron login email) |
| 3 | Team owner has defined a **disaster recovery plan** for the local shard in case the primary holder is unreachable |
| 4 | Two admins (A and B) have backed up the **cloud private key shards** (same offline storage requirements) |
| 5 | Cloud shard holders have each designated a trusted second backup person |
| 6 | Team has defined a **disaster recovery plan** for cloud shards (Safeheron provides full support for cloud shard recovery) |
| 7 | Team has done a **dry run of the private key recovery flow** using a test team, to become familiar with the process before it is ever needed in an emergency |

---

## 2. Policy Configuration

Policies control the approval workflow for every transaction. A misconfigured policy is one of the most common causes of accidental fund loss.

| # | Check Item |
|---|-----------|
| 1 | All possible transaction scenarios and types are enumerated (user withdrawal, internal transfer, manual outflow, auto-sweep) |
| 2 | Each scenario has a policy rule covering: approval flow, amount limits, approved initiators |
| 3 | All transactions fall within a defined policy — no "unmatched policy" gaps (the `NO_MATCHING_TRANSACTION_POLICY` webhook event is your safety net — subscribe to it) |
| 4 | Policy attack simulation performed: attempt to send to a non-whitelisted address via API, initiate from an unauthorized API Key, exceed policy amount limits — confirm that all are blocked |
| 5 | Policy review with Safeheron CSM completed (Safeheron offers 1:1 video policy review sessions before go-live) |

See `references/{lang}/POLICY_STRATEGY.md` for recommended policy configurations.

---

## 3. Safeheron App Security Settings

All team members with access to the Safeheron App should have the following configured:

| # | Check Item |
|---|-----------|
| 1 | **2FA enabled** — Safeheron App supports Google Authenticator. Enable it for every account. This prevents unauthorized access even if a password is compromised |
| 2 | **Wallet password set** — Use the App wallet password to lock the app itself. The password must contain letters, digits, and special characters. Avoid birthdates, names, or dictionary words. Rotate periodically |

---

## 4. Server-Side Checklist

### 4-1. API Retry Mechanism

| # | Check Item |
|---|-----------|
| 1 | All API calls that create transactions use a pre-generated `customerRefId` (stored to DB **before** calling Safeheron) |
| 2 | On network timeout or non-structured Safeheron response, the system **retries with the same `customerRefId`** — this is idempotent and prevents duplicate transactions |
| 3 | Error `9001` (duplicate `customerRefId`) is handled as "transaction already exists" — query by `customerRefId` instead of creating a new one |

### 4-2. Webhook Compensation (Fallback)

| # | Check Item |
|---|-----------|
| 1 | The polling job deduplicates against already-processed transactions (idempotent by `txKey`) |
| 2 | Webhook handler returns HTTP 200 immediately and processes events asynchronously |
| 3 | Webhook handler avoids **status rollback** — if the current DB status is `COMPLETED`, it must not downgrade to `CONFIRMING` even if that event arrives late |
| 4 | `/v1/transactions/one` is called periodically to re-request delivery of failed events |

### 4-3. Safeheron IP Whitelist for Webhook Server

| # | Check Item |
|---|-----------|
| 1 | Webhook server only accepts requests from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182` |
| 2 | Approval Callback Service only accepts requests from the **API Co-Signer host IP** (not from the public internet) |
| 3 | Webhook **signature is verified** before any payload is processed (verify `sig` field using Safeheron's RSA public key) |

### 4-4. Server Least Privilege (Principle of Least Privilege)

| # | Check Item |
|---|-----------|
| 1 | API Keys are created with only the **minimum required permissions** for each use case (e.g., an API Key for initiating withdrawals does not also have account management permission) |
| 2 | Access to server credentials, private keys, and Safeheron Console is restricted to authorized personnel only |
| 3 | Permissions are reviewed periodically — revoked for users who no longer need them |

### 4-5. Server Network Isolation

| # | Check Item |
|---|-----------|
| 1 | API servers are only accessible via internal network / VPN — not exposed directly to the public internet |
| 2 | Firewall rules are configured to block inbound traffic not originating from authorized sources |
| 3 | Outbound connections are restricted to only the necessary endpoints |

---

## 5. Disaster Drill

Safeheron recommends practicing the following emergency scenarios **before go-live**, using a test team:

| # | Scenario |
|---|---------|
| 1 | **Team owner switches device / uninstalls and reinstalls App** — verify local shard recovery works |
| 2 | **Admin switches device / uninstalls and reinstalls App** — verify cloud shard recovery works |
| 3 | **Member switches device** — verify re-enrollment works |

These drills ensure the team knows exactly what to do when an unexpected device change or loss occurs in production.

---

## Summary: Critical Non-Negotiables

These items are the most consequential — do not go live without them:

- Local private key shard backed up offline by the team owner
- At least 2 distinct **human** approvers required for large outflows — API Co-Signer counts as one automated approver (1-of-1 is acceptable for automated low-value tiers only); never use 1-of-1 human-only approval on high-value policies
- `NO_MATCHING_TRANSACTION_POLICY` webhook subscribed and alerted
- Webhook signature verification implemented
- `customerRefId` generated and stored before every API transaction call
- Webhook server IP-whitelisted to only Safeheron's egress IPs
