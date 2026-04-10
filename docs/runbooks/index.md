---
title: "Runbooks"
tags: [runbook]
---

# Runbooks

Step-by-step deployment guides for building the SASE PoC stack from scratch. Each runbook restores the linear build narrative from the original implementation documents, with warnings and gotchas placed exactly at the step where they occur.

**How to use:** Follow the runbooks in order. Each runbook states its prerequisites — which prior runbooks must be completed first. Every step ends with a verification so you know when to proceed. For deeper conceptual understanding, follow the links to the corresponding component, concept, and decision pages.

---

## Build order

```
01 Lab Environment
 └─► 02 ZTNA Overlay
      ├─► 03 Proxy & WPAD ─► 04 Malware & DLP
      │                      05 IDS (parallel with 04)
      └─► 06 DNS Threat Intel
           └─► 07 Access Policy (after all above)
```

## Runbook catalog

| # | Runbook | Node(s) | Status | Description |
|---|---------|---------|--------|-------------|
| 1 | [Lab Environment](01-lab-environment.md) | GNS3 host (poc-1a) | Operational | Proxmox VM, GNS3 Server, topology, IP addressing, snapshots |
| 2 | [ZTNA Overlay](02-ztna-overlay.md) | mgmt01 + all peers | Operational | NetBird + Zitadel + Entra ID, groups, ACLs, exit node, DNS |
| 3 | [Proxy & WPAD](03-proxy-wpad.md) | pop01 + mgmt01 | Operational | Squid explicit proxy, WPAD/PAC via Caddy, SSL Bump, URL filtering |
| 4 | [Malware & DLP](04-malware-dlp.md) | pop01 + mgmt01 | Operational | ClamAV/c-icap RESPMOD, YARA rules, Python DLP REQMOD |
| 5 | [IDS](05-ids.md) | pop01 | Operational | Suricata on WAN+LAN, Hyperscan, 79,620+ rules, drop/alert policies |
| 6 | [DNS Threat Intel](06-dns-threat-intel.md) | mgmt01 + pop01 | Operational | ioc2rpz feeds, BIND TSIG intermediary, Unbound RPZ, 71,767 records |
| 7 | [Access Policy](07-access-policy.md) | Entra ID + NetBird | Planned | Conditional Access (4 policies), posture checks, five validation scenarios |

## Dependency graph

| Runbook | Depends on |
|---------|------------|
| 01 Lab Environment | — |
| 02 ZTNA Overlay | 01 |
| 03 Proxy & WPAD | 02 |
| 04 Malware & DLP | 03 (SSL Bump required) |
| 05 IDS | 03 (Squid operational), 01 (8 GB RAM) |
| 06 DNS Threat Intel | 02 (NetBird DNS relay) |
| 07 Access Policy | 02, 03, 04, 05, 06 (full stack) |

## Quick reference — key ports

| Service | Address | Port |
|---------|---------|------|
| Squid proxy (BYOD) | `100.70.154.79` | 3128 |
| Unbound DNS | `100.70.154.79` | 53 |
| BIND (TSIG secondary) | `127.0.0.1` | 53530 |
| ioc2rpz | `192.168.122.23` | 53 |
| Python DLP ICAP | `192.168.122.23` | 1345 |
| ClamAV/c-icap | `127.0.0.1` | 1344 |
| NetBird Dashboard | `netbird.sandbox.local` | 443 |
| ioc2rpz GUI | `ioc2rpz.sandbox.local` | 443 |
| WPAD PAC file | `wpad.sandbox.local` | 80 |
