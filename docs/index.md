---
title: "Project Wiki"
tags: [sase, architecture]
---

# Project Wiki

This wiki documents the implementation of a SASE (Secure Access Service Edge) proof-of-concept for Atlascollege — a school with 4 000 users on managed Windows devices (Intune-managed, Entra joined). The stack replaces a traditional perimeter model with identity-based, context-aware access control using open-source components: NetBird (ZTNA), Squid (SWG), ClamAV + Python DLP (ICAP inspection), Suricata (IDS), ioc2rpz + Unbound (DNS threat intelligence), Entra ID (identity), NATS JetStream (event bus), Wazuh (SIEM), and a Python control daemon (real-time quarantine). All components run on a GNS3 topology hosted on Proxmox.

> All source documents (session reports, addenda, implementation documents) are available on Microsoft Teams: **1A › project-wiki.rar**

[:material-sitemap: Open the functional schema &nearr;](demos/functioneel-schema.html){ .md-button .md-button--primary target=_blank }

An interactive end-to-end diagram of the SASE PoC — every component, traffic flow, and policy
decision point. Opens full-width in a new tab.

## Catalog

| Page | Summary |
|------|---------|
| **Overview** | |
| [Architecture](overview/architecture.md) | Full system: nodes, network segments, traffic flows, inspection pipeline, trust boundaries |
| **Components** | |
| [Squid](components/squid.md) | Explicit HTTP/HTTPS proxy — WPAD/PAC distribution, SSL Bump, URL filtering, ICAP orchestration |
| [ClamAV/c-icap](components/clamav-cicap.md) | Malware scanning + DLP Layer 1 (downloads) via ICAP RESPMOD; YARA rules; StructuredDataDetection |
| [Python DLP](components/python-dlp.md) | Upload DLP (ICAP REQMOD) — Luhn, IBAN mod-97, BSN 11-proof, AWS key detection on multipart bodies |
| [Suricata](components/suricata.md) | IDS on WAN (vtnet0) + LAN (vtnet1); Hyperscan; 79 620+ rules; IPS ready for physical hardware |
| [ioc2rpz](components/ioc2rpz.md) | DNS threat intelligence — URLhaus + ThreatFox → RPZ zone → BIND → Unbound; 71 767 blocked domains |
| [NetBird](components/netbird.md) | WireGuard ZTNA overlay — Zitadel + Entra ID IdP chain; ACL policies; DNS primary nameserver |
| [Caddy](components/caddy.md) | WPAD PAC file server, NetBird TLS terminator, ioc2rpz GUI reverse proxy |
| [GNS3](components/gns3.md) | Lab virtualization platform — nested QEMU/KVM on Proxmox; multi-user; GNS3 vs EVE-NG |
| [VyOS](components/vyos.md) | SASE Gateway for remote site (site01); Zero Trust Branch model; NAT for Site-LAN |
| [Identity Bridge](components/identity-bridge.md) | NetBird overlay-IP → Entra ID group (SWG↔ZTNA coupling) |
| [NATS JetStream](components/nats-jetstream.md) | Central event bus — connects detection silos |
| [Control Daemon](components/control-daemon.md) | Threat scoring + real-time quarantine via NetBird API |
| [Wazuh](components/wazuh.md) | SIEM — NATS forwarder + pop01 agent + M365 Active Response |
| [Zitadel](components/zitadel.md) | OIDC IdP broker — Entra ID → JWT group sync → NetBird |
| **Concepts** | |
| [SASE](concepts/sase.md) | Five SASE pillars; our open-source implementation; commercial equivalents |
| [Zero Trust](concepts/zero-trust.md) | Three-gate model; never trust always verify; Gates 1–3 |
| [ICAP](concepts/icap.md) | REQMOD vs RESPMOD; Squid orchestration; why two ICAP servers |
| [SSL Bump](concepts/ssl-bump.md) | HTTPS interception; SASE-PoC-CA; no-bump list |
| [WPAD/PAC](concepts/wpad-pac.md) | Browser proxy auto-configuration; why transparent proxy fails on wt0 |
| [RPZ](concepts/rpz.md) | DNS Response Policy Zones; zone transfer chain; NXDOMAIN enforcement |
| [DLP](concepts/dlp.md) | Two-layer DLP; algorithmic validation vs pattern matching; upload vs download |
| [Identity Flow](concepts/identity-flow.md) | Full identity chain: Entra ID → Zitadel → NetBird → Identity Bridge → Squid |
| **Decisions** | |
| [WPAD/PAC vs Transparent Proxy](decisions/wpad-vs-transparent-proxy.md) | Why explicit proxy — pf rdr does not work on wt0 |
| [Two-Layer DLP](decisions/two-layer-dlp.md) | ClamAV (downloads) + Python DLP (uploads) — multipart parsing gap |
| [ioc2rpz vs Unbound native](decisions/ioc2rpz-vs-unbound-native.md) | Why ioc2rpz for feed aggregation instead of cron + zone file |
| [BIND as TSIG intermediary](decisions/bind-tsig-intermediary.md) | Unbound 1.24.2 lacks TSIG — BIND bridges the gap |
| [IDS vs IPS](decisions/ids-vs-ips.md) | Netmap IPS fails on virtio NICs — IDS mode with policies ready |
| [Suricata WAN+LAN](decisions/suricata-wan-lan.md) | Why vtnet0 + vtnet1 — wt0 shows 0 packets in BPF |
| [GNS3 vs EVE-NG](decisions/gns3-vs-eveng.md) | Multi-user requirement; QCOW2 format; EVE-NG single-session limitation |
| [Zitadel as IdP broker](decisions/zitadel-idp-broker.md) | Quickstart installs Zitadel; Entra ID as external IdP; CA still fires |
| [CA + Posture hybrid (Three-Gate Model)](decisions/ca-posture-hybrid.md) | Gate 1 (Entra ID CA) + Gate 2 (posture) — complementary not substitutable; managed-device three-gate model |
| [SD-WAN Descoped (F12, F13, F14)](decisions/sdwan-descoped.md) | Classic IPsec + uCPE removed (site-to-site tunnels contradict Zero Trust); QoS + failover reimplemented under the ZT-Branch model; sitepc01 → NetBird enrollment |
| [NetBird Service PAT](decisions/netbird-service-pat.md) | Service-user PAT for Identity Bridge API auth — avoids NetBird issue #3127 |
| [NATS accounts auth](decisions/nats-accounts-auth.md) | `accounts{}` model required for JetStream API access |
| [GroupSync Path B](decisions/groupsync-pad-b.md) | Zitadel strips `2ITCSC1A-` prefix — clean persona group names |
| [CASB three layers](decisions/casb-three-layers.md) | Inline (Squid) + API (Wazuh) + Real-time (NATS+daemon) |
| [Managed devices scope](decisions/managed-devices-scope.md) | BYOD → managed Windows devices (lector mandate R11) |
| [ZT SD-WAN Branch](decisions/zt-sdwan-branch.md) | Zero Trust Branch replaces classic IPsec SD-WAN |
| [Control Daemon scope](decisions/control-daemon-scope.md) | IDS correlation + proxy_block removed from threat scoring |
| **Testing** | |
| [Acceptance Tests (F1–F15)](testing/acceptance-tests.md) | Full F1–F15 status table, test commands, actual output, per-pillar coverage, planned tests |
| [Attack & Bypass Scenarios](testing/attack-scenarios.md) | Demo validation scenarios by SASE pillar: ZTNA, SWG, CASB, FWaaS/IDS, DNS, NATS enforcement |
| [Demo Script (per rubric)](testing/demo-script.md) | Interactive presentation script mapping each rubric criterion to test, proof, screencap, and voiceover |
| **Runbooks** | |
| [Runbooks overview](runbooks/index.md) | Build order, dependency graph, step-by-step deployment guides |
| [01 — Lab Environment](runbooks/01-lab-environment.md) | Proxmox VM, GNS3 Server, topology, IP addressing, snapshots |
| [02 — ZTNA Overlay](runbooks/02-ztna-overlay.md) | NetBird + Zitadel + Entra ID, groups, ACLs, exit node, DNS |
| [03 — Proxy & WPAD](runbooks/03-proxy-wpad.md) | Squid explicit proxy, WPAD/PAC via Caddy, SSL Bump, URL filtering |
| [04 — Malware & DLP](runbooks/04-malware-dlp.md) | ClamAV/c-icap RESPMOD, YARA rules, Python DLP REQMOD |
| [05 — IDS](runbooks/05-ids.md) | Suricata on WAN+LAN, Hyperscan, 79,620+ rules |
| [06 — DNS Threat Intel](runbooks/06-dns-threat-intel.md) | ioc2rpz, BIND TSIG, Unbound RPZ, 71,767 records |
| [07 — Access Policy](runbooks/07-access-policy.md) | Conditional Access, posture checks, validation scenarios (planned) |
| [08 — GroupSync](runbooks/08-groupsync.md) | JWT group sync, Entra ID token config, Zitadel Actions |
| [09 — Identity Bridge](runbooks/09-identity-bridge.md) | FastAPI overlay-IP → persona group, Squid external_acl |
| [10 — NATS JetStream](runbooks/10-nats-jetstream.md) | Event bus, producers, Control Daemon, Redis |
| [11 — Wazuh](runbooks/11-wazuh.md) | SIEM stack, NATS forwarder, M365 Active Response |
| **Findings** | |
| [wt0 pf rdr limitation](findings/wt0-pf-rdr-limitation.md) | WireGuard Layer 3 — pf rdr cannot intercept on wt0 |
| [pre-auth ssl-bump params](findings/pre-auth-ssl-bump-params.md) | Bare http_port without ssl-bump = no inspection on overlay listener |
| [Squid clearlog destroys file](findings/squid-clearlog-destroys-file.md) | WebUI "Clear log" deletes, not truncates; use `> access.log` |
| [StevenBlack incompatible](findings/stevenblack-incompatible.md) | Hosts format incompatible with OPNsense Remote ACL — use UT1 Toulouse |
| [Suricata interface default bug](findings/suricata-interface-default-bug.md) | `interface: default` captures only vtnet0 — explicit declarations required |
| [Suricata Netmap/virtio](findings/suricata-netmap-virtio.md) | Netmap IPS = 0 packets on virtio NICs |
| [Unbound no TSIG](findings/unbound-no-tsig.md) | Unbound 1.24.2 missing TSIG for zone transfers (NLnetLabs issue #336) |
| [Unbound config path](findings/unbound-config-path.md) | `/var/unbound/etc/` is deleted on restart — use `/usr/local/etc/unbound.opnsense.d/` |
| [ioc2rpz GUI JS bug](findings/ioc2rpz-gui-js-bug.md) | Missing e.preventDefault() in signIn — sed fix after container start |
| [pyicap collections bug](findings/pyicap-collections-bug.md) | `collections.Callable` removed in Python 3.10 — sed patch in Dockerfile |
| [iptables FORWARD ordering](findings/iptables-forward-ordering.md) | libvirt REJECT before appended rules — always use `-I FORWARD 1` |
| [NetBird primary nameserver](findings/netbird-primary-nameserver.md) | Empty match-domains required for RPZ to cover external domains |
| [NetBird config zero bytes](findings/netbird-config-zero-bytes.md) | FreeBSD UFS + QEMU SIGKILL = 0-byte config.json — backup after each session |
| [Docker volume recreation](findings/docker-volume-recreation.md) | `docker compose restart` doesn't apply volume mount changes — use `up -d` |
| [curl --ssl-no-revoke on Windows](findings/curl-ssl-no-revoke.md) | Schannel CRL check fails on SASE-PoC-CA — add `--ssl-no-revoke` to all proxy tests from Windows |
| [Suricata connection pooling](findings/suricata-connection-pooling.md) | Squid reuses upstream TCP connections — one Suricata alert per SID per flow is correct, not suppression |
| [NetBird issue #3127](findings/netbird-issue-3127.md) | Management API rejects peer-level group changes — service-user PAT as workaround |
| [Squid overlay bind race](findings/squid-overlay-bind-race.md) | Squid overlay listener lost on restart — `commBind EADDRNOTAVAIL` race |
| [Overlay IP instability](findings/overlay-ip-instability.md) | NetBird overlay IPs can change — Identity Bridge must handle stale cache |
| [NATS store dir](findings/nats-store-dir.md) | JetStream store must be on persistent Docker volume |
| [Wazuh CPU glibc](findings/wazuh-cpu-glibc.md) | CPU spike on startup due to glibc compatibility |
| [Wazuh dashboard airgate](findings/wazuh-dashboard-airgate.md) | Dashboard Offline caused by empty UUID after down -v — resolved, in-place 4.14.5 GA bump |
| [NetBird JWT allow-groups lockout](findings/netbird-jwt-allow-groups-lockout.md) | Enabling JWT allow-groups can lock out all users if misconfigured |
| [DC-LAN isolation route ACL](findings/dc-lan-isolation-route-acl.md) | NetBird Networks + ACL required for DC-LAN isolation |

---

## Quick reference

**Key ports:**

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

**Key IPs:**

| Node | WAN | Overlay |
|------|-----|---------|
| pop01 | `192.168.122.13` | `100.70.154.79` |
| mgmt01 | `192.168.122.23` | `100.70.135.241` |
| mobile01 | — | `100.70.95.98` |
| dc01 | — | `10.0.0.100` |
| GNS3 host | `10.158.10.67` | — |

**Gate status:**

| Gate | Status | Technology |
|------|--------|-----------|
| Gate 1 — Identity | ✅ Operational | Entra ID Conditional Access (5 policies) |
| Gate 2 — Device | ✅ Operational | Intune device compliance (Report-only until demo) |
| Gate 3 — Content | ✅ Operational | Squid + ClamAV + Python DLP + Unbound RPZ + Suricata |
