# Compliance (KYT / AML) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / AML (Anti-Money Laundering) providers to assess transaction risk.

Two main integration points:
1. **ComplianceApi** -- retrieve KYT risk reports for completed transactions.
2. **ToolsApi (AML Checker)** -- proactively assess a blockchain address before transacting. See [TOOLS_API.md](TOOLS_API.md).

---

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
complianceApi := api.ComplianceApi{Client: sc}
```

---

## Retrieve KYT Risk Report for a Transaction

After a transaction completes, retrieve its AML/KYT risk assessment:

```go
req := api.KytReportRequest{
    TxKey: "your-tx-key",
    // OR: CustomerRefId: "your-ref-id",
}

var resp api.KytReportResponse
if err := complianceApi.KytReport(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get KYT report: %w", err))
}

for _, aml := range resp.AmlList {
    fmt.Printf("Provider: %s, Risk Level: %s, Status: %s\n",
        aml.Provider, aml.RiskLevel, aml.Status)
}
```

### KytReportRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `TxKey` | Yes* | `string` | Safeheron transaction key |
| `CustomerRefId` | Yes* | `string` | Your reference ID (max 100 chars) |

> *Either `TxKey` or `CustomerRefId` must be provided.

### Aml Fields

| Field | Type | Description |
|-------|------|-------------|
| `Provider` | `string` | AML provider name (e.g. "Elliptic", "Chainalysis") |
| `Timestamp` | `string` | Assessment timestamp (milliseconds) |
| `Status` | `string` | Assessment status |
| `RiskLevel` | `string` | Risk level: `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `LastUpdateTime` | `string` | Last update timestamp |
| `Payload` | `interface{}` | Raw provider-specific data |

---

## AML Lock Behavior

When `FailOnAml=true` (default) in `CreateTransactionsRequest`, Safeheron will:
1. Automatically run AML checks on the destination address.
2. Block the transaction if the address scores high-risk (`AmlLock = "YES"`).

To disable AML blocking (not recommended for production):

```go
req := api.CreateTransactionsRequest{
    FailOnAml: false, // disable AML lock -- use with caution
    // ... other fields
}
```

---

## AML Risk Assessment in Transaction Response

The transaction response (from list/query transaction APIs) includes inline AML data:

```go
var tx api.OneTransactionsResponse
if err := transactionApi.OneTransactions(oneReq, &tx); err != nil {
    panic(err)
}

for _, aml := range tx.AmlList {
    fmt.Printf("%s risk: %s\n", aml.Provider, aml.RiskLevel)
}
fmt.Printf("AML locked: %s\n", tx.AmlLock) // "YES" or "NO"
```

---

## Best Practices

- For address-level risk checks *before* initiating a transaction, use the Tools AML Checker -- see [TOOLS_API.md](TOOLS_API.md).
- Safeheron's server timezone is UTC+0.
- The `FailOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
