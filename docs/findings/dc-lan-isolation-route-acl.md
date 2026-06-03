---
title: "Finding: DC-LAN isolation via empty NetBird route-ACL group"
tags: [finding, netbird, vyos, network-route, isolation]
---

# Finding: DC-LAN isolation via empty NetBird route-ACL group

**Component:** [NetBird](../components/netbird.md), [VyOS](../components/vyos.md)  
**Severity:** Insight

## What happened

DC-LAN isolation was achieved using an empty route-ACL group on a NetBird Network Route. Peers without an explicit group match receive no access to the route, even though the route exists in the NetBird configuration. The result: port 22 (SSH to DC-LAN hosts) is closed to unprivileged peers, while internet access (0.0.0.0/0 exit node) remains open.

## Root cause

NetBird Network Routes distribute routing information only to peers that belong to the specified access control groups. An empty or non-matching group list means the route is invisible to those peers — their NetBird client never receives the route and therefore never attempts to use it. This is by-design behavior, not a bug or misconfiguration.

## Resolution / workaround

No fix needed — this is intentional and provides a clean isolation pattern. To grant a peer access to the DC-LAN route, add it to the appropriate access control group. To revoke access, remove it from the group.

This approach provides selective network segment isolation without requiring separate firewall rules on VyOS or the DC-LAN hosts themselves.

## Lessons

- NetBird Network Routes double as access control for specific network segments — leverage empty route-ACL groups for isolation patterns
- This is more elegant than firewall-based isolation because it operates at the routing layer: peers without access never even see the route, reducing the attack surface
- The pattern works for any network segment behind a NetBird routing peer, not just DC-LAN
