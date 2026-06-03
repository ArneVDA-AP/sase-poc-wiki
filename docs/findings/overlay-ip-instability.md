---
title: "Finding: NetBird overlay IPs are not stable across re-enrollment"
tags: [finding, netbird, identity-bridge, overlay]
---

# Finding: NetBird overlay IPs are not stable across re-enrollment

**Component:** [Identity Bridge](../components/identity-bridge.md), [NetBird](../components/netbird.md)  
**Severity:** Gotcha

## What happened

NetBird overlay IPs drift across re-enrollment cycles. Addendum H hardcoded IP `.95.98`; after re-enrollment (peer delete + re-register), the actual IP was `.218.100`. After a subsequent rename and reboot it drifted again to `.64.80`. Any configuration or documentation referencing the old IP broke silently.

## Root cause

NetBird assigns overlay IPs dynamically from its CGNAT pool. Re-enrollment (deleting a peer and re-registering it) allocates a new IP from the pool. There is no IP reservation mechanism in NetBird CE — the peer identity is the peer ID, not its overlay IP.

## Resolution / workaround

The Identity Bridge cache invalidation logic must be peer-ID based, not IP-based. The `/api/peers` endpoint returns both the stable peer ID and the current overlay IP, so the bridge can track identity by peer ID and update its IP mappings when they change.

For configuration files and documentation, reference peers by name or peer ID rather than overlay IP wherever possible.

## Lessons

- Never hardcode overlay IPs in configuration or documentation — treat them as ephemeral, similar to DHCP leases
- Peer ID is the only stable identifier for a NetBird peer across re-enrollment cycles
- Any system that caches or references overlay IPs must handle IP changes gracefully
