---
title: "Concept: RPZ — DNS Response Policy Zones"
tags: [ioc2rpz, dns, rpz, network, sase]
---

# Concept: RPZ — DNS Response Policy Zones

**One-line definition:** An extension to DNS authoritative zone serving that lets a resolver return synthetic responses (NXDOMAIN, NODATA, a walled garden IP) for matching query names, enabling DNS-level blocking of known-malicious domains before any network connection is established.

## How it applies here

RPZ intercepts at the earliest possible point in the threat lifecycle: name resolution. If a BYOD client tries to resolve the hostname of a known C2 server, Unbound returns NXDOMAIN — the TCP connection never starts, regardless of port or protocol.

The RPZ zone in this stack contains ~71 767 records from two Abuse.ch feeds (URLhaus + ThreatFox), updated by ioc2rpz from live threat intelligence feeds. When a client queries a hostname that matches an RPZ record, Unbound substitutes the configured action (NXDOMAIN) instead of returning the real answer.

**Zone transfer chain:**

```
URLhaus + ThreatFox (Abuse.ch feeds)
    ↓ HTTP pull by ioc2rpz
ioc2rpz → BIND 9.20 (TSIG AXFR) → Unbound (RPZ, respip module)
                                         ↓
                              All client DNS queries
```

Unbound loads the RPZ zone into memory. Per-query performance impact is negligible — the zone is already resident.

## Where it appears in the stack

**[ioc2rpz](../components/ioc2rpz.md)** — the zone source. Aggregates URLhaus and ThreatFox feeds into a single RPZ zone (`threat-intel.rpz.sase`). Acts as an authoritative DNS server that BIND transfers from.

**[NetBird](../components/netbird.md)** — the primary nameserver configuration routes all BYOD client DNS queries through pop01 Unbound. Without this, external queries bypass Unbound entirely, defeating RPZ protection for non-internal domains.

**[Suricata](../components/suricata.md)** — complementary control. Suricata detects connections to known-malicious IPs (from Abuse.ch SSL IP Blacklist and similar); RPZ prevents DNS resolution of known-malicious domains. Both use Abuse.ch as a source; they cover different threat surfaces (connection vs. name resolution).

## Key distinctions

**RPZ action = NXDOMAIN + authoritative flag** — Unbound's RPZ response returns NXDOMAIN with the `aa` (authoritative answer) flag set. This distinguishes a local RPZ-triggered NXDOMAIN from a real NXDOMAIN from the internet. The `aa` flag in a test response (`drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch`) confirms the RPZ is active, not that the domain is genuinely non-existent.

**RPZ does not block IP-based threats** — a client that bypasses DNS (uses a hardcoded IP) is not protected by RPZ. Suricata on vtnet0 covers this case via IP-based signature rules.

**`respip` module position in Unbound** — RPZ enforcement requires the `respip` module in Unbound's module-config. It must be listed first: `"respip python validator iterator"`. The `python` module must stay (OPNsense uses it for the built-in DNSBL feature). Removing either breaks either RPZ or OPNsense's own blocklist.

## Sources

- `raw/Doc4_DNS_Threat_Intelligence.md` (full RPZ architecture, ioc2rpz + BIND + Unbound)
- `raw/Verslag24.md`, `raw/Verslag25.md`, `raw/Verslag26.md` (implementation sessions)
