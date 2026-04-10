---
title: "Decision: Suricata on WAN + LAN (not wt0)"
tags: [decision, suricata, network, opnsense]
---

# Decision: Suricata on WAN + LAN (not wt0)

**Status:** Implemented  
**Date:** March/April 2026 (Verslag22–23)

## Context

Suricata can be configured to monitor specific network interfaces. The stack has three interfaces on pop01: `vtnet0` (WAN), `vtnet1` (LAN/DC-LAN), and `wt0` (NetBird WireGuard overlay). Which interfaces should Suricata monitor?

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **vtnet0 (WAN) only** | Sees all internet-bound traffic from pop01 | Misses internal DC-LAN traffic (lateral movement, dc01 activity) |
| **vtnet0 + vtnet1 (WAN + LAN)** | Covers both internet-facing and internal segments | Does not see decrypted BYOD traffic (WireGuard payload is not visible) |
| **vtnet0 + vtnet1 + wt0** | Maximum coverage | wt0 is a WireGuard tunnel; BPF on wt0 sees 0 TCP packets — WireGuard traffic does not appear as ingress frames from pf's perspective |

## Decision

Suricata on vtnet0 (WAN) and vtnet1 (LAN) only.

wt0 was tested: `tcpdump -i wt0 -n 'tcp'` showed 0 packets when mobile01 was active. WireGuard is a Layer 3 VPN — traffic routed via wt0 does not appear as ingress frames on that interface from BPF's perspective. Suricata on wt0 would see nothing.

vtnet1 (LAN) was added to detect threats on the internal DC-LAN segment — lateral movement, anomalous behavior from dc01 itself (e.g., software update traffic). This covers the "assume breach" zero-trust principle: internal traffic is not trusted by default.

## Consequences

- Suricata requires explicit per-interface declarations in `custom.yaml` — using `interface: default` only captures vtnet0, not vtnet1. See [Finding: Suricata interface default bug](../findings/suricata-interface-default-bug.md)
- BYOD client traffic (decrypted by Squid) is visible on vtnet0 as re-encrypted TLS sessions from Squid to origin servers — Suricata extracts TLS metadata (SNI, JA3/JA4) from these
- WireGuard traffic from mobile01 appears as encrypted UDP/51820 on vtnet0 — the payload is not inspectable by Suricata
- `HOME_NET` must include `100.64.0.0/10` (NetBird overlay) so rules using `$HOME_NET` do not misclassify NetBird peers as external hosts
