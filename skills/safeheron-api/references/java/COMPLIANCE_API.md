# Compliance (KYT / AML) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / AML (Anti-Money Laundering) providers to assess transaction risk.

Two main integration points:
1. **ComplianceApiService** — retrieve KYT risk reports for completed transactions.
2. **ToolsApiService (AML Checker)** — proactively assess a blockchain address before transacting. See [TOOLS_API.md](TOOLS_API.md).

---

## Imports

```java
import com.safeheron.client.api.ComplianceApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.KytReportRequest;
import com.safeheron.client.response.KytReportResponse;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
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
// Optional: req.setCustomerRefId("your-ref-id");

KytReportResponse resp = ServiceExecutor.execute(complianceApi.kytReport(req));

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

> *Either `txKey` or `customerRefId` must be provided.

### KytReportResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `amlList` | List\<Aml\> | List of AML assessment results |

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `provider` | String | AML provider name (e.g. "Elliptic", "Chainalysis") |
| `timestamp` | String | Assessment timestamp (milliseconds) |
| `status` | String | Assessment status |
| `riskLevel` | String | Risk level: `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `lastUpdateTime` | String | Last update timestamp |
| `payload` | Object | Raw provider-specific data |

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

## Best Practices

- For address-level risk checks *before* initiating a transaction, use the Tools AML Checker — see [TOOLS_API.md](TOOLS_API.md).
- Safeheron's server timezone is UTC+0.
- The `failOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
