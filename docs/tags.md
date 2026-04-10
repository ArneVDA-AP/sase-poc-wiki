---
title: "Controlled Tag Vocabulary"
---

# Controlled Tag Vocabulary

All wiki pages use tags drawn exclusively from this list. A new tag is added only when a concept appears in at least two source documents and no existing tag covers it.

---

## Page-type tags

These structural tags identify the page's role in the wiki. Every decision and finding page carries one.

| Tag | Applies to |
|-----|-----------|
| `architecture` | Pages describing system structure or topology |
| `decision` | Decision record pages (`decisions/`) |
| `finding` | Finding / gotcha pages (`findings/`) |
| `workaround` | Findings that document a workaround for a bug or limitation |

---

## Component tags

One tag per distinct tool or system component.

| Tag | Component |
|-----|-----------|
| `bind` | BIND 9.20 — TSIG-authenticated secondary zone server on pop01 |
| `c-icap` | c-icap server — ICAP daemon that fronts ClamAV on pop01 |
| `caddy` | Caddy — WPAD server, TLS terminator, reverse proxy on mgmt01 |
| `clamav` | ClamAV — malware scanner and DLP layer 1 engine on pop01 |
| `docker` | Docker / Docker Compose — container runtime on mgmt01 |
| `ioc2rpz` | ioc2rpz — RPZ zone aggregator on mgmt01 |
| `netbird` | NetBird — WireGuard-based ZTNA overlay (includes Zitadel, Entra ID) |
| `opnsense` | OPNsense 25.1 — firewall OS on pop01 |
| `python` | Python ICAP server — upload DLP layer 2 on mgmt01 |
| `squid` | Squid 6.x — explicit HTTP/HTTPS proxy on pop01 |
| `suricata` | Suricata 7.x — IDS on pop01 WAN + LAN interfaces |
| `unbound` | Unbound — DNS resolver with RPZ enforcement on pop01 |

---

## Protocol and technology tags

| Tag | Concept |
|-----|---------|
| `dns` | Domain Name System — resolution, zones, threat intelligence |
| `firewall` | Packet-filtering firewall (OPNsense pf) |
| `icap` | Internet Content Adaptation Protocol — proxy inspection offload |
| `ids` | Intrusion Detection System mode (PCAP, alert-only) |
| `ips` | Intrusion Prevention System mode (inline, drop-capable) |
| `multipart` | `multipart/form-data` encoding — upload body parsing |
| `network` | General networking topics: routing, interfaces, segments |
| `pac` | Proxy Auto-Configuration — the JavaScript PAC file format |
| `proxy` | HTTP/HTTPS proxy — explicit or transparent |
| `reqmod` | ICAP REQMOD — request-side content adaptation |
| `respmod` | ICAP RESPMOD — response-side content adaptation |
| `rpz` | DNS Response Policy Zones — DNS-level domain blocking |
| `ssl-bump` | Squid SSL Bump — HTTPS interception via on-the-fly CA |
| `tls` | TLS/SSL — encryption, certificates, trust stores |
| `wpad` | Web Proxy Auto-Discovery — browser proxy configuration protocol |

---

## Security concept tags

| Tag | Concept |
|-----|---------|
| `antivirus` | Signature-based malware detection (ClamAV databases) |
| `dlp` | Data Loss Prevention — detecting and blocking sensitive data exfiltration |
| `sase` | Secure Access Service Edge — the overarching architectural framework |
| `yara` | YARA rules — pattern-matching engine for malware and DLP detection |
| `zero-trust` | Zero Trust security model — never trust, always verify; three-gate model |
