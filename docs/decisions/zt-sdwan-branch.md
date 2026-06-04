---
title: "Decision: Zero Trust Branch Instead of Classic IPsec SD-WAN"
tags: [decision, sd-wan, vyos, netbird, zero-trust]
---

# Decision: Zero Trust Branch Instead of Classic IPsec SD-WAN

**Status:** Implemented  
**Date:** May-June 2026 (Verslag41, Verslag43)

## Context

Traditional SD-WAN (IPsec site-to-site tunnels, uCPE, QoS) was initially part of the architecture. Herziening v3 (March 2026) descoped the classic IPsec/uCPE approach (see [Decision: SD-WAN Descoped](sdwan-descoped.md)). The question remained: what replaces the branch connectivity model? Simply enrolling each device individually via NetBird is functional but does not address QoS or failover — two capabilities that SD-WAN traditionally provides.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Classic IPsec-based SD-WAN** | Traditional approach; site-to-site tunnel provides transparent subnet access; well-understood QoS models | Grants implicit subnet-level access based on network location — contradicts Zero Trust. Duplicates the ZTNA overlay. F12-F14 test framework measures the wrong paradigm |
| **Zero Trust Branch (NetBird overlay + VyOS QoS)** | Per-device authentication; no implicit subnet trust; aligns with Zscaler/Netskope ZT-SD-WAN model; VyOS provides QoS via DSCP without IPsec | VyOS QoS is applied at the site gateway, not end-to-end; failover is detection-and-alerting only (single-WAN lab), not an automatic dual-WAN switch |

## Decision

Zero Trust Branch model: VyOS site01 operates as the SASE Gateway on Site-LAN (172.16.10.0/24), the NetBird overlay serves as the transport layer, and VyOS applies DSCP EF marking for QoS classification via `tc` on eth0. This aligns with the Zscaler/Netskope Zero Trust SD-WAN model where branches are treated as untrusted networks and every device authenticates individually.

## Evidence

The ZT-Branch implementation was validated through two dedicated tests:

- **Test #5 (QoS under load, V43):** DSCP EF traffic: 300/300 packets, 0 drops. Bulk traffic under same load: 26 drops + 17,000+ overlimits. Demonstrates effective traffic prioritization without IPsec tunnels.
- **Test #6 (Failover detection, V43):** a VyOS health-check script pinging pop01, the internet gateway, and `8.8.8.8` produced a CRITICAL detection within 30 seconds when pop01's interface was taken down, with recovery logged once it returned. This is honest single-WAN framing — detection and alerting, not an automatic dual-WAN switch.

Spoor-1-contract acceptance criteria B1-B4 are fully closed based on these results.

## Consequences

- **F12-F14 (classic IPsec tests) are N/A.** These tests measured a different paradigm. The ZT-Branch tests (#5 and #6) are the correct validation framework for the implemented architecture.
- **VyOS role is redefined:** VyOS site01 is a SASE Gateway with DSCP marking, not an IPsec router. Its configuration is `tc`-based QoS + NAT, not IPsec + BGP.
- **Per-device authentication is mandatory:** Every device on Site-LAN must have its own NetBird enrollment. There is no site-level trust — a compromised device on Site-LAN does not grant access to datacenter resources for other devices.
- **QoS is site-egress only:** DSCP marking at VyOS applies to traffic leaving the site. End-to-end QoS depends on intermediate network hops honoring DSCP markings, which is not guaranteed on the public internet.

See also: [Decision: SD-WAN Descoped](sdwan-descoped.md), [Component: VyOS](../components/vyos.md), [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)
