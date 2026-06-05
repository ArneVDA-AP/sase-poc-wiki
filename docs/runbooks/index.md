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
      ├─► 06 DNS Threat Intel
      ├─► 08 GroupSync (after 02)
      │    └─► 09 Identity Bridge (after 08)
      ├─► 10 NATS JetStream (after 03, 04, 05, 06)
      │    └─► 11 Wazuh (after 10)
      ├─► 07 Access Policy (after all above)
      └─► 12 Intune Endpoint (after 07)
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
| 7 | [Access Policy](07-access-policy.md) | Entra ID + NetBird | Operational | Conditional Access (5 policies), posture checks, validation scenarios |
| 8 | [GroupSync](08-groupsync.md) | Entra ID + Zitadel + NetBird | Operational | JWT group sync, Entra ID token config, Zitadel Actions |
| 9 | [Identity Bridge](09-identity-bridge.md) | mgmt01 + pop01 | Operational | FastAPI overlay-IP → persona group, Squid external_acl |
| 10 | [NATS JetStream](10-nats-jetstream.md) | mgmt01 + pop01 | Operational | Event bus, producers, Control Daemon, Redis |
| 11 | [Wazuh](11-wazuh.md) | mgmt01 + pop01 | Operational | SIEM stack, NATS forwarder, M365 Active Response |
| 12 | [Intune Endpoint](12-intune-endpoint.md) | Entra ID + Intune | Implemented | Cert, forced-PAC, firewall block-set, Teams QoS, split-tunnel Remediation; SITE01 enrollment |

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
| 08 GroupSync | 02 |
| 09 Identity Bridge | 08 (GroupSync complete) |
| 10 NATS JetStream | 03, 04, 05, 06 (all producers operational) |
| 11 Wazuh | 10 (NATS operational) |
| 12 Intune Endpoint | 07 (managed device Entra-joined + Intune-enrolled) |

## Quick reference — key ports

| Service | Address | Port |
|---------|---------|------|
| Squid proxy | `100.70.154.79` | 3128 |
| Unbound DNS | `100.70.154.79` | 53 |
| BIND (TSIG secondary) | `127.0.0.1` | 53530 |
| ioc2rpz | `192.168.122.23` | 53 |
| Python DLP ICAP | `192.168.122.23` | 1345 |
| ClamAV/c-icap | `127.0.0.1` | 1344 |
| NetBird Dashboard | `netbird.sandbox.local` | 443 |
| ioc2rpz GUI | `ioc2rpz.sandbox.local` | 443 |
| WPAD PAC file | `wpad.sandbox.local` | 80 |
