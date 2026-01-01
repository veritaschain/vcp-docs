# VCP v1.1 Completeness Guarantees

**Document ID:** VSO-IMPL-001-COMP  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

VCP v1.1 extends tamper-evidence to **completeness guarantees**, enabling third parties to cryptographically verify not only that events were not altered, but that **no required events were omitted**.

This addresses:
- **Omission attacks:** Selectively hiding unfavorable events
- **Split-view attacks:** Showing different event sets to different parties

---

## The Problem

### Without Completeness Guarantees

```
Producer logs:  [E1] [E2] [E3] [E4] [E5]
                         ↓ omit E3
Disclosed to auditor:  [E1] [E2] [E4] [E5]
```

If E3 was a risk limit breach, the producer could hide it while still providing "tamper-evident" logs for E1, E2, E4, E5.

### With Completeness Guarantees

```
Producer logs:  [E1] [E2] [E3] [E4] [E5]
                         ↓
Merkle Root anchored: abc123...
                         ↓
Auditor can verify: "Is E3 in the tree with root abc123?"
                         ↓
If E3 omitted: Merkle proof fails
```

---

## How It Works

### 1. Batch Formation

Events are collected into time-bounded batches:

| Tier | Batch Window | Events per Batch (typical) |
|------|--------------|---------------------------|
| Silver | 24 hours | 100 - 10,000 |
| Gold | 1 hour | 1,000 - 100,000 |
| Platinum | 10 minutes | 10,000 - 1,000,000 |

### 2. Merkle Tree Construction

All events in a batch are included in an RFC 6962 Merkle tree:

```
                    [Root]
                   /      \
              [H12]        [H34]
             /    \       /    \
          [H1]   [H2]  [H3]   [H4]
           |      |     |      |
          E1     E2    E3     E4
```

### 3. External Anchoring

The Merkle root is anchored to an external, tamper-evident store:

```
Merkle Root: abc123...
     ↓
Anchor to Bitcoin/TSA
     ↓
Third parties can verify: "At time T, the producer committed to root abc123"
```

### 4. Verification

To verify event inclusion:

```python
def verify_completeness(event, audit_path, anchored_root) -> bool:
    """
    Verify that event was included in the anchored batch.
    
    If this returns True:
    - Event existed at anchor time
    - Event has not been modified
    - Event was not added after anchoring
    """
    computed_root = compute_root_from_path(event, audit_path)
    return computed_root == anchored_root
```

---

## Attack Scenarios

### Scenario 1: Omission Attack

**Attack:** Producer omits event E3 from disclosure.

**Detection:**
1. Auditor requests Merkle proof for E3
2. Producer cannot provide valid proof (E3 not in tree)
3. Or: Auditor compares event count with batch metadata

**Mitigation:** External anchor commits to complete batch.

### Scenario 2: Late Addition Attack

**Attack:** Producer adds favorable event E6 after anchoring.

**Detection:**
1. E6's Merkle proof leads to different root
2. Different root not in external anchor

**Mitigation:** External anchor timestamp proves batch contents at specific time.

### Scenario 3: Split-View Attack

**Attack:** Producer shows different batches to different parties.

**Detection:**
1. Parties compare Merkle roots
2. Roots differ → different batches

**Mitigation:** Single anchored root is publicly verifiable.

---

## Implementation Requirements

### Batch Metadata (REQUIRED)

Each batch MUST include metadata for completeness verification:

```json
{
  "batch_metadata": {
    "batch_id": "uuid",
    "event_count": 1234,
    "first_event_id": "uuid",
    "last_event_id": "uuid",
    "first_timestamp": "ISO-8601",
    "last_timestamp": "ISO-8601",
    "merkle_root": "sha256-hex",
    "tree_size": 1234,
    "anchor": {
      "type": "OPENTIMESTAMPS",
      "anchored_at": "ISO-8601",
      "proof": "base64"
    }
  }
}
```

### Audit Path Storage (REQUIRED)

Implementations MUST store audit paths for all events:

```python
class BatchStore:
    def store_batch(self, events: List[dict], tree: MerkleTree):
        batch_id = str(uuid.uuid4())
        
        for i, event in enumerate(events):
            # Store event with its audit path
            audit_path = tree.get_audit_path(i)
            self.db.insert({
                "batch_id": batch_id,
                "event_id": event["header"]["event_id"],
                "event": event,
                "audit_path": audit_path,
                "leaf_index": i
            })
        
        # Store batch metadata
        self.db.insert({
            "batch_id": batch_id,
            "merkle_root": tree.root(),
            "event_count": len(events),
            "anchored_at": datetime.now(timezone.utc).isoformat()
        })
```

### Verification API (RECOMMENDED)

Provide API for third-party verification:

```
GET /api/v1/events/{event_id}/proof

Response:
{
  "event_id": "uuid",
  "batch_id": "uuid",
  "merkle_root": "sha256-hex",
  "leaf_index": 42,
  "audit_path": [
    {"hash": "abc...", "position": "R"},
    {"hash": "def...", "position": "L"},
    ...
  ],
  "anchor": {
    "type": "OPENTIMESTAMPS",
    "proof": "base64"
  }
}
```

---

## Consistency Proofs

For ongoing verification, implementations SHOULD support consistency proofs between batches:

```python
def verify_consistency(old_root: str, new_root: str, proof: List[str]) -> bool:
    """
    Verify that new_root is an extension of old_root.
    
    This proves:
    - All events in old batch are still in new batch
    - No events were removed or modified
    """
    # RFC 6962 consistency proof verification
    pass
```

---

## Completeness vs. Hash Chains

| Mechanism | Detects Tampering | Detects Omission | Real-time | Third-party Verifiable |
|-----------|-------------------|------------------|-----------|----------------------|
| **Hash Chain (PrevHash)** | ✅ Yes | ⚠️ Only if verified in real-time | ✅ Yes | ❌ No (requires trust) |
| **Merkle + Anchor** | ✅ Yes | ✅ Yes | ❌ Batch-based | ✅ Yes |
| **Both Combined** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |

**Recommendation:** Use both for maximum assurance (Gold/Platinum tiers).

---

## Regulatory Alignment

### MiFID II RTS 25

> "Investment firms shall ensure that records... are protected from unauthorized access... and maintained in a way that prevents any manipulation or alteration"

**VCP Compliance:** Merkle proofs + external anchoring provide cryptographic evidence of record completeness.

### EU AI Act Article 12

> "High-risk AI systems shall be designed... to ensure traceability of results... with sufficient information to ensure that the outputs can be traced back..."

**VCP Compliance:** Completeness guarantees ensure no decision records are omitted from audit trails.

---

## See Also

- [integrity-and-anchoring.md](./integrity-and-anchoring.md) — Merkle tree implementation
- [architecture.md](./architecture.md) — Three-layer overview

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
