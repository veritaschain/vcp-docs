# VCP v1.1 VCP-XREF Dual Logging

**Document ID:** VSO-IMPL-001-XREF  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

VCP-XREF is an **OPTIONAL** extension that enables **dual logging** — multiple independent parties record the same transaction for cross-verification. This provides higher assurance than single-party logging.

---

## Architecture

```
┌──────────────────┐          ┌──────────────────┐
│     Trader       │─────────▶│     Broker       │
│   (INITIATOR)    │  Trade   │  (COUNTERPARTY)  │
└────────┬─────────┘          └────────┬─────────┘
         │                             │
         ▼                             ▼
┌──────────────────┐          ┌──────────────────┐
│   VCP Sidecar    │          │   Broker VCP     │
│   (Trader side)  │          │   (Broker side)  │
└────────┬─────────┘          └────────┬─────────┘
         │                             │
         │    Same CrossReferenceID    │
         └───────────┬─────────────────┘
                     ▼
            ┌─────────────────┐
            │  Cross-Reference │
            │   Verification   │
            └─────────────────┘
```

---

## Schema

```json
{
  "vcp_xref": {
    "version": "1.1",
    "cross_reference_id": "01934e3a-7b2c-7f93-8f2a-1234567890ab",
    "party_role": "INITIATOR",
    "counterparty_id": "broker.example.com",
    "shared_event_key": {
      "order_id": "ORD-12345",
      "alternate_keys": ["BROKER-REF-789"],
      "timestamp": 1735560000000,
      "tolerance_ms": 1000
    },
    "reconciliation_status": "PENDING"
  }
}
```

---

## Field Definitions

### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | ✅ Yes | VCP-XREF version ("1.1") |
| `cross_reference_id` | uuid | ✅ Yes | Shared ID between parties |
| `party_role` | enum | ✅ Yes | INITIATOR, COUNTERPARTY, OBSERVER |
| `counterparty_id` | string | ✅ Yes | Identifier of other party |

### Shared Event Key

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | ✅ Yes | Primary matching key |
| `alternate_keys` | array | ❌ No | Additional matching keys |
| `timestamp` | int64 | ✅ Yes | Event timestamp (ms) |
| `tolerance_ms` | int32 | ✅ Yes | Timestamp match tolerance |

### Reconciliation Status

| Status | Description |
|--------|-------------|
| `PENDING` | Awaiting counterparty record |
| `MATCHED` | Both records match |
| `DISCREPANCY` | Records do not match |
| `TIMEOUT` | Counterparty record not received |

---

## Party Roles

| Role | Description | Example |
|------|-------------|---------|
| **INITIATOR** | Party that initiated the transaction | Trader sending order |
| **COUNTERPARTY** | Party that received/executed | Broker, Exchange |
| **OBSERVER** | Third party recording for audit | Regulator, Auditor |

---

## Implementation

### Trader Side (INITIATOR)

```python
import uuid
from datetime import datetime, timezone

class VCPXRefClient:
    def __init__(self, base_client, party_id: str):
        self.client = base_client
        self.party_id = party_id
    
    def log_with_xref(
        self,
        event_type: str,
        payload: dict,
        counterparty_id: str,
        order_id: str,
        cross_reference_id: str = None
    ) -> dict:
        """Log event with VCP-XREF extension."""
        
        # Generate or use provided cross reference ID
        xref_id = cross_reference_id or str(uuid.uuid4())
        
        # Add VCP-XREF to payload
        payload["vcp_xref"] = {
            "version": "1.1",
            "cross_reference_id": xref_id,
            "party_role": "INITIATOR",
            "counterparty_id": counterparty_id,
            "shared_event_key": {
                "order_id": order_id,
                "timestamp": int(datetime.now(timezone.utc).timestamp() * 1000),
                "tolerance_ms": 1000
            },
            "reconciliation_status": "PENDING"
        }
        
        return self.client.log(event_type, payload)

# Usage
trader_vcp = VCPXRefClient(vcp_client, "trader.example.com")

order_event = trader_vcp.log_with_xref(
    event_type="ORD",
    payload={
        "trade_data": {
            "order_id": "ORD-12345",
            "symbol": "XAUUSD",
            "side": "BUY",
            "quantity": "1.0"
        }
    },
    counterparty_id="broker.example.com",
    order_id="ORD-12345"
)

# Share xref_id with broker for matching
xref_id = order_event["payload"]["vcp_xref"]["cross_reference_id"]
```

### Broker Side (COUNTERPARTY)

```python
# Broker receives xref_id from trader (e.g., in FIX message)
def log_counterparty_event(
    broker_vcp,
    event_type: str,
    payload: dict,
    xref_id: str,
    trader_id: str,
    order_id: str
) -> dict:
    """Log counterparty event with matching xref_id."""
    
    payload["vcp_xref"] = {
        "version": "1.1",
        "cross_reference_id": xref_id,  # SAME ID as trader
        "party_role": "COUNTERPARTY",
        "counterparty_id": trader_id,
        "shared_event_key": {
            "order_id": order_id,
            "timestamp": int(datetime.now(timezone.utc).timestamp() * 1000),
            "tolerance_ms": 1000
        },
        "reconciliation_status": "PENDING"
    }
    
    return broker_vcp.log(event_type, payload)
```

---

## Reconciliation

### Matching Algorithm

```python
class XRefReconciler:
    def __init__(self, db):
        self.db = db
    
    def reconcile(self, xref_id: str) -> str:
        """Reconcile events from both parties."""
        
        events = self.db.query(
            "SELECT * FROM vcp_events WHERE xref_id = ?",
            [xref_id]
        )
        
        if len(events) < 2:
            return "PENDING"
        
        initiator = next(
            (e for e in events if e["party_role"] == "INITIATOR"),
            None
        )
        counterparty = next(
            (e for e in events if e["party_role"] == "COUNTERPARTY"),
            None
        )
        
        if not initiator or not counterparty:
            return "PENDING"
        
        if self._compare_events(initiator, counterparty):
            return "MATCHED"
        else:
            return "DISCREPANCY"
    
    def _compare_events(self, e1: dict, e2: dict) -> bool:
        """Compare key fields within tolerance."""
        
        # Order ID must match exactly
        if e1["order_id"] != e2["order_id"]:
            return False
        
        # Timestamp within tolerance
        tolerance = max(e1["tolerance_ms"], e2["tolerance_ms"])
        if abs(e1["timestamp"] - e2["timestamp"]) > tolerance:
            return False
        
        # Add more comparisons as needed (price, quantity, etc.)
        return True
```

### Discrepancy Handling

When records don't match, log a discrepancy event:

```python
def log_discrepancy(
    client,
    xref_id: str,
    initiator_event: dict,
    counterparty_event: dict,
    differences: list
):
    """Log VCP-XREF discrepancy for investigation."""
    
    payload = {
        "xref_discrepancy": {
            "cross_reference_id": xref_id,
            "initiator_event_id": initiator_event["header"]["event_id"],
            "counterparty_event_id": counterparty_event["header"]["event_id"],
            "differences": differences,
            "detected_at": datetime.now(timezone.utc).isoformat()
        }
    }
    
    return client.log("XREF_DISCREPANCY", payload)
```

---

## Use Cases

### 1. Prop Firm Trading

**Problem:** Disputes over trade execution between trader and prop firm.

**Solution:** Both parties log with VCP-XREF:
- Trader logs order submission
- Prop firm logs execution
- Discrepancies automatically detected

### 2. Broker Best Execution

**Problem:** Proving best execution to regulators.

**Solution:**
- Client logs order
- Broker logs execution with venue details
- Cross-reference proves execution path

### 3. Multi-Party Transactions

**Problem:** Complex trades involving multiple parties.

**Solution:**
- Each party logs as INITIATOR/COUNTERPARTY/OBSERVER
- Full transaction chain verifiable

---

## Relationship to External Anchoring

VCP-XREF is **complementary** to External Anchoring:

| Mechanism | Provides | Limitation |
|-----------|----------|------------|
| **External Anchor** | Tamper evidence (single party) | Single party could omit events |
| **VCP-XREF** | Cross-party verification | Requires counterparty cooperation |
| **Both combined** | Maximum assurance | Collusion required to manipulate |

---

## Protocol Integration

### FIX Protocol

Include `xref_id` in FIX messages using custom tag or existing field:

```
Tag 11 (ClOrdID): ORD-12345
Tag 20000 (custom): xref_id=01934e3a-7b2c-7f93-8f2a-1234567890ab
```

### REST API

Include in order request:

```json
POST /api/orders
{
  "order_id": "ORD-12345",
  "symbol": "XAUUSD",
  "side": "BUY",
  "vcp_xref_id": "01934e3a-7b2c-7f93-8f2a-1234567890ab"
}
```

---

## See Also

- [integrity-and-anchoring.md](./integrity-and-anchoring.md) — External anchoring
- [completeness-guarantees.md](./completeness-guarantees.md) — Omission detection

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
