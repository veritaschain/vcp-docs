# VCP v1.0 Implementation Guide

**Document ID:** VSO-IMPL-v1.0-001  
**Status:** Stable (Superseded by v1.1)  
**Maintainer:** VeritasChain Standards Organization (VSO)

---

## 1. Overview

VCP v1.0 provides a cryptographic audit trail for algorithmic trading using:

- UUID v4/v7 event identifiers
- SHA-256 hashing (RFC 8785 canonicalization)
- Ed25519 digital signatures
- Hash chain linking (PrevHash)

---

## 2. Event Structure

### 2.1 Schema

```json
{
  "header": {
    "event_id": "uuid",
    "event_type": "ORD | EXE | CXL | SIG | RSK | ...",
    "timestamp_iso": "ISO-8601",
    "timestamp_int": "milliseconds-string",
    "timestamp_precision": "MILLISECOND | MICROSECOND | NANOSECOND",
    "clock_sync_status": "BEST_EFFORT | NTP_SYNCED | PTP_LOCKED",
    "vcp_version": "1.0",
    "hash_algo": "SHA256",
    "sign_algo": "ED25519"
  },
  "payload": {
    "trade_data": { ... },
    "vcp_risk": { ... },
    "vcp_gov": { ... }
  },
  "security": {
    "event_hash": "sha256-hex",
    "prev_hash": "sha256-hex",
    "signature": "ed25519-hex"
  }
}
```

### 2.2 Event Types

| Type | Description |
|------|-------------|
| `ORD` | Order submitted |
| `EXE` | Order executed |
| `CXL` | Order cancelled |
| `MOD` | Order modified |
| `SIG` | Signal generated |
| `RSK` | Risk event |

---

## 3. Hash Chain (REQUIRED in v1.0)

### 3.1 Overview

Every event MUST include `prev_hash` linking to the previous event:

```
[E1] ──hash──▶ [E2] ──hash──▶ [E3] ──hash──▶ [E4]
```

### 3.2 Implementation

```python
import hashlib
import json

class HashChain:
    def __init__(self):
        self.prev_hash = "0" * 64  # Genesis: 64 zeros
    
    def compute_hash(self, event: dict) -> str:
        """Compute EventHash."""
        hashable = {
            "header": event["header"],
            "payload": event["payload"]
        }
        canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
        return hashlib.sha256(canonical.encode()).hexdigest()
    
    def add_security(self, event: dict) -> dict:
        """Add security section with hash chain."""
        event_hash = self.compute_hash(event)
        
        event["security"] = {
            "event_hash": event_hash,
            "prev_hash": self.prev_hash  # REQUIRED in v1.0
        }
        
        self.prev_hash = event_hash  # Update chain
        return event
```

### 3.3 Genesis Event

The first event uses `prev_hash = "0" * 64` (64 zeros).

---

## 4. Compliance Tiers

### 4.1 Silver Tier

**Target:** MT4/MT5, Retail traders

```yaml
Clock Sync: BEST_EFFORT
Timestamp: MILLISECOND
Hash Chain: REQUIRED
Signature: Delegated or Ed25519
External Anchor: OPTIONAL
```

### 4.2 Gold Tier

**Target:** Prop firms, Institutional

```yaml
Clock Sync: NTP_SYNCED
Timestamp: MICROSECOND
Hash Chain: REQUIRED
Signature: Ed25519 (self-signing)
External Anchor: REQUIRED (hourly)
```

### 4.3 Platinum Tier

**Target:** HFT, Exchanges

```yaml
Clock Sync: PTP_LOCKED (<1μs)
Timestamp: NANOSECOND
Hash Chain: REQUIRED
Signature: Ed25519 (HSM recommended)
External Anchor: REQUIRED (10 minutes)
```

---

## 5. Code Examples

### 5.1 Python (Silver Tier)

```python
"""VCP v1.0 Silver Tier Implementation"""
import hashlib
import json
import uuid
from datetime import datetime, timezone

class VCPv10Client:
    def __init__(self):
        self.prev_hash = "0" * 64
    
    def log(self, event_type: str, payload: dict) -> dict:
        now = datetime.now(timezone.utc)
        
        event = {
            "header": {
                "event_id": str(uuid.uuid4()),
                "event_type": event_type,
                "timestamp_iso": now.isoformat(),
                "timestamp_int": str(int(now.timestamp() * 1000)),
                "timestamp_precision": "MILLISECOND",
                "clock_sync_status": "BEST_EFFORT",
                "vcp_version": "1.0",
                "hash_algo": "SHA256"
            },
            "payload": payload
        }
        
        # Compute hash
        hashable = {"header": event["header"], "payload": event["payload"]}
        canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
        event_hash = hashlib.sha256(canonical.encode()).hexdigest()
        
        # v1.0: prev_hash REQUIRED
        event["security"] = {
            "event_hash": event_hash,
            "prev_hash": self.prev_hash
        }
        
        self.prev_hash = event_hash
        return event
    
    def log_order(self, order_id: str, symbol: str, side: str, 
                  price: float, qty: float) -> dict:
        return self.log("ORD", {
            "trade_data": {
                "order_id": order_id,
                "symbol": symbol,
                "side": side,
                "price": str(price),
                "quantity": str(qty)
            }
        })
    
    def log_execution(self, order_id: str, symbol: str, side: str,
                      exec_price: float, exec_qty: float) -> dict:
        return self.log("EXE", {
            "trade_data": {
                "order_id": order_id,
                "symbol": symbol,
                "side": side,
                "execution_price": str(exec_price),
                "executed_qty": str(exec_qty)
            }
        })

# Usage
if __name__ == "__main__":
    vcp = VCPv10Client()
    
    order = vcp.log_order("ORD-001", "XAUUSD", "BUY", 2650.50, 1.0)
    print(f"Order hash: {order['security']['event_hash']}")
    
    exe = vcp.log_execution("ORD-001", "XAUUSD", "BUY", 2650.45, 1.0)
    print(f"Execution prev_hash: {exe['security']['prev_hash']}")
```

### 5.2 MQL5 Bridge

```mql5
//+------------------------------------------------------------------+
//| VCP v1.0 MQL5 Bridge                                              |
//+------------------------------------------------------------------+
#property copyright "VeritasChain Standards Organization"
#property version   "1.00"

#include <JAson.mqh>

input string VCP_SidecarURL = "http://localhost:8080/vcp/log";

class CVCPLoggerV10
{
private:
    string m_sidecarUrl;
    string m_prevHash;
    
public:
    CVCPLoggerV10(string sidecarUrl)
    {
        m_sidecarUrl = sidecarUrl;
        m_prevHash = "0000000000000000000000000000000000000000000000000000000000000000";
    }
    
    bool LogOrder(string orderId, string symbol, string side, double price, double qty)
    {
        CJAVal event;
        
        event["header"]["event_type"] = "ORD";
        event["header"]["timestamp_iso"] = TimeToString(TimeCurrent(), TIME_DATE|TIME_SECONDS);
        event["header"]["timestamp_int"] = IntegerToString(TimeCurrent() * 1000);
        event["header"]["vcp_version"] = "1.0";
        
        event["payload"]["trade_data"]["order_id"] = orderId;
        event["payload"]["trade_data"]["symbol"] = symbol;
        event["payload"]["trade_data"]["side"] = side;
        event["payload"]["trade_data"]["price"] = DoubleToString(price, 5);
        event["payload"]["trade_data"]["quantity"] = DoubleToString(qty, 2);
        
        // v1.0: prev_hash REQUIRED (sidecar computes event_hash)
        event["security"]["prev_hash"] = m_prevHash;
        
        return SendToSidecar(event.Serialize());
    }
    
private:
    bool SendToSidecar(string jsonData)
    {
        char data[], result[];
        string headers = "Content-Type: application/json\r\n";
        
        StringToCharArray(jsonData, data, 0, StringLen(jsonData));
        
        int res = WebRequest("POST", m_sidecarUrl, headers, 5000, data, result, headers);
        
        return (res == 200 || res == 201);
    }
};
```

---

## 6. Verification

### 6.1 Hash Chain Verification

```python
def verify_chain(events: list) -> bool:
    """Verify hash chain integrity."""
    expected_prev = "0" * 64
    
    for event in events:
        # Check prev_hash
        if event["security"]["prev_hash"] != expected_prev:
            return False
        
        # Recompute event_hash
        hashable = {"header": event["header"], "payload": event["payload"]}
        canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
        computed = hashlib.sha256(canonical.encode()).hexdigest()
        
        if event["security"]["event_hash"] != computed:
            return False
        
        expected_prev = computed
    
    return True
```

---

## 7. Limitations (Addressed in v1.1)

| Issue | v1.0 Behavior | v1.1 Solution |
|-------|---------------|---------------|
| Silver tier anchoring | Optional | REQUIRED |
| Hash chain dependency | Required for all | Optional |
| Policy declaration | None | Policy Identification |
| Cross-party verification | None | VCP-XREF |

---

## 8. Migration to v1.1

**Recommended:** Upgrade to v1.1 for enhanced features.

See [../v1.1/migration-from-v1.0.md](../v1.1/migration-from-v1.0.md)

---

## Contact

- Website: https://veritaschain.org
- Technical: technical@veritaschain.org

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
