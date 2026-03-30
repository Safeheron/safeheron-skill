# Tools API Reference (AML Address Checker)

## Overview

The Tools API provides utilities for address-level AML risk assessment *before* initiating a transaction. This is distinct from transaction-level KYT reports (see [COMPLIANCE_API.md](COMPLIANCE_API.md)).

Use Case: Check whether a recipient address is flagged as high-risk **before** sending funds.

---

## Imports

```go
import (
    "fmt"
    "time"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
toolsApi := api.ToolsApi{Client: sc}
```

---

## AML Address Risk Assessment (Two-Step Async Flow)

Assessment is asynchronous: submit the address, then poll for results.

### Step 1: Submit Address for AML Assessment

```go
req := api.AmlCheckerRequestRequest{
    Network: "Ethereum", // "Bitcoin", "Ethereum", or "Tron"
    Address: "0xAbCd1234...",
}

var resp api.AmlCheckerRequestResponse
if err := toolsApi.AmlCheckerRequest(req, &resp); err != nil {
    panic(fmt.Errorf("failed to submit AML check: %w", err))
}
requestId := resp.RequestId // save this for polling
```

### Step 2: Retrieve AML Assessment Result

```go
pollReq := api.AmlCheckerRetrievesRequest{
    RequestId: requestId,
}

var result api.AmlCheckerRetrievesResponse
for i := 0; i < 30; i++ {
    if err := toolsApi.AmlCheckerRetrieves(pollReq, &result); err != nil {
        panic(fmt.Errorf("failed to poll AML result: %w", err))
    }
    if result.MistTrack.Status == "SUCCESS" {
        break
    }
    time.Sleep(2 * time.Second)
}

fmt.Printf("Risk Level: %s\n", result.MistTrack.RiskLevel)
```

### AmlCheckerRetrievesResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `RequestId` | `string` | The assessment request ID |
| `Network` | `string` | Blockchain network |
| `Address` | `string` | Assessed address |
| `MistTrack.Status` | `string` | Assessment status (EVALUATING / SUCCESS) |
| `MistTrack.RiskLevel` | `string` | `Low`, `Moderate`, `High`, `Severe` |

---

## Full Example: Check Address Before Sending

```go
// 1. Submit address for assessment
submitReq := api.AmlCheckerRequestRequest{
    Network: "Ethereum",
    Address: destinationAddress,
}
var submitResp api.AmlCheckerRequestResponse
if err := toolsApi.AmlCheckerRequest(submitReq, &submitResp); err != nil {
    panic(err)
}

// 2. Poll until result is ready
pollReq := api.AmlCheckerRetrievesRequest{
    RequestId: submitResp.RequestId,
}
var result api.AmlCheckerRetrievesResponse
for i := 0; i < 30; i++ {
    if err := toolsApi.AmlCheckerRetrieves(pollReq, &result); err != nil {
        panic(err)
    }
    if result.MistTrack.Status == "SUCCESS" {
        break
    }
    time.Sleep(2 * time.Second)
}

// 3. Block high-risk addresses
if result.MistTrack.RiskLevel == "High" || result.MistTrack.RiskLevel == "Severe" {
    panic("AML check failed: destination address is high-risk")
}

// 4. Proceed with transaction
req := api.CreateTransactionsRequest{
    DestinationAddress: destinationAddress,
    // ... set other fields
}
```

---

## Supported Networks

| Network Value | Supported Chains |
|---------------|-----------------|
| `Bitcoin` | Bitcoin (BTC) |
| `Ethereum` | Ethereum, ERC-20 tokens, and EVM-compatible chains |
| `Tron` | TRON (TRX), TRC-20 tokens |

---

## Best Practices

- AML checker uses external providers -- results may take several seconds.
- For transaction-level KYT reports (after a transaction completes), use `ComplianceApi.KytReport()` -- see [COMPLIANCE_API.md](COMPLIANCE_API.md).
- The `FailOnAml=true` flag in `CreateTransactionsRequest` (default) enables automatic server-side AML blocking -- the Tools API is for manual pre-screening before sending.
