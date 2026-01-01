# VCP v1.1 Error Event Standardization

**Document ID:** VSO-IMPL-001-ERR  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

VCP v1.1 introduces **standardized error event types** to ensure consistent error logging across implementations. Error events MUST be included in Merkle batches and anchored alongside normal events.

---

## Standard Error Types

| EventType | Category | Severity | Description |
|-----------|----------|----------|-------------|
| `ERR_CONN` | Connection | CRITICAL | Connection failure to broker/exchange |
| `ERR_AUTH` | Authentication | CRITICAL | Authentication or authorization failure |
| `ERR_TIMEOUT` | Timeout | WARNING | Operation timeout |
| `ERR_REJECT` | Rejection | WARNING | Order rejected by broker/exchange |
| `ERR_PARSE` | Data | WARNING | Parse or validation failure |
| `ERR_SYNC` | Synchronization | WARNING | Clock synchronization lost |
| `ERR_RISK` | Risk | CRITICAL | Risk limit breach |
| `ERR_SYSTEM` | System | CRITICAL | Internal system error |
| `ERR_RECOVER` | Recovery | INFO | Recovery initiated or completed |

---

## Severity Levels

| Severity | Description | Action |
|----------|-------------|--------|
| **CRITICAL** | System integrity at risk | Immediate attention required |
| **WARNING** | Degraded operation | Monitor and investigate |
| **INFO** | Informational | No action required |

---

## Schema

### Error Event Structure

```json
{
  "header": {
    "event_id": "01934e3a-7b2c-7f93-8f2a-1234567890ab",
    "event_type": "ERR_RISK",
    "timestamp_iso": "2025-12-30T12:00:00.000Z",
    "timestamp_int": "1735560000000",
    "timestamp_precision": "MILLISECOND",
    "clock_sync_status": "BEST_EFFORT",
    "vcp_version": "1.1",
    "hash_algo": "SHA256"
  },
  "payload": {
    "error_details": {
      "error_code": "MAX_POSITION_EXCEEDED",
      "error_message": "Position 150 lots exceeds limit of 100 lots",
      "severity": "CRITICAL",
      "affected_component": "risk-manager",
      "recovery_action": "Order rejected, position unchanged",
      "correlated_event_id": "01934e3a-1234-7f93-8f2a-abcdef123456"
    }
  },
  "policy_identification": {
    "version": "1.1",
    "policy_id": "com.example:silver-v1",
    "conformance_tier": "SILVER"
  },
  "security": {
    "event_hash": "abc123..."
  }
}
```

### Error Details Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `error_code` | string | ✅ Yes | Machine-readable error code |
| `error_message` | string | ✅ Yes | Human-readable description |
| `severity` | enum | ✅ Yes | CRITICAL, WARNING, INFO |
| `affected_component` | string | ✅ Yes | Component that raised error |
| `recovery_action` | string | ❌ No | Action taken or recommended |
| `correlated_event_id` | uuid | ❌ No | Related event (e.g., rejected order) |

---

## Implementation

### Python

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class ErrorType(Enum):
    ERR_CONN = "ERR_CONN"
    ERR_AUTH = "ERR_AUTH"
    ERR_TIMEOUT = "ERR_TIMEOUT"
    ERR_REJECT = "ERR_REJECT"
    ERR_PARSE = "ERR_PARSE"
    ERR_SYNC = "ERR_SYNC"
    ERR_RISK = "ERR_RISK"
    ERR_SYSTEM = "ERR_SYSTEM"
    ERR_RECOVER = "ERR_RECOVER"

class Severity(Enum):
    CRITICAL = "CRITICAL"
    WARNING = "WARNING"
    INFO = "INFO"

def log_error(
    client,
    error_type: ErrorType,
    error_code: str,
    error_message: str,
    severity: Severity,
    affected_component: str,
    recovery_action: Optional[str] = None,
    correlated_event_id: Optional[str] = None
) -> dict:
    """
    Log standardized error event.
    
    IMPORTANT: Error events MUST be included in Merkle batches.
    Do NOT filter out error events before anchoring.
    """
    payload = {
        "error_details": {
            "error_code": error_code,
            "error_message": error_message,
            "severity": severity.value,
            "affected_component": affected_component
        }
    }
    
    if recovery_action:
        payload["error_details"]["recovery_action"] = recovery_action
    
    if correlated_event_id:
        payload["error_details"]["correlated_event_id"] = correlated_event_id
    
    return client.log(error_type.value, payload)
```

### Usage Examples

```python
# Connection failure
log_error(
    client=vcp,
    error_type=ErrorType.ERR_CONN,
    error_code="BROKER_DISCONNECT",
    error_message="Lost connection to broker gateway at 10.0.0.1:443",
    severity=Severity.CRITICAL,
    affected_component="broker-gateway",
    recovery_action="Attempting reconnect with exponential backoff"
)

# Risk limit breach
log_error(
    client=vcp,
    error_type=ErrorType.ERR_RISK,
    error_code="MAX_DRAWDOWN_EXCEEDED",
    error_message="Daily drawdown 12.5% exceeds limit of 10%",
    severity=Severity.CRITICAL,
    affected_component="risk-manager",
    recovery_action="All open positions closed, trading halted"
)

# Order rejection
log_error(
    client=vcp,
    error_type=ErrorType.ERR_REJECT,
    error_code="INSUFFICIENT_MARGIN",
    error_message="Order rejected: insufficient margin for 50 lots XAUUSD",
    severity=Severity.WARNING,
    affected_component="order-manager",
    correlated_event_id="01934e3a-1234-7f93-8f2a-abcdef123456"
)

# Clock sync lost
log_error(
    client=vcp,
    error_type=ErrorType.ERR_SYNC,
    error_code="NTP_DRIFT_EXCEEDED",
    error_message="Clock drift 150ms exceeds threshold 100ms",
    severity=Severity.WARNING,
    affected_component="time-sync",
    recovery_action="Forcing NTP resync"
)

# Recovery completed
log_error(
    client=vcp,
    error_type=ErrorType.ERR_RECOVER,
    error_code="CONNECTION_RESTORED",
    error_message="Broker connection restored after 45 seconds",
    severity=Severity.INFO,
    affected_component="broker-gateway"
)
```

---

## Error Code Conventions

### Naming Pattern

```
<COMPONENT>_<CONDITION>

Examples:
  BROKER_DISCONNECT
  ORDER_REJECTED
  MARGIN_INSUFFICIENT
  CLOCK_DRIFT_EXCEEDED
  RATE_LIMIT_HIT
  API_KEY_EXPIRED
```

### Recommended Error Codes by Type

**ERR_CONN:**
- `BROKER_DISCONNECT`
- `EXCHANGE_UNREACHABLE`
- `NETWORK_TIMEOUT`
- `SSL_HANDSHAKE_FAILED`

**ERR_AUTH:**
- `API_KEY_INVALID`
- `API_KEY_EXPIRED`
- `IP_NOT_WHITELISTED`
- `PERMISSION_DENIED`

**ERR_RISK:**
- `MAX_POSITION_EXCEEDED`
- `MAX_DRAWDOWN_EXCEEDED`
- `MAX_LOSS_EXCEEDED`
- `CONCENTRATION_LIMIT_HIT`

**ERR_REJECT:**
- `INSUFFICIENT_MARGIN`
- `INVALID_SYMBOL`
- `MARKET_CLOSED`
- `PRICE_OUT_OF_RANGE`

---

## Important Rules

### 1. Include All Errors in Batches

Error events MUST NOT be filtered out before Merkle tree construction:

```python
# ❌ WRONG - filtering errors
events_to_anchor = [e for e in events if not e["header"]["event_type"].startswith("ERR_")]

# ✅ CORRECT - include all events
events_to_anchor = events  # All events including errors
```

### 2. Correlate with Related Events

When an error relates to another event (e.g., rejected order), include `correlated_event_id`:

```python
# Log order
order_event = vcp.log("ORD", {...})

# Later: order rejected
log_error(
    ...,
    correlated_event_id=order_event["header"]["event_id"]
)
```

### 3. Use Appropriate Severity

| Condition | Severity |
|-----------|----------|
| Trading cannot continue | CRITICAL |
| Trading degraded but functional | WARNING |
| Normal operational status | INFO |

---

## See Also

- [architecture.md](./architecture.md) — Event types overview
- [integrity-and-anchoring.md](./integrity-and-anchoring.md) — Including errors in batches

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
