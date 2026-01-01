# VCP v1.1 Three-Layer Architecture

**Document ID:** VSO-IMPL-001-ARCH  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

VCP v1.1 explicitly defines a **three-layer architecture** for integrity and security. This structure clarifies the relationship between different cryptographic mechanisms and their roles in ensuring audit trail integrity.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  LAYER 3: External Verifiability                                    │
│  ─────────────────────────────────                                  │
│  Purpose: Third-party verification without trusting the producer    │
│                                                                     │
│  Components:                                                        │
│  ├─ Digital Signature (Ed25519/Dilithium): REQUIRED                │
│  ├─ Timestamp (dual format ISO+int64): REQUIRED                    │
│  └─ External Anchor (Blockchain/TSA): REQUIRED                     │
│                                                                     │
│  Frequency: Tier-dependent (10min / 1hr / 24hr)                    │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LAYER 2: Collection Integrity    ← Core for external verifiability│
│  ──────────────────────────────                                     │
│  Purpose: Prove completeness of event batches                       │
│                                                                     │
│  Components:                                                        │
│  ├─ Merkle Tree (RFC 6962): REQUIRED                               │
│  ├─ Merkle Root: REQUIRED                                          │
│  └─ Audit Path (for verification): REQUIRED                        │
│                                                                     │
│  Note: Enables third-party verification of batch completeness      │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LAYER 1: Event Integrity                                           │
│  ─────────────────────────                                          │
│  Purpose: Individual event integrity                                │
│                                                                     │
│  Components:                                                        │
│  ├─ EventHash (SHA-256 of canonical event): REQUIRED               │
│  └─ PrevHash (link to previous event): OPTIONAL                    │
│                                                                     │
│  Note: PrevHash provides real-time detection (OPTIONAL in v1.1)    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer Responsibilities

| Layer | Purpose | REQUIRED | OPTIONAL |
|-------|---------|----------|----------|
| **Layer 3** | External Verifiability | Signature, Timestamp, External Anchor | Dual signatures (PQC) |
| **Layer 2** | Collection Integrity | Merkle Tree, Merkle Root, Audit Path | — |
| **Layer 1** | Event Integrity | EventHash | PrevHash (hash chain) |

---

## Why This Architecture?

### The v1.0 → v1.1 Evolution

**v1.0 Design Decision:**
- PrevHash-based hash chaining was REQUIRED
- Prioritized real-time, in-process tamper detection
- External Anchor was OPTIONAL for Silver tier

**v1.1 Design Decision:**
- PrevHash is OPTIONAL
- External Anchor is REQUIRED for all tiers
- Equivalent or stronger integrity through Merkle + External Anchor

### Rationale

1. **Simplifies Silver tier implementations** without sacrificing external verifiability
2. **Aligns with "Verify, Don't Trust"** by emphasizing externally verifiable proofs
3. **Maintains full backward compatibility** with v1.0 implementations

> **Recommendation:** Implementations that benefit from real-time tamper detection (e.g., HFT systems) SHOULD continue to use PrevHash.

---

## When to Use Hash Chains (PrevHash)

| Use Case | Hash Chain Recommended? | Rationale |
|----------|------------------------|-----------|
| HFT Systems | ✅ Yes | Real-time detection of event loss |
| Regulatory Submission | ✅ Yes | Familiar to auditors |
| Development/Testing | ❌ No | Simplifies implementation |
| Backtesting Analysis | ❌ No | Events may be generated out of order |
| MT4/MT5 Integration | ❌ No | Reduces DLL complexity |

---

## Data Flow

```
[Trading System]
       │
       ▼
┌──────────────────┐
│  Event Builder   │ ─────▶ Create event with header + payload
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Layer 1: Hash   │ ─────▶ Compute EventHash (+ optional PrevHash)
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Layer 2: Merkle │ ─────▶ Add to Merkle Tree, compute root
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Layer 3: Anchor │ ─────▶ Sign + Anchor to external store
└──────────────────┘
       │
       ▼
[Verifiable Audit Trail]
```

---

## See Also

- [integrity-and-anchoring.md](./integrity-and-anchoring.md) — Detailed implementation
- [completeness-guarantees.md](./completeness-guarantees.md) — Omission detection

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
