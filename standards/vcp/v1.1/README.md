# VCP v1.1 Official Implementation Guide

**Document ID:** VSO-IMPL-001  
**Status:** Production Ready  
**Version:** 1.1.0  
**Date:** 2025-12-30  
**Maintainer:** VeritasChain Standards Organization (VSO)  
**License:** CC BY 4.0

---

## Overview

This directory contains the official implementation guidance for **VeritasChain Protocol (VCP) v1.1**.

VCP v1.1 introduces mandatory external anchoring for all tiers, Policy Identification, and clarifies the three-layer integrity architecture that enables the "Verify, Don't Trust" principle.

> **Note:** This documentation is **non-normative** but **authoritative**. For the normative specification, see [vcp-spec](https://github.com/veritaschain/vcp-spec).

---

## What's New in v1.1

| Feature | v1.0 | v1.1 | Impact |
|---------|------|------|--------|
| External Anchor (Silver) | OPTIONAL | **REQUIRED** | Implementation required |
| PrevHash (Hash Chain) | REQUIRED | **OPTIONAL** | Simplification allowed |
| Policy Identification | N/A | **REQUIRED** | New field required |
| VCP-XREF | N/A | OPTIONAL | New extension available |
| Error Events | Ad-hoc | Standardized | Schema update |

**Protocol Compatibility:** v1.1 is fully backward compatible with v1.0.

---

## Document Index

| Document | Description |
|----------|-------------|
| [architecture.md](./architecture.md) | Three-layer integrity architecture |
| [integrity-and-anchoring.md](./integrity-and-anchoring.md) | EventHash, Merkle Tree, External Anchoring |
| [policy-identification.md](./policy-identification.md) | PolicyID requirements and naming |
| [completeness-guarantees.md](./completeness-guarantees.md) | Omission detection and batch verification |
| [error-events.md](./error-events.md) | Standardized ERR_* event types |
| [vcp-xref.md](./vcp-xref.md) | Dual logging for cross-party verification |
| [sidecar/](./sidecar/) | Sidecar integration patterns |
| [migration-from-v1.0.md](./migration-from-v1.0.md) | Migration guide and deadlines |

---

## Compliance Tiers

| Tier | Clock Sync | Timestamp | External Anchor | Target |
|------|------------|-----------|-----------------|--------|
| **Silver** | BEST_EFFORT | MILLISECOND | Daily (24hr) | MT4/MT5, Retail |
| **Gold** | NTP_SYNCED | MICROSECOND | Hourly (1hr) | Prop Firms, Institutional |
| **Platinum** | PTP_LOCKED | NANOSECOND | 10 minutes | HFT, Exchanges |

---

## Certification Deadlines

| Requirement | Deadline |
|-------------|----------|
| Policy Identification | 2026-03-25 |
| External Anchor (Silver) | 2026-06-25 |

---

## Quick Links

- **Normative Specification:** [vcp-spec](https://github.com/veritaschain/vcp-spec)
- **SDK Implementation:** [vcp-sdk-spec](https://github.com/veritaschain/vcp-sdk-spec)
- **Reference Implementation:** [vcp-rta-reference](https://github.com/veritaschain/vcp-rta-reference)
- **IETF Draft:** [draft-kamimura-scitt-vcp](https://datatracker.ietf.org/doc/draft-kamimura-scitt-vcp/)

---

## Contact

**VeritasChain Standards Organization (VSO)**

- Website: https://veritaschain.org
- Technical: technical@veritaschain.org
- Standards: standards@veritaschain.org

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
