---
title: "Decision: GNS3 Hub vs Switch for Full Traffic Visibility"
tags: [decision, zeek, network, gns3, architecture]
---

# Decision: GNS3 Hub vs Switch for Full Traffic Visibility

**Status:** Implemented (PoC-validated on the parallel stack — sandbox integration pending)  
**Date:** April 2026 (Zeek/RITA implementation report)

> **Scope.** This decision belongs to the Zeek/RITA experiment built on the **parallel stack**
> (`mgmt01` = `192.168.122.20`). It is not yet wired into the main sandbox topology.

## Context

Zeek needs to see every frame on a segment to do behavioral analysis. A monitor NIC on
`mgmt01` only produces useful output if every frame on the core segment actually reaches that
NIC. In GNS3 the core segment was originally an Ethernet Switch node, and the switch turned out
to be the wrong building block for a monitoring vantage point.

GNS3's built-in Ethernet Switch is `ubridge`, a userspace learning L2 bridge. Once it has
learned MAC addresses it forwards unicast frames only to the destination port, exactly like a
production switch. A monitor port on that switch therefore sees broadcast and its own traffic,
but not the unicast conversations between other nodes. The deciding constraint: `ubridge` has no
CLI, no management plane, and no SPAN / port-mirroring feature. The capability does not exist in
the codebase, so there is no way to ask the switch to copy traffic to the monitor port.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **GNS3 Ethernet Switch (ubridge)** | Realistic L2 behavior; the default node | No SPAN / port mirror; monitor port sees no inter-node unicast. Zeek goes blind to everything except broadcast. |
| **GNS3 Hub** | Floods every frame to every port by design, so the monitor port receives all traffic; zero config | Not how a production network behaves; introduces flooding overhead (negligible at lab traffic levels of kbit/s to low Mbit/s) |
| **Open vSwitch (OVS) VM** | Real L2 switching plus native SPAN via `ovs-vsctl` | Extra VM to deploy and maintain; overkill at this scale |

## Decision

Replace the core Ethernet Switch with a GNS3 **Hub** (`Hub1`).

A Hub floods every frame to every port, which means the monitor NIC on `mgmt01` (`ens4`,
connected to `Hub1` port `Ethernet5`) receives a copy of every conversation on the core segment.
This is the functional equivalent of a SPAN / mirror port on a managed switch: all traffic is
visible to the monitoring port without any switch configuration, because there is nothing to
configure. At the lab's traffic volume the flooding overhead is not measurable.

OVS was documented as the production-grade refinement (real switching with native SPAN), but it
was not implemented because the Hub achieves the same monitoring result at this scale with far
less moving parts.

## Consequences

- The core segment is no longer a faithful L2 switch. For a behavioral-analysis PoC that is the
  point: full visibility beats realistic forwarding.
- The monitor NIC must run in promiscuous mode with NIC offloading disabled, otherwise the
  kernel hands Zeek reassembled jumbo frames instead of wire frames. See the
  [Zeek & RITA runbook](../runbooks/14-zeek-rita.md) for the `ens4` setup.
- This solves visibility for the **core** segment only. Remote segments sit behind their own
  GNS3 switches and need a different mechanism (kernel mirror into a GRE tunnel). The DC segment
  hits a hard limit there, documented in
  [Finding: DC inner-segment mirror limit](../findings/dc-segment-mirror-limit.md).
- A production deployment would swap the Hub for an OVS or ERSPAN-capable managed switch; the
  Zeek side does not change, only the frame-delivery mechanism.

## Related

- [Component: Zeek](../components/zeek.md)
- [Component: GNS3](../components/gns3.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
- [Finding: DC inner-segment mirror limit](../findings/dc-segment-mirror-limit.md)
