---
title: "Finding: Squid fails to bind overlay listener when NetBird wt0 is not ready"
tags: [finding, squid, netbird, startup]
---

# Finding: Squid fails to bind overlay listener when NetBird wt0 is not ready

**Component:** [Squid](../components/squid.md), [NetBird](../components/netbird.md)  
**Severity:** Gotcha

## What happened

Squid tries to bind the overlay listener on `100.x.x.x` before NetBird wt0 has assigned the overlay IP. The result is errno 49 (EADDRNOTAVAIL). Squid fails silently — no overlay listener is created, but the process stays running. From the outside, Squid appears healthy but is unreachable on the overlay network.

This was triggered by reboots, failover tests (V43 `ifconfig vtnet0 down/up`), and GUI-regen scenarios.

## Root cause

Boot order race condition: Squid starts before NetBird finishes tunnel negotiation. The wt0 interface and its overlay IP are not available yet at the moment Squid attempts to bind. FreeBSD rc.d ordering does not enforce a dependency on the NetBird overlay being fully established.

## Resolution / workaround

Introduce a startup delay or explicit rc.d dependency ordering on pop01. Squid's startup must be gated on the wt0 interface being present and carrying an assigned IP address.

A simple check before Squid starts:

```bash
# Wait for wt0 to have an IP before starting Squid
until ifconfig wt0 2>/dev/null | grep -q 'inet 100\.'; do
  sleep 1
done
```

## Lessons

- Any service binding to a NetBird overlay IP must handle the interface not being ready yet — this is a structural issue, not a one-time race
- Squid does not retry failed binds after startup; a failed bind is permanent until the process is restarted
- The failure is silent: Squid logs no error for a bind failure on a non-existent interface, making this hard to diagnose without explicitly checking listener state
