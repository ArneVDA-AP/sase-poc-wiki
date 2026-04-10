---
title: "Finding: Unbound 1.24.2 does not support TSIG for zone transfers"
tags: [finding, network, ioc2rpz]
---

# Finding: Unbound 1.24.2 does not support TSIG for zone transfers

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Severity:** Blocker

## What happened

Attempting to configure Unbound to transfer the RPZ zone directly from ioc2rpz using TSIG authentication failed. Unbound's `rpz:` block accepts a `primary:` address but has no option for specifying a TSIG key.

ioc2rpz requires TSIG authentication for zone transfer authorization. Without it, AXFR requests are rejected by ioc2rpz.

## Root cause

Unbound 1.24.2 does not implement TSIG for incoming zone transfers (`rpz: primary:` configuration). This is a documented missing feature in NLnetLabs/unbound GitHub issue #336, open since October 2020.

## Resolution / workaround

Insert BIND 9.20 (OPNsense `os-bind` plugin) as an intermediary. BIND handles the TSIG-authenticated AXFR from ioc2rpz and presents the zone to Unbound via loopback on `127.0.0.1:53530` without authentication.

See [Decision: BIND as TSIG intermediary](../decisions/bind-tsig-intermediary.md).

## Lessons

- Always check current feature support in release notes before designing a transfer chain — missing features in long-running open issues may not be fixed on your timeline
- BIND is a well-supported secondary zone mechanism; using it as a TSIG-capable intermediary is a clean workaround with minimal overhead
- NLnetLabs/unbound issue #336 is the authoritative reference if this limitation affects future projects
