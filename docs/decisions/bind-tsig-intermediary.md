---
title: "Decision: BIND as TSIG Intermediary"
tags: [decision, dns, network, ioc2rpz]
---

# Decision: BIND as TSIG Intermediary

**Status:** Implemented  
**Date:** April 2026 (Verslag25)

## Context

ioc2rpz authenticates zone transfers with TSIG (Transaction Signature, hmac-sha256). Unbound 1.24.2 supports loading RPZ zones via zone transfer (`rpz: primary:` directive), but does not support TSIG authentication for those transfers. This is a documented missing feature in NLnetLabs/unbound GitHub issue #336, open since October 2020.

Without TSIG support in Unbound, the zone transfer from ioc2rpz cannot be authenticated.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Unbound direct (no TSIG)** | Simplest chain | ioc2rpz requires TSIG for zone transfer authorization; without it, AXFR is rejected |
| **Configure ioc2rpz without TSIG** | Removes the requirement | Removes transfer authentication — any DNS client could pull the RPZ zone |
| **BIND as intermediary** | BIND fully supports TSIG; presents zone to Unbound over loopback without auth | Extra component; adds complexity; requires os-bind plugin on OPNsense |
| **Custom forwarder/script** | Could bridge the gap | Significant custom code, fragile |

## Decision

Insert BIND 9.20 (via OPNsense `os-bind` plugin) as an intermediary between ioc2rpz and Unbound.

BIND handles the TSIG-authenticated AXFR from ioc2rpz and acts as a secondary nameserver for the RPZ zone. Unbound then transfers the zone from BIND on loopback (`127.0.0.1:53530`) without TSIG authentication. The transfer is unauthenticated but confined to localhost — acceptable on a single-node setup.

This is a protocol-gap bridge: BIND exists solely because Unbound 1.24.2 lacks TSIG support. If NLnetLabs/unbound issue #336 is resolved in a future release, BIND can be removed and Unbound can pull directly from ioc2rpz.

## Consequences

- BIND (os-bind plugin) must be installed and configured on pop01 alongside Unbound
- BIND listens only on `127.0.0.1:53530` with `loopback_only` ACLs — it is not exposed as a resolver
- ioc2rpz sends NOTIFY to `192.168.122.13:53` (default DNS port, hitting Unbound, not BIND on 53530). BIND discovers zone updates only via SOA refresh (3600 s). Manual retransfer: `rndc -p 953 retransfer threat-intel.rpz.sase`
- TSIG key (`tkey_rpz_transfer`, hmac-sha256) must be explicitly linked to the RPZ zone in the ioc2rpz GUI — leaving it empty causes TSIG errors on every AXFR attempt

## Related

- [Component: ioc2rpz](../components/ioc2rpz.md)
- [Concept: RPZ](../concepts/rpz.md)
- [Finding: Unbound no TSIG](../findings/unbound-no-tsig.md)
