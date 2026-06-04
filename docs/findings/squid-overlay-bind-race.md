---
title: "Finding: Squid fails to bind overlay listener when NetBird wt0 is not ready"
tags: [finding, squid, netbird, startup]
---

# Finding: Squid fails to bind overlay listener when NetBird wt0 is not ready

**Component:** [Squid](../components/squid.md), [NetBird](../components/netbird.md)  
**Severity:** Gotcha

## What happened

Squid fails to bind its overlay listener on `100.70.154.79:3128` when it starts before NetBird's `wt0` interface has the overlay IP assigned. `sockstat -4 -l | grep ':3128'` then shows only the LAN listener `10.0.0.1:3128` — the overlay listener is absent. The Squid process stays up (the bind failure is non-fatal), so from the outside Squid looks healthy but is unreachable on the overlay: clients see `curl 000` / `TcpTestSucceeded=False` with an empty access.log.

Three triggers are documented: a pop01 reboot (V37.4), a Squid restart after a GUI config/CA regen (V31.7), and the V43 failover simulation (`ifconfig vtnet0 down/up` on pop01) (V44.6).

## Root cause

Boot order race condition: Squid starts before NetBird finishes tunnel negotiation. The wt0 interface and its overlay IP are not available yet at the moment Squid attempts to bind. FreeBSD rc.d ordering does not enforce a dependency on the NetBird overlay being fully established.

## Resolution / workaround

Operational recovery — once `wt0` is up, rebind by restarting Squid:

```bash
sockstat -4 -l | grep ':3128'   # overlay listener absent?
configctl proxy restart          # NOT 'reload' — that command does not exist on OPNsense (V37.4 / V44.6)
# after: both 10.0.0.1:3128 AND 100.70.154.79:3128 are present
```

A permanent fix — gating Squid startup on `wt0` having an assigned IP, via an rc.d dependency or a wt0-up hook/watchdog — is **proposed hardening but remains an open item** (V31/V37/V44 open point 5). Until it lands, check the overlay listener after every reboot, failover test, or GUI regen and restart Squid if it is missing. The gate would look like:

```bash
# proposed (not yet implemented): wait for wt0 IP before starting Squid
until ifconfig wt0 2>/dev/null | grep -q 'inet 100\.'; do
  sleep 1
done
```

## Lessons

- Any service binding to a NetBird overlay IP must handle the interface not being ready yet — this is a structural issue, not a one-time race
- Squid does not retry failed binds after startup; a failed bind is permanent until the process is restarted (`configctl proxy restart`)
- The failure is silent only at the proxy level — Squid keeps running and access.log stays empty — but the bind error **is** recorded in `cache.log`: `commBind Cannot bind socket FD … to 100.70.154.79:3128: (49) Can't assign requested address`. Diagnose via `sockstat` (overlay listener absent) or that cache.log line, not via access.log
