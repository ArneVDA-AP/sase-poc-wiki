---
title: "Finding: NetBird primary nameserver required for external RPZ coverage"
tags: [finding, network, ioc2rpz, workaround]
---

# Finding: NetBird primary nameserver required for external RPZ coverage

**Component:** [ioc2rpz](../components/ioc2rpz.md), [NetBird](../components/netbird.md)  
**Severity:** Blocker (for DNS threat intelligence)

## What happened

Unbound RPZ was confirmed working for `*.sandbox.local` internal domains — `testentry.rpz.urlhaus.abuse.ch` returned NXDOMAIN correctly when queried directly from pop01. However, from mobile01 via NetBird, external domain queries to known-malicious domains were not being blocked. `nslookup testentry.rpz.urlhaus.abuse.ch` on mobile01 returned the real (non-blocked) IP.

## Root cause

NetBird's Custom DNS Zone (`sandbox.local → pop01`) only routes `*.sandbox.local` queries through Unbound. For all other domains, mobile01 used its local network adapter's default DNS (typically the ISP resolver or router). Those queries never reached pop01 Unbound, so the RPZ never evaluated them.

## Resolution / workaround

In NetBird Dashboard → DNS: configure a Primary Nameserver:

```
Primary nameserver: pop01 (100.70.154.79)
Match domains: (empty)
```

The empty match-domains field is critical — it causes NetBird to route *all* DNS queries from the client through pop01 Unbound, not just queries for specific domains. With this setting, external queries are also subject to Unbound RPZ evaluation.

After applying: confirmed via `nslookup testentry.rpz.urlhaus.abuse.ch` on mobile01 returning `Non-existent domain` and the Unbound log showing `rpz: applied [ioc2rpz-threat-intel]`.

## Lessons

- NetBird Custom DNS Zones only affect queries matching the specified domain suffix
- For RPZ to protect against external threats, all DNS traffic (not just internal) must route through the enforcing resolver
- The Primary Nameserver setting with empty match-domains is the NetBird mechanism for this — it replaces the client's adapter DNS for all queries while the tunnel is active
- This is architecturally equivalent to forcing all DNS through a corporate resolver in a traditional VPN setup
