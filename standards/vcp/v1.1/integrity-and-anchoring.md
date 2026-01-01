# VCP v1.1 Integrity and Anchoring

**Document ID:** VSO-IMPL-001-INTEG  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Layer 1: Event Integrity

### EventHash Calculation (REQUIRED)

Every VCP event MUST include an EventHash computed over its canonical form.

```python
import hashlib
import json

def compute_event_hash(event: dict) -> str:
    """
    Compute EventHash per VCP v1.1.
    
    Steps:
    1. Extract header + payload (exclude security)
    2. Canonicalize JSON (RFC 8785)
    3. SHA-256 hash
    """
    hashable = {
        "header": event["header"],
        "payload": event["payload"]
    }
    
    # RFC 8785 canonicalization (simplified)
    canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
    
    return hashlib.sha256(canonical.encode()).hexdigest()
```

### PrevHash (OPTIONAL in v1.1)

Implementations MAY include PrevHash for real-time tamper detection.

```python
# v1.0: prev_hash was REQUIRED
# v1.1: prev_hash is OPTIONAL

security = {
    "event_hash": event_hash,
    "prev_hash": previous_event_hash  # OPTIONAL - omit if not using hash chain
}
```

**Genesis Event:** When using hash chains, the first event MUST have `prev_hash` set to `"0" * 64` (64 zeros).

---

## Layer 2: Collection Integrity

### Merkle Tree Construction (REQUIRED)

All VCP implementations MUST construct Merkle Trees over event batches per RFC 6962.

```python
import hashlib
from typing import List

class MerkleTree:
    """RFC 6962 compliant Merkle Tree."""
    
    LEAF_PREFIX = b'\x00'   # Domain separation for leaves
    NODE_PREFIX = b'\x01'   # Domain separation for internal nodes
    
    def __init__(self):
        self.leaves: List[bytes] = []
    
    def add(self, event_hash: str):
        """Add event hash as leaf."""
        leaf = hashlib.sha256(
            self.LEAF_PREFIX + bytes.fromhex(event_hash)
        ).digest()
        self.leaves.append(leaf)
    
    def root(self) -> str:
        """Compute Merkle root."""
        if not self.leaves:
            return "0" * 64
        
        nodes = self.leaves.copy()
        while len(nodes) > 1:
            # Duplicate last node if odd count
            if len(nodes) % 2 == 1:
                nodes.append(nodes[-1])
            
            next_level = []
            for i in range(0, len(nodes), 2):
                combined = hashlib.sha256(
                    self.NODE_PREFIX + nodes[i] + nodes[i+1]
                ).digest()
                next_level.append(combined)
            nodes = next_level
        
        return nodes[0].hex()
    
    def get_audit_path(self, index: int) -> List[tuple]:
        """Generate audit path for verification."""
        # Implementation per RFC 6962
        pass
```

### Merkle Proof Verification

```python
def verify_merkle_proof(
    event_hash: str,
    audit_path: List[tuple],
    expected_root: str
) -> bool:
    """Verify event inclusion in Merkle tree."""
    current = hashlib.sha256(b'\x00' + bytes.fromhex(event_hash)).digest()
    
    for sibling_hex, position in audit_path:
        sibling = bytes.fromhex(sibling_hex)
        if position == 'L':
            current = hashlib.sha256(b'\x01' + sibling + current).digest()
        else:
            current = hashlib.sha256(b'\x01' + current + sibling).digest()
    
    return current.hex() == expected_root
```

---

## Layer 3: External Anchoring

### Requirements by Tier

| Tier | Frequency | REQUIRED |
|------|-----------|----------|
| **Platinum** | Every 10 minutes | ✅ |
| **Gold** | Every 1 hour | ✅ |
| **Silver** | Every 24 hours | ✅ (NEW in v1.1) |

### Anchor Targets

| Target | Description | Cost | Recommended For |
|--------|-------------|------|-----------------|
| **OpenTimestamps** | Bitcoin-backed, free | FREE | Silver |
| **FreeTSA** | RFC 3161, free | FREE | Silver/Gold |
| **DigiCert TSA** | RFC 3161, commercial | $ | Gold |
| **Ethereum** | On-chain anchoring | $$ | Platinum |
| **Certified Database** | SOC 2 audited DB | $$$ | All tiers |

### OpenTimestamps Implementation (Silver)

```bash
pip install opentimestamps-client
```

```python
import opentimestamps

def anchor_opentimestamps(merkle_root: str) -> dict:
    """
    Anchor to Bitcoin via OpenTimestamps.
    FREE service - no API key required.
    """
    root_bytes = bytes.fromhex(merkle_root)
    timestamp = opentimestamps.stamp(root_bytes)
    
    # Save proof file (IMPORTANT: retain for verification)
    proof_path = f"anchors/{merkle_root[:16]}.ots"
    with open(proof_path, 'wb') as f:
        f.write(timestamp.serialize())
    
    return {
        "anchor_type": "OPENTIMESTAMPS",
        "target": "BITCOIN",
        "merkle_root": merkle_root,
        "proof_file": proof_path,
        "status": "PENDING"  # Confirmed in ~2 hours
    }
```

### RFC 3161 TSA Implementation (Gold/Platinum)

```python
import requests
from cryptography.hazmat.primitives import hashes
from cryptography.x509 import tsp

def anchor_rfc3161(merkle_root: str, tsa_url: str) -> dict:
    """
    Anchor to RFC 3161 Time Stamp Authority.
    
    TSA URLs:
    - FreeTSA: https://freetsa.org/tsr
    - DigiCert: https://timestamp.digicert.com
    - Sectigo: http://timestamp.sectigo.com
    """
    request = tsp.TimeStampRequest(
        message_imprint=tsp.MessageImprint(
            hash_algorithm=hashes.SHA256(),
            hashed_message=bytes.fromhex(merkle_root)
        ),
        cert_req=True
    )
    
    response = requests.post(
        tsa_url,
        data=request.public_bytes(),
        headers={"Content-Type": "application/timestamp-query"}
    )
    
    tsr = tsp.TimeStampResponse.from_bytes(response.content)
    
    return {
        "anchor_type": "RFC3161",
        "tsa_url": tsa_url,
        "merkle_root": merkle_root,
        "anchored_at": tsr.time_stamp_token.time.isoformat(),
        "serial_number": str(tsr.serial_number),
        "tsr_bytes": response.content.hex()
    }
```

### Anchor Unavailability Handling

Implementations MUST handle anchor target unavailability gracefully:

```python
class AnchorManager:
    def __init__(self, primary, fallbacks: list):
        self.primary = primary
        self.fallbacks = fallbacks
        self.pending = []
    
    def anchor(self, merkle_root: str) -> dict:
        """Anchor with fallback support."""
        try:
            return self.primary.anchor(merkle_root)
        except Exception:
            for fallback in self.fallbacks:
                try:
                    return fallback.anchor(merkle_root)
                except:
                    continue
            
            # All failed - queue for retry
            self.pending.append({
                "merkle_root": merkle_root,
                "queued_at": datetime.now(timezone.utc).isoformat(),
                "retry_count": 0
            })
            raise Exception("All anchor targets unavailable")
    
    def retry_pending(self):
        """Retry pending anchors (call periodically)."""
        for item in self.pending[:]:
            if item["retry_count"] > 5:
                # Log critical failure
                continue
            try:
                self.anchor(item["merkle_root"])
                self.pending.remove(item)
            except:
                item["retry_count"] += 1
```

---

## Certified Database Requirements

A "Certified Database" MUST meet these criteria:

1. **Third-Party Audit:** Annual independent audit (SOC 2 Type II or equivalent)
2. **Tamper Detection:** Cryptographic integrity checks
3. **Access Control:** Role-based access with audit logging
4. **Retention Policy:** Data retention ≥ regulatory minimum (typically 7 years)
5. **Availability SLA:** ≥ 99.9% uptime commitment

**Examples:**
- AWS QLDB (Quantum Ledger Database)
- Azure SQL Ledger
- Google Cloud Spanner with audit trail
- Self-hosted PostgreSQL with annual cryptographic audit

---

## Anchor Record Schema

```json
{
  "anchor": {
    "anchor_type": "OPENTIMESTAMPS | RFC3161 | ETHEREUM | CERTIFIED_DB",
    "merkle_root": "sha256-hex-string",
    "anchored_at": "ISO-8601-timestamp",
    "event_count": 1234,
    "batch_start": "first-event-timestamp",
    "batch_end": "last-event-timestamp",
    "proof": "base64-or-hex-encoded-proof",
    "target_details": {
      "network": "bitcoin-mainnet | ethereum-mainnet | ...",
      "tx_hash": "transaction-hash-if-applicable",
      "block_number": 12345678
    }
  }
}
```

---

## See Also

- [architecture.md](./architecture.md) — Three-layer overview
- [completeness-guarantees.md](./completeness-guarantees.md) — Omission detection

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
