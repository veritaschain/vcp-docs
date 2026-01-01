# VCP v1.1 Policy Identification

**Document ID:** VSO-IMPL-001-POL  
**Parent:** [VCP v1.1 Implementation Guide](./README.md)

---

## Overview

Policy Identification is **REQUIRED** in VCP v1.1. It declares which compliance tier and verification policy an event follows, enabling verifiers to understand the integrity guarantees provided.

**Certification Deadline:** 2026-03-25

---

## Schema

```json
{
  "policy_identification": {
    "version": "1.1",
    "policy_id": "com.example:silver-mt5-v1",
    "conformance_tier": "SILVER",
    "registration_policy": {
      "issuer": "Example Trading Corp",
      "policy_uri": "https://example.com/vcp-policy/silver-v1",
      "effective_date": 1704067200000,
      "expiration_date": null
    },
    "verification_depth": {
      "hash_chain_validation": false,
      "merkle_proof_required": true,
      "external_anchor_required": true
    }
  }
}
```

---

## Field Definitions

### Core Fields (REQUIRED)

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | VCP version ("1.1") |
| `policy_id` | string | Unique policy identifier |
| `conformance_tier` | enum | "SILVER" \| "GOLD" \| "PLATINUM" |

### Registration Policy (OPTIONAL)

| Field | Type | Description |
|-------|------|-------------|
| `issuer` | string | Organization name |
| `policy_uri` | string | URL to policy documentation |
| `effective_date` | int64 | Policy start timestamp (ms) |
| `expiration_date` | int64 \| null | Policy end timestamp (null = no expiration) |

### Verification Depth (REQUIRED)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `hash_chain_validation` | boolean | false | Whether PrevHash is used |
| `merkle_proof_required` | boolean | true | Merkle proofs available |
| `external_anchor_required` | boolean | true | External anchoring used |

---

## PolicyID Naming Convention

```
PolicyID = <reverse-domain>:<local-identifier>
```

### Format Rules

1. **Reverse Domain:** Organization's domain in reverse notation
2. **Separator:** Single colon (`:`)
3. **Local Identifier:** Alphanumeric with hyphens/underscores

### Valid Examples

```
org.veritaschain.prod:hft-system-001
com.propfirm.eval:silver-challenge-v2
jp.co.fintokei:gold-algo-v1
com.broker.platform:platinum-fix-gateway
io.exchange.spot:market-maker-001
```

### Invalid Examples

```
❌ hft-system-001                    # Missing domain
❌ com.example                       # Missing local identifier
❌ com.example::system               # Double colon
❌ COM.EXAMPLE:system                # Uppercase domain
❌ com.example:system with spaces    # Spaces not allowed
```

---

## Implementation

### Python

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class PolicyIdentification:
    version: str = "1.1"
    policy_id: str = ""
    conformance_tier: str = "SILVER"
    issuer: Optional[str] = None
    policy_uri: Optional[str] = None
    effective_date: Optional[int] = None
    expiration_date: Optional[int] = None
    hash_chain_validation: bool = False
    merkle_proof_required: bool = True
    external_anchor_required: bool = True
    
    def to_dict(self) -> dict:
        result = {
            "version": self.version,
            "policy_id": self.policy_id,
            "conformance_tier": self.conformance_tier,
            "verification_depth": {
                "hash_chain_validation": self.hash_chain_validation,
                "merkle_proof_required": self.merkle_proof_required,
                "external_anchor_required": self.external_anchor_required
            }
        }
        
        if self.issuer or self.policy_uri:
            result["registration_policy"] = {}
            if self.issuer:
                result["registration_policy"]["issuer"] = self.issuer
            if self.policy_uri:
                result["registration_policy"]["policy_uri"] = self.policy_uri
            if self.effective_date:
                result["registration_policy"]["effective_date"] = self.effective_date
            if self.expiration_date:
                result["registration_policy"]["expiration_date"] = self.expiration_date
        
        return result

# Usage
policy = PolicyIdentification(
    policy_id="com.example:trading-algo-v1",
    conformance_tier="GOLD",
    issuer="Example Trading Corp",
    policy_uri="https://example.com/vcp-policy/gold-v1",
    hash_chain_validation=True
)

event["policy_identification"] = policy.to_dict()
```

### TypeScript

```typescript
interface PolicyIdentification {
  version: '1.1';
  policy_id: string;
  conformance_tier: 'SILVER' | 'GOLD' | 'PLATINUM';
  registration_policy?: {
    issuer?: string;
    policy_uri?: string;
    effective_date?: number;
    expiration_date?: number | null;
  };
  verification_depth: {
    hash_chain_validation: boolean;
    merkle_proof_required: boolean;
    external_anchor_required: boolean;
  };
}

function createPolicy(
  policyId: string,
  tier: 'SILVER' | 'GOLD' | 'PLATINUM',
  useHashChain: boolean = false
): PolicyIdentification {
  return {
    version: '1.1',
    policy_id: policyId,
    conformance_tier: tier,
    verification_depth: {
      hash_chain_validation: useHashChain,
      merkle_proof_required: true,
      external_anchor_required: true,
    },
  };
}
```

---

## Tier-Specific Defaults

### Silver Tier

```json
{
  "policy_identification": {
    "version": "1.1",
    "policy_id": "com.example:silver-mt5",
    "conformance_tier": "SILVER",
    "verification_depth": {
      "hash_chain_validation": false,
      "merkle_proof_required": true,
      "external_anchor_required": true
    }
  }
}
```

### Gold Tier

```json
{
  "policy_identification": {
    "version": "1.1",
    "policy_id": "com.example:gold-algo",
    "conformance_tier": "GOLD",
    "verification_depth": {
      "hash_chain_validation": true,
      "merkle_proof_required": true,
      "external_anchor_required": true
    }
  }
}
```

### Platinum Tier

```json
{
  "policy_identification": {
    "version": "1.1",
    "policy_id": "com.example:platinum-hft",
    "conformance_tier": "PLATINUM",
    "verification_depth": {
      "hash_chain_validation": true,
      "merkle_proof_required": true,
      "external_anchor_required": true
    }
  }
}
```

---

## Validation Rules

1. **PolicyID Format:** MUST match pattern `^[a-z0-9.-]+:[a-zA-Z0-9_-]+$`
2. **Version:** MUST be "1.1" for v1.1 events
3. **Conformance Tier:** MUST be one of SILVER, GOLD, PLATINUM
4. **external_anchor_required:** MUST be `true` in v1.1

---

## See Also

- [architecture.md](./architecture.md) — Tier requirements
- [migration-from-v1.0.md](./migration-from-v1.0.md) — Adding PolicyID to existing systems

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
