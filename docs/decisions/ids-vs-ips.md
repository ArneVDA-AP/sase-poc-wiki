---
title: "Decision: IDS Mode vs IPS Mode for Suricata"
tags: [decision, suricata, network, opnsense]
---

# Decision: IDS Mode vs IPS Mode for Suricata

**Status:** Implemented (IDS mode)  
**Date:** April 2026 (Verslag23)

## Context

Suricata can operate in two modes:
- **IDS (Intrusion Detection System):** PCAP capture — reads packet copies passively, generates alerts, cannot drop traffic
- **IPS (Intrusion Prevention System):** Inline — sits in the traffic path, can drop packets in real time

OPNsense offers two IPS activation paths: Netmap (native kernel bypass) and Divert (pf divert-to rules).

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **IDS (PCAP mode)** | Works on virtio NICs; no NIC driver requirements; stable | Cannot drop traffic in real time; threat is detected, not blocked |
| **IPS via Netmap** | Real packet dropping; lowest latency inline mode | Requires NIC drivers with native Netmap support (Intel igb/ixgbe, Broadcom bge). QEMU virtio NICs have no Netmap support. When enabled: Suricata processes 0 packets. |
| **IPS via Divert** | Does not require special NIC drivers | Requires explicit `pf divert-to` firewall rules; not configured in this stack |

## Decision

IDS mode (PCAP), with the drop/alert policy table configured and ready for IPS activation when deployed on physical hardware.

Netmap IPS was tested and confirmed non-functional on virtio NICs. When Netmap IPS was activated: Suricata reported 0 packets processed, and all previous alert activity stopped. Reverting to PCAP IDS restored normal operation. (Verslag23, Bevinding 23.3/23.4/23.5)

The differentiated policy table (drop vs. alert by category) is configured in OPNsense. On physical hardware with supported NICs, switching to IPS mode requires only enabling the IPS toggle — the policies are already correct.

## Consequences

- Suricata detects and alerts, but does not block traffic in real time in the current sandbox
- Drop-category threats (emerging-malware, botcc, Abuse.ch) appear in logs; blocking only occurs via Gate 3 (Unbound RPZ for DNS, ClamAV for downloads, Python DLP for uploads)
- `procstat -f <PID> | grep bpf` is the correct way to verify Suricata is capturing — `sockstat` shows nothing because Suricata uses BPF, not TCP/UDP sockets
- Switching to IPS on physical hardware requires `pf divert-to` rules or native Netmap NIC drivers

See also: [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md), [Decision: Suricata WAN+LAN](suricata-wan-lan.md)
