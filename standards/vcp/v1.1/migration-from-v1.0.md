# VCP v1.1 Migration from v1.0

**Document ID:** VSO-IMPL-001-MIG  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

This guide covers migrating existing VCP v1.0 implementations to v1.1. VCP v1.1 is **fully backward compatible** at the protocol level, but introduces new **certification requirements**.

---

## Compatibility Summary

| Aspect | Compatibility |
|--------|--------------|
| **Protocol** | ✅ Fully compatible |
| **v1.0 → v1.1 Reading** | ✅ v1.1 systems can read v1.0 logs |
| **v1.1 → v1.0 Reading** | ✅ v1.0 systems can read v1.1 logs (ignore new fields) |
| **Certification** | ⚠️ New requirements for VC-Certified status |

---

## Certification Deadlines

| Requirement | Deadline | Affected Tiers |
|-------------|----------|----------------|
| Policy Identification | **2026-03-25** | All |
| External Anchor | **2026-06-25** | Silver |

---

## Migration Steps

### Step 1: Update vcp_version

Change the version field in event headers:

```python
# Before (v1.0)
header = {
    "vcp_version": "1.0",
    ...
}

# After (v1.1)
header = {
    "vcp_version": "1.1",
    ...
}
```

### Step 2: Add Policy Identification (REQUIRED)

Add `policy_identification` to all events:

```python
# Before (v1.0)
event = {
    "header": {...},
    "payload": {...},
    "security": {...}
}

# After (v1.1)
event = {
    "header": {...},
    "payload": {...},
    "policy_identification": {  # NEW - REQUIRED
        "version": "1.1",
        "policy_id": "com.yourcompany:system-001",
        "conformance_tier": "SILVER",
        "verification_depth": {
            "hash_chain_validation": False,
            "merkle_proof_required": True,
            "external_anchor_required": True
        }
    },
    "security": {...}
}
```

**PolicyID Format:** `<reverse-domain>:<local-identifier>`

Examples:
- `com.propfirm.eval:silver-challenge-v2`
- `jp.co.broker:gold-algo-trading`

### Step 3: Add External Anchoring (Silver Tier)

If you're Silver tier, implement daily anchoring:

```bash
# Install OpenTimestamps (free, Bitcoin-backed)
pip install opentimestamps-client
```

```python
import opentimestamps

def daily_anchor(merkle_root: str):
    """
    Anchor Merkle root to Bitcoin via OpenTimestamps.
    Call this at least once per 24 hours.
    """
    root_bytes = bytes.fromhex(merkle_root)
    timestamp = opentimestamps.stamp(root_bytes)
    
    # Save proof file (IMPORTANT: retain for verification)
    proof_path = f"anchors/{datetime.now().strftime('%Y%m%d')}.ots"
    with open(proof_path, 'wb') as f:
        f.write(timestamp.serialize())
    
    return {
        "merkle_root": merkle_root,
        "anchor_type": "OPENTIMESTAMPS",
        "proof_file": proof_path
    }
```

**Integration Point:**
```python
class VCPClient:
    def __init__(self, ...):
        self.last_anchor = None
    
    def log(self, event_type, payload):
        # ... existing logic ...
        
        # Check if daily anchor is due
        self._check_anchor()
    
    def _check_anchor(self):
        now = datetime.now(timezone.utc)
        if self.last_anchor is None or (now - self.last_anchor).days >= 1:
            self.anchor()
    
    def anchor(self):
        root = self.merkle_tree.root()
        daily_anchor(root)
        self.last_anchor = datetime.now(timezone.utc)
        self.merkle_tree.reset()
```

### Step 4: (Optional) Remove Hash Chain

If you were using PrevHash and want to simplify:

```python
# v1.0: prev_hash was REQUIRED
security = {
    "event_hash": event_hash,
    "prev_hash": self.prev_hash  # Required in v1.0
}

# v1.1: prev_hash is OPTIONAL
security = {
    "event_hash": event_hash
    # prev_hash omitted - valid in v1.1
}
```

**When to Keep Hash Chain:**
- HFT systems (real-time detection)
- Regulatory submissions (auditor familiarity)
- Gold/Platinum tiers (recommended)

**When to Remove:**
- Silver tier (simplification)
- Development/testing
- Backtesting systems

---

## Code Changes by Language

### Python

```python
# v1.0 Client
class VCPClientV10:
    def log(self, event_type, payload):
        event = {
            "header": {
                "vcp_version": "1.0",
                ...
            },
            "payload": payload,
            "security": {
                "event_hash": ...,
                "prev_hash": ...  # Required
            }
        }
        return event

# v1.1 Client
class VCPClientV11:
    def __init__(self, policy_id: str, tier: str):
        self.policy_id = policy_id
        self.tier = tier
    
    def log(self, event_type, payload):
        event = {
            "header": {
                "vcp_version": "1.1",  # Updated
                ...
            },
            "payload": payload,
            "policy_identification": {  # NEW
                "version": "1.1",
                "policy_id": self.policy_id,
                "conformance_tier": self.tier,
                "verification_depth": {
                    "hash_chain_validation": False,
                    "merkle_proof_required": True,
                    "external_anchor_required": True
                }
            },
            "security": {
                "event_hash": ...
                # prev_hash optional
            }
        }
        return event
```

### TypeScript

```typescript
// v1.0
interface VCPEventV10 {
  header: { vcp_version: '1.0'; ... };
  payload: unknown;
  security: {
    event_hash: string;
    prev_hash: string;  // Required
  };
}

// v1.1
interface VCPEventV11 {
  header: { vcp_version: '1.1'; ... };
  payload: unknown;
  policy_identification: {  // NEW
    version: '1.1';
    policy_id: string;
    conformance_tier: 'SILVER' | 'GOLD' | 'PLATINUM';
    verification_depth: {
      hash_chain_validation: boolean;
      merkle_proof_required: boolean;
      external_anchor_required: boolean;
    };
  };
  security: {
    event_hash: string;
    prev_hash?: string;  // Optional
  };
}
```

### MQL5

```mql5
// v1.0
event["header"]["vcp_version"] = "1.0";
event["security"]["prev_hash"] = prevHash;  // Required

// v1.1
event["header"]["vcp_version"] = "1.1";
event["policy_identification"]["version"] = "1.1";
event["policy_identification"]["policy_id"] = policyId;
event["policy_identification"]["conformance_tier"] = "SILVER";
// prev_hash can be omitted
```

---

## Migration Checklist

### All Tiers

```
[ ] Update vcp_version: "1.0" → "1.1"
[ ] Define PolicyID following naming convention
[ ] Add policy_identification to all events
[ ] Update event schema/types
[ ] Update unit tests
[ ] Update integration tests
```

### Silver Tier (Additional)

```
[ ] Install OpenTimestamps or alternative anchor
[ ] Implement daily anchoring logic
[ ] Configure anchor storage (proof files)
[ ] Set up anchor monitoring/alerts
[ ] Test anchor verification
```

### Gold/Platinum Tier (Verification)

```
[ ] Verify external anchoring already in place
[ ] Update anchor frequency documentation
[ ] No additional implementation needed
```

---

## Testing Migration

### Verify Policy Identification

```python
def test_policy_identification():
    client = VCPClientV11(
        policy_id="com.test:migration-v1",
        tier="SILVER"
    )
    
    event = client.log("ORD", {"test": "data"})
    
    assert "policy_identification" in event
    assert event["policy_identification"]["version"] == "1.1"
    assert event["policy_identification"]["policy_id"] == "com.test:migration-v1"
    assert event["policy_identification"]["conformance_tier"] == "SILVER"
```

### Verify External Anchoring

```python
def test_external_anchor():
    client = VCPClientV11(...)
    
    # Log events
    for i in range(10):
        client.log("ORD", {"order_id": f"ORD-{i}"})
    
    # Anchor
    anchor = client.anchor()
    
    assert "merkle_root" in anchor
    assert len(anchor["merkle_root"]) == 64  # SHA-256 hex
    assert "proof_file" in anchor or "anchor_type" in anchor
```

---

## Rollback Plan

If issues occur during migration:

1. **Keep v1.0 code path available** for first 30 days
2. **Log both versions** during transition period
3. **Monitor for verification failures**

```python
class VCPClientDualMode:
    def __init__(self, use_v11: bool = True):
        self.use_v11 = use_v11
    
    def log(self, event_type, payload):
        if self.use_v11:
            return self._log_v11(event_type, payload)
        else:
            return self._log_v10(event_type, payload)
```

---

## FAQ

### Q: Do I need to re-process historical v1.0 logs?

**A:** No. v1.0 logs remain valid. v1.1 is backward compatible for reading.

### Q: What if I miss the deadline?

**A:** After the deadline, new VC-Certified certifications require v1.1 compliance. Existing certifications have a 6-month grace period.

### Q: Can I use both versions simultaneously?

**A:** Yes, during transition. But new events should use v1.1 after migration.

### Q: Is OpenTimestamps the only option for Silver anchoring?

**A:** No. Any anchor target meeting VCP requirements is acceptable:
- OpenTimestamps (free, recommended)
- FreeTSA (free RFC 3161)
- Commercial TSAs
- Certified databases

---

## See Also

- [README.md](./README.md) — v1.1 overview
- [policy-identification.md](./policy-identification.md) — PolicyID details
- [integrity-and-anchoring.md](./integrity-and-anchoring.md) — Anchoring implementation

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
