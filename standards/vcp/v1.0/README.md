# VCP v1.0 Implementation Guide

**Document ID:** VSO-IMPL-v1.0-001  
**Status:** Stable (Superseded by v1.1)  
**Version:** 1.0.0  
**Maintainer:** VeritasChain Standards Organization (VSO)  
**License:** CC BY 4.0

---

## Overview

This directory contains implementation guidance for **VCP v1.0**.

> **Note:** VCP v1.1 is now available with enhanced features. See [../v1.1/](../v1.1/) for the latest implementation guide.

---

## v1.0 vs v1.1

| Feature | v1.0 | v1.1 |
|---------|------|------|
| PrevHash (Hash Chain) | **REQUIRED** | OPTIONAL |
| External Anchor (Silver) | OPTIONAL | **REQUIRED** |
| Policy Identification | N/A | **REQUIRED** |
| VCP-XREF | N/A | OPTIONAL |

---

## Document Index

| Document | Description |
|----------|-------------|
| [implementation-guide.md](./implementation-guide.md) | Complete v1.0 implementation guide |

---

## Quick Start

```python
from datetime import datetime, timezone
import hashlib
import json
import uuid

class VCPv10Client:
    def __init__(self):
        self.prev_hash = "0" * 64  # Genesis
    
    def log(self, event_type: str, payload: dict) -> dict:
        now = datetime.now(timezone.utc)
        
        event = {
            "header": {
                "event_id": str(uuid.uuid4()),
                "event_type": event_type,
                "timestamp_iso": now.isoformat(),
                "timestamp_int": str(int(now.timestamp() * 1000)),
                "vcp_version": "1.0",
                "hash_algo": "SHA256"
            },
            "payload": payload
        }
        
        # Compute hash
        hashable = {"header": event["header"], "payload": event["payload"]}
        canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
        event_hash = hashlib.sha256(canonical.encode()).hexdigest()
        
        # v1.0: PrevHash REQUIRED
        event["security"] = {
            "event_hash": event_hash,
            "prev_hash": self.prev_hash
        }
        
        self.prev_hash = event_hash
        return event
```

---

## Compliance Tiers (v1.0)

| Tier | Clock Sync | Timestamp | External Anchor |
|------|------------|-----------|-----------------|
| **Silver** | BEST_EFFORT | MILLISECOND | OPTIONAL |
| **Gold** | NTP_SYNCED | MICROSECOND | REQUIRED (hourly) |
| **Platinum** | PTP_LOCKED | NANOSECOND | REQUIRED (10min) |

---

## Migration to v1.1

See [../v1.1/migration-from-v1.0.md](../v1.1/migration-from-v1.0.md)

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
