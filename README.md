# VeritasChain Protocol (VCP) – Documentation Hub

Welcome to the documentation hub for the **VeritasChain Protocol (VCP)** —  
an open VSO specification for tamper-evident, cryptographically verifiable audit
trails in algorithmic and AI-driven trading. VCP has not been adopted by any
established standards body; its IETF profile is an individual Internet-Draft.

This repository hosts implementation guides and discussion materials maintained
by the **VeritasChain Standards Organization (VSO)**. Other VCP documentation
lives in its own repositories:

## 📘 Where to find VCP documentation

### **1. VCP Explorer (GUI)**
- Repository: [veritaschain/vcp-explorer-gui](https://github.com/veritaschain/vcp-explorer-gui)
- The legacy dashboard uses mock data by default. The separate
  [SDK Evidence Demo](https://github.com/veritaschain/vcp-explorer-gui/blob/main/docs/EVIDENCE_DEMO_INTEGRATION.md)
  displays bounded SDK results for synthetic evidence through a configured API.

### **2. VCP Explorer API**
- API reference, developer guide and OpenAPI description (Explorer API package v1.1.0)
- Repository: [veritaschain/vcp-explorer-api](https://github.com/veritaschain/vcp-explorer-api)
- Default event, system and entity routes return mock data, not verified evidence.
- The optional [SDK evidence routes](https://github.com/veritaschain/vcp-explorer-api/blob/main/docs/EVIDENCE_DEMO.md)
  run local checks over synthetic fixtures. The intact case remains
  `INDETERMINATE` overall because no external anchor is verified.

### **3. VCP Specification**
- Current release: VCP v1.2 (EN normative; JA / ZH translations); v1.0 and v1.1 are kept for reference
- Repository: [veritaschain/vcp-spec](https://github.com/veritaschain/vcp-spec)

### **4. VCP SDK (offline core)**
- Repository: [veritaschain/vcp-sdk](https://github.com/veritaschain/vcp-sdk)
- First-party Python, TypeScript and MQL5 source for VCP v1.2; unreleased
  development source with package metadata `0.1.0`.
- Follow the repository's [source installation instructions](https://github.com/veritaschain/vcp-sdk#install)
  and record the full source commit. Its
  [implementation interpretations](https://github.com/veritaschain/vcp-sdk/blob/main/INTERPRETATIONS.md)
  and verification limits do not establish protocol conformance.

### **5. VC-Certified Program**
- Scheme information: https://veritaschain.org/certified/
- The scheme requires Conformity Assessment Bodies (CABs) independent of the
  scheme owner; the only registered CAB, VeritasChain Co., Ltd., operates VSO and
  is not independent of it. Conformance is not regulatory compliance.

### **6. Standards & Implementation Guides (this repository)**

#### VCP v1.1 Implementation Guide (informative; superseded by v1.2)
Implementation guidance for VCP v1.1, including:
- Three-layer integrity architecture
- External anchoring (REQUIRED for all tiers)
- Policy Identification
- Error event standardization
- Sidecar integration patterns (MT5, FIX, Kafka)
- Migration from v1.0

Located under: `/standards/vcp/v1.1/`

#### Discussion Materials
- FIX Protocol gap analysis and tag mapping (non-normative)
- VCP–FIX sidecar integration architecture
- ISO/TC 68 JWG7 technical input (AI in Financial Services)

These materials are provided **for discussion purposes only** and do not represent
formal standard proposals, endorsements, or official positions of any standards body.

Located under: `/standards/`

---

## 🔐 About VCP

VCP provides a tamper-evident, cryptographically verifiable audit trail for
algorithmic trading systems using:

- UUIDv7 time-sortable identifiers  
- RFC 8785 canonical JSON  
- RFC 6962 Merkle proofs  
- Ed25519 digital signatures  
- Multi-module structure (CORE, TRADE, RISK, GOV, PRIVACY)

---

## 🌍 Philosophy

**Verify, Don't Trust.**  
Independent verification requires the relevant evidence and an explicit trust
policy. Use each implementation's documented verification scope: unavailable
keys or external proofs can leave checks unresolved, and a local match does not
establish full protocol conformance.

---

## 📄 Licensing

All documentation is licensed under **CC BY 4.0** unless otherwise noted.

---

## 🛠 Maintained By

**VeritasChain Standards Organization (VSO)**  
Encoding Trust in the Algorithmic Age  
https://veritaschain.org
