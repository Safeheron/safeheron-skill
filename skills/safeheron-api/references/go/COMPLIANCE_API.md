# Compliance (KYT / KYA) API Reference

## Overview

Safeheron integrates with KYT (Know Your Transaction) / KYA (Know Your Address) AML providers to assess transaction and address risk.

Two main integration points:
1. **KYT** — retrieve AML risk reports for completed transactions (`KytReport`).
2. **KYA** — proactively screen a blockchain address before transacting (`KyaScreeningCreate` / `KyaScreeningOne` / `KyaScreeningOrderOne`).

---

## Imports

```go
import (
    "fmt"
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

fmt.Println("Screening state:", resp.AmlScreeningTriggeredState)
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

> *Either `TxKey` or `CustomerRefId` must be provided. If both are provided, `TxKey` takes precedence.

### KytReportResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `TxKey` | `string` | Safeheron transaction key |
| `CustomerRefId` | `string` | Your reference ID |
| `AmlScreeningTriggeredState` | `string` | `IN_PROGRESS` / `TRIGGERED` / `UNTRIGGERED` — see below |
| `AmlList` | `[]AmlReport` | List of AML assessment results (empty when `UNTRIGGERED`) |

**`AmlScreeningTriggeredState` values:**

| Value | Meaning |
|-------|---------|
| `IN_PROGRESS` | Evaluating — screening not yet determined; `AmlList` unavailable, poll again |
| `TRIGGERED` | Screening initiated; check `AmlList` for risk results |
| `UNTRIGGERED` | Screening not triggered; `AmlList` is empty |

### AmlReport Fields

| Field | Type | Description |
|-------|------|-------------|
| `Provider` | `string` | AML provider name: `Elliptic`, `Chainalysis`, `MistTrack` |
| `Timestamp` | `string` | Screening creation time (UNIX ms timestamp) |
| `Status` | `string` | `PENDING` / `COMPLETED` / `SKIPPED` / `FAILED` |
| `RiskLevel` | `string` | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `LastUpdateTime` | `string` | Last update time (UNIX ms timestamp) |
| `Payload` | `any` | Raw provider-specific report (structure varies by provider) |

---

## Create KYA Screening Request

Initiate an address screening against one or more AML providers in parallel:

```go
req := api.KyaScreeningRequest{
    Address:   "0xRecipientAddress",
    ChainType: "ETH",
    // Network: "ethereum",  // required when Providers includes MistTrack
    Providers: []string{"Elliptic", "Chainalysis"},
}

var resp api.KyaScreeningCreateResponse
if err := complianceApi.KyaScreeningCreate(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create KYA screening: %w", err))
}

fmt.Println("Screen ID:", resp.ScreenId)
for _, order := range resp.Orders {
    fmt.Printf("Order ID: %s, Provider: %s\n", order.ScreenOrderId, order.Provider)
}
```

### KyaScreeningRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `Address` | Yes | `string` | On-chain address to screen (not limited to Safeheron addresses) |
| `ChainType` | Yes | `string` | Chain type — see `KyaSupportedNetworks()` for valid values |
| `Network` | Conditional | `string` | Network identifier — required when `Providers` includes `MistTrack` |
| `Providers` | Yes | `[]string` | At least one of: `MistTrack`, `Elliptic`, `Chainalysis`; no duplicates |

### KyaScreeningCreateResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `ScreenId` | `string` | Screening request ID (use for polling) |
| `Address` | `string` | Screened address |
| `ChainType` | `string` | Chain type |
| `Network` | `string` | Network identifier |
| `Orders` | `[]KyaScreeningOrder` | One order per provider |
| `CreateTime` | `int64` | Creation time (UNIX ms timestamp) |

**KyaScreeningOrder fields:** `ScreenOrderId` (string), `Provider` (string)

---

## Retrieve KYA Screening Summary

Poll until `Status` is `FINISHED`:

```go
req := api.KyaScreeningOneRequest{
    ScreenId: "your-screen-id",
}

var resp api.KyaScreeningOneResponse
if err := complianceApi.KyaScreeningOne(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get KYA screening: %w", err))
}

fmt.Println("Overall status:", resp.Status) // PROCESSING / FINISHED
for _, order := range resp.Orders {
    fmt.Printf("  %s: %s (risk=%s)\n", order.Provider, order.Status, order.RiskLevel)
}
```

### KyaScreeningOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `ScreenId` | `string` | Screening request ID |
| `Address` | `string` | Screened address |
| `ChainType` | `string` | Chain type |
| `Network` | `string` | Network identifier |
| `Status` | `string` | `PROCESSING` (some orders pending) / `FINISHED` (all completed) |
| `CreateTime` | `int64` | Creation time (UNIX ms timestamp) |
| `Orders` | `[]KyaScreeningOrderSummary` | Per-provider order summaries |

**KyaScreeningOrderSummary fields:**

| Field | Type | Description |
|-------|------|-------------|
| `ScreenOrderId` | `string` | Screening order ID |
| `Provider` | `string` | Provider identifier |
| `Status` | `string` | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `RiskLevel` | `string` | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `CompletedAt` | `int64` | Completion time (UNIX ms timestamp); only set in terminal state |

---

## Retrieve KYA Screening Order Details

Get the full provider report for a single order (includes raw `Payload`):

```go
req := api.KyaScreeningOrderOneRequest{
    ScreenOrderId: "your-order-id",
}

var resp api.KyaScreeningOrderOneResponse
if err := complianceApi.KyaScreeningOrderOne(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get order detail: %w", err))
}

fmt.Println("Risk Level:", resp.RiskLevel)
fmt.Println("Address Type:", resp.AddressType)
fmt.Println("Payload:", resp.Payload)
```

### KyaScreeningOrderOneResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `ScreenOrderId` | `string` | Screening order ID |
| `ScreenId` | `string` | Parent screening request ID |
| `Provider` | `string` | Provider identifier |
| `Address` | `string` | Screened address |
| `AddressType` | `string` | `VAULT_ACCOUNT` / `WHITELISTING_ACCOUNT` / `ONE_TIME_ADDRESS` |
| `ChainType` | `string` | Chain type |
| `Network` | `string` | Network identifier |
| `Status` | `string` | `PENDING` / `COMPLETED` / `FAILED` / `SKIPPED` |
| `RiskLevel` | `string` | `LOW` / `MEDIUM` / `HIGH` / `SEVERE` / `UNKNOWN` |
| `CompletedAt` | `int64` | Completion time (UNIX ms timestamp) |
| `CreateTime` | `int64` | Screening initiation time (UNIX ms timestamp) |
| `Payload` | `any` | Provider's raw report (available after completion; structure varies by provider) |

---

## Retrieve Supported Networks & Providers

Query which chain/network combinations are supported by each AML provider:

```go
var networks api.KyaSupportedNetworksResponseList
if err := complianceApi.KyaSupportedNetworks(&networks); err != nil {
    panic(fmt.Errorf("failed to get supported networks: %w", err))
}

for _, n := range networks {
    fmt.Printf("Network: %s, ChainType: %s, Providers: %v\n",
        n.Network, n.ChainType, n.Providers)
}
```

### KyaSupportedNetworksResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `Network` | `string` | Network identifier |
| `ChainType` | `string` | Chain type |
| `Providers` | `[]string` | Providers supporting this network |

---

## AML Lock Behavior

When `FailOnAml=true` (default) in `CreateTransactionsRequest`, Safeheron will:
1. Automatically run AML checks on the destination address.
2. Block the transaction if the address scores high-risk (`AmlLock = "YES"`).

To disable AML blocking (not recommended for production):

```go
b := false
req := api.CreateTransactionsRequest{
    FailOnAml: &b, // disable AML lock -- use with caution
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

## Full Example: Screen Address Before Sending

Use KYA screening to check a destination address before initiating any outbound transfer:

```go
// Step 1: Create KYA screening
createReq := api.KyaScreeningRequest{
    Address:   destinationAddress,
    ChainType: "ETH",
    Providers: []string{"Chainalysis"},
}
var created api.KyaScreeningCreateResponse
if err := complianceApi.KyaScreeningCreate(createReq, &created); err != nil {
    panic(fmt.Errorf("KYA screening create failed: %w", err))
}

// Step 2: Poll until FINISHED
pollReq := api.KyaScreeningOneRequest{ScreenId: created.ScreenId}
var result api.KyaScreeningOneResponse
for i := 0; i < 30; i++ {
    if err := complianceApi.KyaScreeningOne(pollReq, &result); err != nil {
        panic(err)
    }
    if result.Status == "FINISHED" {
        break
    }
    time.Sleep(2 * time.Second)
}

// Step 3: Handle each riskLevel according to your own business logic
for _, order := range result.Orders {
    riskLevel := order.RiskLevel
    // Decide what to do based on riskLevel (LOW / MEDIUM / HIGH / SEVERE / UNKNOWN).
    // For example, block HIGH/SEVERE, require manual review for MEDIUM, allow LOW, etc.
    // Implement the subsequent business logic (block, alert, proceed with transaction, ...) yourself.
    _ = riskLevel
}
```

---

## Best Practices

- Use KYA screening (`KyaScreeningCreate`) to check an address **before** initiating a transaction. Poll `KyaScreeningOne` until `Status=FINISHED`.
- Call `KyaSupportedNetworks()` to determine valid `ChainType` / `Network` / `Provider` combinations before submitting a screening.
- `Network` is required when `Providers` includes `MistTrack`.
- The `FailOnAml` flag defaults to `true`. Setting it to `false` bypasses AML blocking entirely.
- Safeheron's server timezone is UTC+0.
