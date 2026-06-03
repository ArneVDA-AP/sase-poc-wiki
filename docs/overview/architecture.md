---
title: "System Architecture"
tags: [sase, architecture, network, opnsense, netbird, squid, suricata, zero-trust]
---

# System Architecture

This page describes the complete architecture of the SASE PoC: its nodes, network segments, the control/data plane split, and how all security components relate to each other. Every component page links back here for architectural context.

---

## 1. Design Philosophy: Castle & Moat → Edge Enforcement

The project rejects the traditional perimeter model where trust derives from network position. In the traditional model a firewall protects one boundary; everyone inside is trusted. This fails for Atlascollege: 4 000 users on managed Windows devices connecting from home, cafés, trains — there is no "inside" anymore.

Instead, inspection and trust decisions happen **at the PoP** (Point of Presence) — the edge node that sits between clients and the internet. Trust is determined by three factors evaluated independently:

1. **Identity** — who are you? (Entra ID + MFA)
2. **Device posture** — what is your device's state? (Intune compliance + Entra ID CA)
3. **Content** — what are you transferring? (SWG inspection pipeline)

This is the [Zero Trust three-gate model](../concepts/zero-trust.md).

---

## 2. Nodes and Their Roles

| Node | Platform | Role | IP (WAN/mgmt) | IP (overlay) |
|------|----------|------|----------------|--------------|
| **pop01** | OPNsense 25.1 (FreeBSD) | Data plane PoP: firewall, proxy, IDS, DNS resolver | `192.168.122.13` | `100.70.154.79` |
| **mgmt01** | Ubuntu 24.04 (Docker) | Management/control plane: NetBird stack, ioc2rpz, DLP ICAP, WPAD | `192.168.122.23` | `100.70.135.241` |
| **dc01** | Ubuntu 24.04 | Datacenter simulation — internal resource target | `10.0.0.100` (DC-LAN) | — |
| **site01** | VyOS | SASE Gateway for remote site — WAN connectivity + NAT | `192.168.122.33` | — |
| **sitepc01** | Windows 11 (Tiny11) | Remote site endpoint — dual-NIC on Site-LAN, sandbox-enrolled | `172.16.10.x` | — |
| **mobile01** | Windows 11 (VMware, off-GNS3) | Managed client — simulates Intune-enrolled device from any location | external | `100.70.95.98` |

All nodes except mobile01 run inside GNS3 on a Proxmox-hosted Ubuntu VM. See [GNS3 / Topology](../components/gns3.md) for the full infrastructure layer.

---

## 3. Network Segments

| Segment | CIDR | Purpose | Connected Nodes |
|---------|------|---------|-----------------|
| **WAN / management** | `192.168.122.0/24` | GNS3 libvirt NAT; underlay for all inter-node traffic; GNS3 host internet | pop01, mgmt01, site01 (all via vtnet0/ens3) |
| **DC-LAN** | `10.0.0.0/24` | Internal datacenter segment — simulates on-premise resources | pop01 (vtnet1 = gateway `.1`), dc01 (`.100`) |
| **Site-LAN** | `172.16.10.0/24` | Remote site segment | site01 (eth1 = gateway `.1`), sitepc01 (`.50`, no OS) |
| **NetBird Overlay** | `100.64.0.0/10` (WireGuard mesh) | Encrypted overlay — client transport for all SASE traffic | pop01, mgmt01, mobile01 |

mobile01 is **not** on the WAN segment. It connects exclusively via the NetBird WireGuard overlay, simulating a genuine remote user with no direct path to the internal network.

---

## 4. Control Plane vs Data Plane

A deliberate architectural split mirrors commercial SASE:

**Management plane (mgmt01):**
- NetBird management stack (Zitadel OIDC, management server, Caddy reverse proxy) — via Docker Compose
- Identity Bridge (FastAPI) — maps overlay IPs to Entra ID persona groups for Squid
- NATS JetStream — central event bus connecting all detection silos
- Control Daemon — consumes NATS events, per-peer threat scoring, quarantine via NetBird API
- Wazuh (manager + indexer + dashboard) — SIEM with NATS→Wazuh forwarder
- Redis — threat score store + session state for control daemon
- ioc2rpz — threat intelligence aggregation and RPZ zone publication
- Caddy — WPAD server, ioc2rpz GUI proxy
- Python DLP ICAP server — upload scanning on port `192.168.122.23:1345`

**Data plane (pop01):**
- OPNsense firewall and routing
- Squid explicit proxy (pre-auth listener `100.70.154.79:3128`)
- SSL Bump (SASE-PoC-CA)
- URL filtering (UT1 Toulouse Remote ACL + manual blacklist)
- ClamAV + c-icap (malware scanning + DLP layer 1, ICAP RESPMOD on `127.0.0.1:1344`)
- Unbound DNS resolver (RPZ enabled via `respip` module)
- BIND 9.20 (os-bind, non-resolving secondary for TSIG-authenticated zone transfer, port `53530`)
- Suricata IDS (vtnet0 WAN + vtnet1 LAN, PCAP mode)
- NetBird agent (exit node, routing peer, WPAD listener)

---

## 5. Traffic Flows

### 5.1 Client HTTP/HTTPS traffic (the main inspection path)

```
mobile01
  │ ① DNS lookup: wpad.sandbox.local → 100.70.135.241 (mgmt01)
  │   (via NetBird primary nameserver → pop01 Unbound → RPZ check → resolved)
  │
  │ ② PAC file fetched: http://wpad.sandbox.local/wpad.dat (Caddy on mgmt01)
  │   Result: PROXY 100.70.154.79:3128 for all external traffic
  │
  │ ③ Browser sends CONNECT to pop01 Squid on 100.70.154.79:3128 (WireGuard tunnel)
  │
  ▼
pop01 Squid (pre-auth listener, ssl-bump)
  │
  ├── Identity Bridge lookup: external_acl queries mgmt01 with overlay IP (%SRC)
  │     → returns Entra ID persona group (Studenten/Docenten/Admins)
  │     → persona-ACL determines policy decision (identity-based filtering)
  ├── URL filter check (UT1 / manual blacklist) → 403 if blocked
  ├── ICAP REQMOD → Python DLP (mgmt01:1345) → 403 if sensitive data in upload
  ├── SSL Bump: TLS termination + re-encryption (SASE-PoC-CA)
  └── ICAP RESPMOD → ClamAV/c-icap (pop01:1344) → 403 if malware or DLP match
  │
  ▼
Internet (via pop01 vtnet0 WAN)
  │
  └── Suricata IDS on vtnet0: sees re-encrypted upstream connections
        TLS metadata, DNS anomalies, C2 signatures, suspicious UA
```

### 5.2 DNS resolution (threat intelligence path)

```
mobile01 or dc01
  │ DNS query (all queries — via NetBird primary nameserver)
  ▼
pop01 Unbound (port 53, respip module active)
  │ RPZ check: is the domain in threat-intel.rpz.sase?
  ├── Yes → NXDOMAIN (authoritative, aa flag)
  └── No  → forward to upstream DNS → response returned
```

### 5.3 Zone transfer chain (threat intelligence update path)

```
URLhaus + ThreatFox feeds
  ↓ HTTP pull (periodic)
ioc2rpz (mgmt01 Docker, 192.168.122.23:53)
  ↓ TSIG-authenticated AXFR (tkey_rpz_transfer, hmac-sha256)
BIND 9.20 (pop01, 127.0.0.1:53530, secondary zone)
  ↓ unauthenticated AXFR (loopback, no auth needed)
Unbound RPZ (pop01, zone loaded in-memory via respip)
```

### 5.4 ZTNA tunnel-establishment (identity verification path)

```
netbird up (mobile01)
  │ OIDC flow → Zitadel (mgmt01) → Entra ID (aplab.be tenant)
  │ Gate 1: Entra ID Conditional Access evaluates (✅ operational)
  │   - CA Policy 1: MFA required ✅
  │   - CA Policy 2: Geo-block (Belgium only) — Report-only
  │   - CA Policy 3: Legacy auth blocked ✅
  │   - CA Policy 4: Risk-based block ✅
  │   - CA Policy 5: Compliant device required — Report-only
  │ OIDC token received
  │
  │ Gate 2: Intune device compliance evaluates (✅ operational)
  │   - OS version, Defender AV + firewall, real-time protection
  │   - mobile01 enrolled as 2ITCSC1A-MOB-1, compliant
  │ WireGuard tunnel negotiation with pop01
  │ WireGuard tunnel active, ACL policies applied
  ▼
pop01 (wt0 interface, overlay IP 100.70.154.79)
  Client traffic now flows through inspection pipeline (§5.1)
```

---

## 6. The Four-Layer Inspection Pipeline

All HTTP/HTTPS traffic from managed clients passes through four distinct inspection layers. The layers are **complementary**, not redundant — each inspects a domain the others cannot see:

| Layer | Component | Location | Inspects | Protocol |
|-------|-----------|----------|----------|----------|
| 1 | Squid + URL filtering | pop01 | URLs, categories, HTTPS connection setup | Application |
| 2 | ClamAV + c-icap + YARA + SDD | pop01:1344 | Downloaded file content (malware + DLP layer 1) | ICAP RESPMOD |
| 3 | Python DLP ICAP | mgmt01:1345 | Uploaded POST/PUT/PATCH bodies (DLP layer 2) | ICAP REQMOD |
| 4 | Suricata IDS | pop01 vtnet0+vtnet1 | Network flows, TLS metadata, DNS, raw packets | Raw pcap |

Layers 1–3 are sequential on each HTTP transaction. Layer 4 runs in parallel on raw network interfaces, independent of the proxy pipeline.

**Why Suricata is NOT in the ICAP pipeline:** Suricata is a packet-flow engine that reassembles TCP streams and correlates patterns across connections. An HTTP body delivered via ICAP loses that context. The split is architecturally fundamental, not a PoC limitation. See [Decision: Suricata WAN+LAN](../decisions/suricata-wan-lan.md).

---

## 7. Component Map

| Component | Page | Role | Status |
|-----------|------|------|--------|
| Squid + WPAD/PAC + SSL Bump | [squid.md](../components/squid.md) | SWG proxy, HTTPS inspection, URL filtering | ✅ Operational |
| ClamAV + c-icap | [clamav-cicap.md](../components/clamav-cicap.md) | Malware scanning + DLP layer 1 (downloads) | ✅ Operational |
| Python DLP ICAP | [python-dlp.md](../components/python-dlp.md) | DLP layer 2 (uploads/POST bodies) | ✅ Operational |
| Suricata IDS | [suricata.md](../components/suricata.md) | Network-level threat detection | ✅ Operational |
| ioc2rpz + BIND + Unbound RPZ | [ioc2rpz.md](../components/ioc2rpz.md) | DNS threat intelligence | ✅ Operational |
| NetBird + Zitadel + Entra ID | [netbird.md](../components/netbird.md) | ZTNA transport layer | ✅ Operational |
| Caddy | [caddy.md](../components/caddy.md) | WPAD server + reverse proxy | ✅ Operational |
| GNS3 / Topology | [gns3.md](../components/gns3.md) | Virtualisation infrastructure | ✅ Operational |
| VyOS | [vyos.md](../components/vyos.md) | SASE Gateway — Zero Trust Branch model (Site-LAN) | ✅ Operational |
| Identity Bridge | [identity-bridge.md](../components/identity-bridge.md) | NetBird overlay-IP → Entra ID group (SWG↔ZTNA coupling) | ✅ Operational |
| NATS JetStream | [nats-jetstream.md](../components/nats-jetstream.md) | Central event bus — connects detection silos | ✅ Operational |
| Control Daemon | [control-daemon.md](../components/control-daemon.md) | Threat scoring + real-time quarantine via NetBird API | ✅ Operational |
| Wazuh | [wazuh.md](../components/wazuh.md) | SIEM — NATS forwarder + pop01 agent | ✅ Operational |
| Entra ID CA + Intune | [ca-posture-hybrid.md](../decisions/ca-posture-hybrid.md) | Context-aware access (Gates 1+2) | ✅ Operational |
| M365 Activity API + Wazuh AR | [wazuh.md](../components/wazuh.md) | CASB Layer 2 — API-mode enforcement | ✅ Operational |

---

## 8. Trust Boundaries

| Boundary | What it enforces | Where |
|----------|-----------------|-------|
| WireGuard tunnel establishment | Identity (OIDC) + device posture (Intune compliance) | pop01 wt0 |
| Squid ACL | Network-level allow/deny per subnet | pop01 Squid |
| ICAP pipeline | Content allow/deny per transaction | pop01 + mgmt01 |
| ICAP fail-open | `bypass=on` — uploads pass uninspected if Python DLP container is down | pop01 Squid → mgmt01 |
| NetBird ACL policies | Which peer can reach which resource | NetBird management |
| Unbound RPZ | DNS-level domain allow/deny | pop01 Unbound |
| OPNsense pf | Stateful firewall (baseline) | pop01 |
| SWG identity layer | Squid queries Identity Bridge for persona group — identity-based filtering per user (Studenten/Docenten/Admins) | pop01 Squid → mgmt01 Identity Bridge |
| DC-LAN egress | No SWG inspection — routed by OPNsense, not proxied | pop01 vtnet1 gateway |

---

## 9. Event-Driven Enforcement Layer

All detection components publish structured events to a central NATS JetStream bus on mgmt01. Two consumers process these events independently:

```
Detection silos (producers)
  ├── Suricata IDS (pop01)          → security.alert.ids
  ├── Squid proxy (pop01)           → security.alert.proxy
  ├── Python DLP (mgmt01)           → security.alert.dlp
  ├── c-icap/ClamAV (pop01)         → security.alert.malware
  ├── DNS-RPZ (pop01)               → security.alert.dns
  └── Identity Bridge (mgmt01)      → identity.login / identity.group_change
                    │
                    ▼
            NATS JetStream (mgmt01)
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    Control Daemon         Wazuh SIEM
    (real-time)            (forensic)
    - threat scoring       - custom rules (100500-100600)
    - quarantine/restore   - M365 Active Response
    - ENFORCE gate         - Discover dashboard
```

The dual-write architecture ensures detection events are available for both real-time response (control daemon) and forensic investigation (Wazuh) independently. Neither path depends on the other.

---

## 10. Gate Model

Three gates enforce access control at different points in the connection lifecycle:

| Gate | Technology | Timing | Status |
|------|-----------|--------|--------|
| Gate 1 — Identity | Entra ID Conditional Access (5 policies) | At authentication (OIDC login) | ✅ Operational |
| Gate 2 — Device | Intune device compliance | At authentication + continuous (8h cycle) | ✅ Operational (Report-only until demo) |
| Gate 3 — Content | Squid + ClamAV + Python DLP + Suricata + Unbound RPZ | Every HTTP/DNS request | ✅ Operational |

Gates are complementary, not redundant. Gate 1 evaluates *who* the user is (MFA, sign-in risk, geo-location). Gate 2 evaluates *what* the device is (OS version, AV status, firewall). Gate 3 evaluates *what* is being transferred (malware, sensitive data, blocked domains). No single gate can substitute for the others.

---

**DC-LAN inspection gap:** dc01 uses pop01 as its default gateway (`10.0.0.1`) for internet access. This traffic is routed by OPNsense and inspected by Suricata on vtnet1, but does **not** pass through Squid/ICAP — there is no WPAD/PAC or proxy configuration on dc01. DNS queries from dc01 do go through Unbound RPZ. This is an accepted scope limitation: DC resources are server workloads, not BYOD browsers.

---

## Related

- [Concept: SASE](../concepts/sase.md)
- [Concept: Zero Trust / Three-Gate Model](../concepts/zero-trust.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Concept: RPZ / DNS Threat Intelligence](../concepts/rpz.md)
- [Concept: DLP](../concepts/dlp.md)
- [Runbooks — deployment guides](../runbooks/index.md)
