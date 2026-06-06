---
title: "Finding: FreeBSD cannot kernel-mirror — DC inner-segment visibility is partial"
tags: [finding, zeek, network, gre, opnsense]
---

# Finding: FreeBSD cannot kernel-mirror — DC inner-segment visibility is partial

**Component:** [Zeek](../components/zeek.md)  
**Severity:** Insight (accepted limitation for the PoC)

> Observed on the **parallel stack** during the Zeek/RITA experiment (`mgmt01` =
> `192.168.122.20`, `OPNsense-1` = `192.168.122.11`, DC segment `10.0.0.0/24`). Not yet part of
> the main sandbox.

## What happened

The Site1 segment was mirrored into a GRE tunnel and reached Zeek's `worker-site1` cleanly. The
DC segment was set up the same way: a GRE tunnel from `OPNsense-1` to `mgmt01`, received as
`gre-dc` and fed to `worker-dc`. The tunnel itself came up and Zeek's `tunnel.log` showed
`Tunnel::GRE Tunnel::DISCOVER` for it, but no mirrored frames ever arrived on `worker-dc`. The DC
worker saw the tunnel and nothing inside it.

## Root cause

`OPNsense-1` runs FreeBSD. FreeBSD has no native kernel-level interface mirror equivalent to
VyOS's `set interfaces ethernet ethX mirror ingress/egress`. There is no command to tell the
kernel "copy every frame on `vtnet1` into this GRE tunnel."

The obvious Linux substitute does not port: piping `tcpdump` output into a GRE device fails on
FreeBSD because the GRE tunnel device expects properly GRE-encapsulated packets, not raw
pcap-formatted bytes. So the tunnel exists, but nothing is feeding the inner DC traffic
(`10.0.0.x` source/destination, before routing) into it.

## Resolution / workaround

Accept partial DC visibility for the PoC. DC traffic is still captured at the **core** level by
`worker-core` on `ens4` whenever it transits OPNsense's WAN interface on the core segment. What
is lost is only the inner-segment perspective: `10.0.0.x ↔ 10.0.0.x` conversations and the
pre-routing source IP of DC hosts.

That loss is acceptable here because the DC segment holds exactly two nodes: `OPNsense-1`
(`10.0.0.1`) and `Ubuntu-Server-DC-1` (`10.0.0.100`). Every connection from DC-1 to anywhere
outside the DC passes through OPNsense and becomes visible at the core. There is no second DC
host, so there is no lateral movement to miss. Overall topology coverage was estimated at ~90%,
with the missing ~10% being exactly this DC inner-segment traffic.

The production fix is to replace the frame-delivery mechanism, not Zeek: an ERSPAN-capable
managed switch, or a Linux-based gateway instead of FreeBSD/OPNsense, both of which can do native
kernel mirroring for the DC segment.

## Lessons

- Interface mirroring is OS-specific. VyOS (Linux) mirrors natively; FreeBSD/OPNsense does not.
  Plan the capture method per gateway, not once for the whole topology.
- GRE tunnel discovery in Zeek's `tunnel.log` confirms the tunnel, not the traffic. A
  `DISCOVER` line with no payload on the worker means the far end is not actually mirroring.
- A monitoring gap is only as bad as the topology behind it. With a single-server DC the inner
  blind spot has no practical security impact; in a multi-server DC it would need an ERSPAN
  switch or a Linux gateway.
- This is the counterpart to the working case on the other segment — see the VyOS mirror in
  [Finding: VyOS GRE two-step commit](vyos-gre-two-step-commit.md).

## Related

- [Component: Zeek](../components/zeek.md)
- [Decision: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.md)
- [Finding: VyOS GRE two-step commit](vyos-gre-two-step-commit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
