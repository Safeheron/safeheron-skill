# Compliance (KYT / KYA) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / KYA (Know Your Address) AML providers to assess transaction and address risk.

Two main integration points:
1. **KYT** — retrieve AML risk reports for completed transactions (`kytReport`).
2. **KYA** — proactively screen a blockchain address before transacting (`kyaScreeningCreate` / `kyaScreeningOne` / `kyaScreeningOrderOne`).

---

## Imports

```java
import com.safeheron.client.api.ComplianceApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.KytReportRequest;
import com.safeheron.client.request.KyaScreeningRequest;
import com.safeheron.client.request.KyaScreeningOneRequest;
import com.safeheron.client.request.KyaScreeningOrderOneRequest;
import com.safeheron.client.response.KytReportResponse;
import com.safeheron.client.response.KyaScreeningCreateResponse;
import com.safeheron.client.response.KyaScreeningOneResponse;
import com.safeheron.client.response.KyaScreeningOrderOneResponse;
import com.safeheron.client.response.KyaSupportedNetworksResponse;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
import com.safeheron.client.request.Aml;
import java.util.List;
```

## Create API Service

```java
ComplianceApiService complianceApi = ServiceCreator.create(ComplianceApiService.class, safeheronConfig);
```

---

## Retrieve KYT Risk Report for a Transaction

After a transaction completes, retrieve its AML/KYT risk assessment:

```java
KytReportRequest req = new KytReportRequest();
req.setTxKey("your-tx-key");
// OR: req.setCustomerRefId("your-ref-id");

KytReportResponse resp = ServiceExecutor.execute(complianceApi.kytReport(req));

System.out.println("Screening state: " + resp.getAmlScreeningTriggeredState());
for (KytReportResponse.Aml aml : resp.getAmlList()) {
    System.out.println("Provider: " + aml.getProvider());
    System.out.println("Risk Level: " + aml.getRiskLevel());
    System.out.println("Status: " + aml.getStatus());
}
```

### KytReportRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `txKey` | Yes* | String | Safeheron transaction key |
| `customerRefId` | Yes* | String | Your reference ID (max 100 chars) |

> *Either `txKey` or `customerRefId` must be provided. If both are provided, `txKey` takes precedence.

### KytReportResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `amlScreeningTriggeredState` | String | `IN_PROGRESS` / `TRIGGERED` / `UNTRIGGERED` — see below |
| `amlList` | List\<Aml\> | List of AML assessment results (empty when `UNTRIGGERED`) |

**`amlScreeningTriggeredState` values:**

| Value | Meaning |
|-------|---------|
| `IN_PROGRESS` | Evaluating — screening not yet determined; `amlList` unavailable, poll again |
| `TRIGGERED` | Screening initiated; check `amlList` for risk results |
| `UNTRIGGERED` | Screening not triggered; `amlList` is empty |

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | String | AML provider name: `Elliptic`, `Chainalysis`, `MistTrack` |
| `timestamp` | String | Screening creation time (UNIX ms timestamp) |
| `status` | String | `PENDING` / `COMPLETED` / `SKIPPED` / `FAILED` |
| `riskLevel` | String | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `lastUpdateTime` | String | Last update time (UNIX ms timestamp) |
| `payload` | Object | Raw provider-specific report (structure varies by provider) |

---

## Create KYA Screening Request

Initiate an address screening against one or more AML providers in parallel:

```java
KyaScreeningRequest req = new KyaScreeningRequest();
req.setAddress("0xRecipientAddress");
req.setChainType("ETH");
// req.setNetwork("ethereum");  // required when providers includes MistTrack
req.setProviders(Arrays.asList("Elliptic", "Chainalysis"));

KyaScreeningCreateResponse resp = ServiceExecutor.execute(complianceApi.kyaScreeningCreate(req));

System.out.println("Screen ID: " + resp.getScreenId());
for (KyaScreeningCreateResponse.Order order : resp.getOrders()) {
    System.out.println("Order ID: " + order.getScreenOrderId() + ", Provider: " + order.getProvider());
}
```

### KyaScreeningRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `address` | Yes | String | On-chain address to screen (not limited to Safeheron addresses) |
| `chainType` | Yes | String | Chain type — see `kyaSupportedNetworks()` for valid values |
| `network` | Conditional | String | Network identifier — required when `providers` includes `MistTrack` |
| `providers` | Yes | List\<String\> | At least one of: `MistTrack`, `Elliptic`, `Chainalysis`; no duplicates |

### KyaScreeningCreateResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | String | Screening request ID (use for polling) |
| `address` | String | Screened address |
| `chainType` | String | Chain type |
| `network` | String | Network identifier |
| `orders` | List\<Order\> | One order per provider |
| `createTime` | Long | Creation time (UNIX ms timestamp) |

**Order fields:** `screenOrderId` (String), `provider` (String)

---

## Retrieve KYA Screening Summary

Poll until `status` is `FINISHED`:

```java
KyaScreeningOneRequest req = new KyaScreeningOneRequest();
req.setScreenId("your-screen-id");

KyaScreeningOneResponse resp = ServiceExecutor.execute(complianceApi.kyaScreeningOne(req));

System.out.println("Overall status: " + resp.getStatus()); // PROCESSING / FINISHED
for (KyaScreeningOneResponse.Order order : resp.getOrders()) {
    System.out.printf("  %s: %s (risk=%s)%n",
        order.getProvider(), order.getStatus(), order.getRiskLevel());
}
```

### KyaScreeningOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenId` | String | Screening request ID |
| `address` | String | Screened address |
| `chainType` | String | Chain type |
| `network` | String | Network identifier |
| `status` | String | `PROCESSING` (some orders pending) / `FINISHED` (all completed) |
| `createTime` | Long | Creation time (UNIX ms timestamp) |
| `orders` | List\<Order\> | Per-provider order summaries |

**Order summary fields:**

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | String | Screening order ID |
| `provider` | String | Provider identifier |
| `status` | String | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | String | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | Long | Completion time (UNIX ms timestamp); only set in terminal state |

---

## Retrieve KYA Screening Order Details

Get the full provider report for a single order (includes raw `payload`):

```java
KyaScreeningOrderOneRequest req = new KyaScreeningOrderOneRequest();
req.setScreenOrderId("your-order-id");

KyaScreeningOrderOneResponse resp = ServiceExecutor.execute(complianceApi.kyaScreeningOrderOne(req));

System.out.println("Risk Level: " + resp.getRiskLevel());
System.out.println("Address Type: " + resp.getAddressType());
System.out.println("Payload: " + resp.getPayload());
```

### KyaScreeningOrderOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `screenOrderId` | String | Screening order ID |
| `screenId` | String | Parent screening request ID |
| `provider` | String | Provider identifier |
| `address` | String | Screened address |
| `addressType` | String | `VAULT_ACCOUNT` / `WHITELISTING_ACCOUNT` / `ONE_TIME_ADDRESS` |
| `chainType` | String | Chain type |
| `network` | String | Network identifier |
| `status` | String | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `riskLevel` | String | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `completedAt` | Long | Completion time (UNIX ms timestamp) |
| `createTime` | Long | Screening initiation time (UNIX ms timestamp) |
| `payload` | Object | Provider's raw report (available after completion; structure varies by provider) |

---

## Retrieve Supported Networks & Providers

Query which chain/network combinations are supported by each AML provider:

```java
List<KyaSupportedNetworksResponse> networks = ServiceExecutor.execute(complianceApi.kyaSupportedNetworks());

for (KyaSupportedNetworksResponse n : networks) {
    System.out.printf("Network: %s, ChainType: %s, Providers: %s%n",
        n.getNetwork(), n.getChainType(), n.getProviders());
}
```

### KyaSupportedNetworksResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `network` | String | Network identifier |
| `chainType` | String | Chain type |
| `providers` | List\<String\> | Providers supporting this network |

---

## AML Lock Behavior

When `failOnAml=true` (default) in `CreateTransactionRequest`, Safeheron will:
1. Automatically run AML checks on the destination address.
2. Block the transaction if the address scores high-risk (`amlLock = "YES"`).

To disable AML blocking (not recommended for production):
```java
CreateTransactionRequest req = new CreateTransactionRequest();
req.setFailOnAml(false);  // disable AML lock — use with caution
```

The `TransactionsResponse.amlLock` field indicates `"YES"` or `"NO"`.

---

## AML Risk Assessment in Transaction Response

The `TransactionsResponse` (from list/query transaction APIs) includes inline AML data:

```java
OneTransactionsResponse tx = ServiceExecutor.execute(transactionApi.oneTransactions(req));
List<Aml> amlList = tx.getAmlList();
for (Aml aml : amlList) {
    System.out.println(aml.getProvider() + " risk: " + aml.getRiskLevel());
}
System.out.println("AML locked: " + tx.getAmlLock()); // "YES" or "NO"
```

---

## Full Example: Screen Address Before Sending

Use KYA screening to check a destination address before initiating any outbound transfer:

```java
// Step 1: Create KYA screening
KyaScreeningRequest screenReq = new KyaScreeningRequest();
screenReq.setAddress(destinationAddress);
screenReq.setChainType("ETH");
screenReq.setProviders(Arrays.asList("Chainalysis"));
KyaScreeningCreateResponse created = ServiceExecutor.execute(complianceApi.kyaScreeningCreate(screenReq));

// Step 2: Poll until FINISHED
KyaScreeningOneRequest pollReq = new KyaScreeningOneRequest();
pollReq.setScreenId(created.getScreenId());
KyaScreeningOneResponse result = null;
for (int i = 0; i < 30; i++) {
    result = ServiceExecutor.execute(complianceApi.kyaScreeningOne(pollReq));
    if ("FINISHED".equals(result.getStatus())) break;
    Thread.sleep(2000);
}

// Step 3: Block high-risk addresses
for (KyaScreeningOneResponse.Order order : result.getOrders()) {
    if ("HIGH".equals(order.getRiskLevel()) || "SEVERE".equals(order.getRiskLevel())) {
        throw new SecurityException("Address is " + order.getRiskLevel() + " risk: " + destinationAddress);
    }
}

// Step 4: Proceed with transaction
CreateTransactionRequest txReq = new CreateTransactionRequest();
txReq.setDestinationAddress(destinationAddress);
// ... set other fields
```

---

## Best Practices

- Use KYA screening (`kyaScreeningCreate`) to check an address **before** initiating a transaction. Poll `kyaScreeningOne` until `status=FINISHED`.
- Call `kyaSupportedNetworks()` to determine valid `chainType` / `network` / `provider` combinations before submitting a screening.
- `network` is required when `providers` includes `MistTrack`.
- The `failOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
- Safeheron's server timezone is UTC+0.
