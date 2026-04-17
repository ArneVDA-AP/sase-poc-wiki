---
title: "Project Wiki"
tags: [sase, architecture]
---

# Project Wiki

This wiki documents the implementation of a SASE (Secure Access Service Edge) proof-of-concept for Atlascollege — a school with 4000+ BYOD students. The stack replaces a traditional perimeter model with identity-based, context-aware access control using open-source components: NetBird (ZTNA), Squid (SWG), ClamAV + Python DLP (ICAP inspection), Suricata (IDS), ioc2rpz + Unbound (DNS threat intelligence), and Entra ID (identity). All components run on a GNS3 topology hosted on Proxmox.

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
| [VyOS](components/vyos.md) | SD-WAN gateway for remote site (site01); NAT for Site-LAN |
| **Concepts** | |
| [SASE](concepts/sase.md) | Five SASE pillars; our open-source implementation; commercial equivalents |
| [Zero Trust](concepts/zero-trust.md) | Three-gate model; never trust always verify; Gates 1–3 |
| [ICAP](concepts/icap.md) | REQMOD vs RESPMOD; Squid orchestration; why two ICAP servers |
| [SSL Bump](concepts/ssl-bump.md) | HTTPS interception; SASE-PoC-CA; no-bump list |
| [WPAD/PAC](concepts/wpad-pac.md) | Browser proxy auto-configuration; why transparent proxy fails on wt0 |
| [RPZ](concepts/rpz.md) | DNS Response Policy Zones; zone transfer chain; NXDOMAIN enforcement |
| [DLP](concepts/dlp.md) | Two-layer DLP; algorithmic validation vs pattern matching; upload vs download |
| **Decisions** | |
| [WPAD/PAC vs Transparent Proxy](decisions/wpad-vs-transparent-proxy.md) | Why explicit proxy — pf rdr does not work on wt0 |
| [Two-Layer DLP](decisions/two-layer-dlp.md) | ClamAV (downloads) + Python DLP (uploads) — multipart parsing gap |
| [ioc2rpz vs Unbound native](decisions/ioc2rpz-vs-unbound-native.md) | Why ioc2rpz for feed aggregation instead of cron + zone file |
| [BIND as TSIG intermediary](decisions/bind-tsig-intermediary.md) | Unbound 1.24.2 lacks TSIG — BIND bridges the gap |
| [IDS vs IPS](decisions/ids-vs-ips.md) | Netmap IPS fails on virtio NICs — IDS mode with policies ready |
| [Suricata WAN+LAN](decisions/suricata-wan-lan.md) | Why vtnet0 + vtnet1 — wt0 shows 0 packets in BPF |
| [GNS3 vs EVE-NG](decisions/gns3-vs-eveng.md) | Multi-user requirement; QCOW2 format; EVE-NG single-session limitation |
| [Zitadel as IdP broker](decisions/zitadel-idp-broker.md) | Quickstart installs Zitadel; Entra ID as external IdP; CA still fires |
| [CA + Posture hybrid (Three-Gate Model)](decisions/ca-posture-hybrid.md) | Gate 1 (Entra ID CA) + Gate 2 (posture) — complementary not substitutable; BYOD Intune gap |
| [SD-WAN Descoped (F12, F13, F14)](decisions/sdwan-descoped.md) | IPsec + QoS + uCPE removed — site-to-site tunnels contradict Zero Trust; sitepc01 → NetBird enrollment |
| **Testing** | |
| [Acceptance Tests (F1–F15)](testing/acceptance-tests.md) | Full F1–F15 status table, test commands, actual output, per-pillar coverage, planned tests |
| **Runbooks** | |
| [Runbooks overview](runbooks/index.md) | Build order, dependency graph, step-by-step deployment guides |
| [01 — Lab Environment](runbooks/01-lab-environment.md) | Proxmox VM, GNS3 Server, topology, IP addressing, snapshots |
| [02 — ZTNA Overlay](runbooks/02-ztna-overlay.md) | NetBird + Zitadel + Entra ID, groups, ACLs, exit node, DNS |
| [03 — Proxy & WPAD](runbooks/03-proxy-wpad.md) | Squid explicit proxy, WPAD/PAC via Caddy, SSL Bump, URL filtering |
| [04 — Malware & DLP](runbooks/04-malware-dlp.md) | ClamAV/c-icap RESPMOD, YARA rules, Python DLP REQMOD |
| [05 — IDS](runbooks/05-ids.md) | Suricata on WAN+LAN, Hyperscan, 79,620+ rules |
| [06 — DNS Threat Intel](runbooks/06-dns-threat-intel.md) | ioc2rpz, BIND TSIG, Unbound RPZ, 71,767 records |
| [07 — Access Policy](runbooks/07-access-policy.md) | Conditional Access, posture checks, validation scenarios (planned) |
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

---

## Quick reference

**Key ports:**

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
| Gate 1 — Identity | ⚠️ Planned | Entra ID Conditional Access (4 policies in Addendum E) |
| Gate 2 — Device | ⚠️ Planned | NetBird Posture Checks (Addendum E) |
| Gate 3 — Content | ✅ Operational | Squid + ClamAV + Python DLP + Unbound RPZ + Suricata |
