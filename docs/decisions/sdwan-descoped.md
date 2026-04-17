---
title: "Decision: SD-WAN Features Descoped (F12, F13, F14)"
tags: [sd-wan, ztna, architecture, decision]
---

# Decision: SD-WAN Features Descoped (F12, F13, F14)

**Status:** Implemented  
**Date:** March 2026 (Herziening v3), confirmed April 2026 (Research SD-WAN session)

## Context

The original Handboek included three SD-WAN acceptance tests:

- **F12** — IPsec tunnel connectivity between site01 and the datacenter
- **F13** — QoS traffic classification on VyOS
- **F14** — Datacenter access from site users via the SD-WAN tunnel

Implementing these would require: an IPsec tunnel on VyOS, a NetBird container running on VyOS as a uCPE (customer premises equipment) host, and VyOS QoS configuration. All three were deferred and then explicitly descoped.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| Implement IPsec + NetBird uCPE on VyOS | Matches original Handboek spec | Replicates pre-SASE network-centric patterns; site-to-site tunnel grants implicit subnet-level access, contradicting Zero Trust; duplicates the ZTNA overlay |
| Descope SD-WAN tests; enroll sitepc01 in NetBird individually | Aligns with Zero Trust architecture; consistent with Zscaler model; simpler | F12/F13/F14 must be marked N/A in all test matrices |
| Implement QoS only (without IPsec) | Partial compliance | QoS configuration on VyOS without the accompanying IPsec tunnel is context-free and not meaningful for the PoC |

## Decision

SD-WAN tests F12, F13, and F14 are descoped. VyOS remains in the topology as the site01 WAN gateway and NAT device for Site-LAN, but is not configured as an IPsec router or uCPE NetBird host.

Site users (sitepc01) will access the datacenter via individual NetBird enrollment — the same mechanism as mobile01 — rather than via a site-to-site tunnel.

## Consequences

- **F12, F13, F14** are marked N/A in all acceptance test matrices. This is an explicit architectural decision, not an implementation gap.
- **F15 steps 7–8** are N/A for the same reason (step 7 requires sitepc01 with NetBird; step 8 requires QoS on VyOS).
- **VyOS** remains in the topology but its configuration is minimal — WAN connectivity and NAT only. See [VyOS](../components/vyos.md).
- **sitepc01** datacenter access is planned via NetBird enrollment (not yet executed as of April 2026).
- The architecture directly aligns with Zscaler's "Zero Trust SD-WAN" model: branches treated as untrusted networks (like cafés), every device authenticates individually, no site-to-site tunnels.

## Rationale

**IPsec replicates what ZTNA replaces.** A site-to-site IPsec tunnel is network-centric: it grants all hosts on Site-LAN implicit access to the datacenter based on network location. This is precisely the perimeter model that SASE is designed to replace. NetBird already provides a Zero Trust overlay where every peer authenticates individually and ACL policies enforce per-peer access.

**Zscaler alignment.** Descoping aligns the architecture with Zscaler's publicly documented Zero Trust SD-WAN approach: no implicit trust from network location, every site treated as an untrusted network, cloud inspection for all traffic.

**Complexity reduction.** Removing IPsec, uCPE, and QoS reduces the implementation surface and keeps the PoC focused on the SASE security components.

## Related

- [VyOS](../components/vyos.md)
- [NetBird](../components/netbird.md)
- [Zero Trust](../concepts/zero-trust.md)
- [SASE](../concepts/sase.md)
- [Testing: Acceptance Tests](../testing/acceptance-tests.md)
