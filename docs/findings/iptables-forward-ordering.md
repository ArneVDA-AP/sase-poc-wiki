---
title: "Finding: iptables FORWARD rule ordering with libvirt"
tags: [finding, network, workaround]
---

# Finding: iptables FORWARD rule ordering with libvirt

**Component:** [GNS3](../components/gns3.md), [ioc2rpz](../components/ioc2rpz.md)  
**Severity:** Gotcha

## What happened

Port-forward ACCEPT rules were added to the FORWARD chain on the GNS3 host using `iptables -A FORWARD` (append). The port forwards did not work — packets were dropped despite matching rules appearing to be in place.

`iptables -L FORWARD --line-numbers` showed the ACCEPT rules at position 24+, while `LIBVIRT_FWI` chain was at position 22 with a REJECT policy.

## Root cause

libvirt installs its own FORWARD chain rules that include a REJECT for traffic not managed by libvirt. When rules are appended (`-A FORWARD`), they are placed after the libvirt REJECT chain. The first matching rule wins — libvirt's REJECT fires before the ACCEPT rule is reached.

## Resolution / workaround

Always insert at the top of the FORWARD chain:

```bash
iptables -I FORWARD 1 -d 192.168.122.0/24 -j ACCEPT
```

`-I FORWARD 1` inserts at position 1 (before all existing rules, including libvirt's REJECT).

The same principle applies to all ACCEPT rules intended to operate alongside libvirt:

```bash
# GUI port forwards for pop01, mgmt01, site01
iptables -I FORWARD 1 -d 192.168.122.13 -j ACCEPT
iptables -I FORWARD 1 -d 192.168.122.23 -j ACCEPT
iptables -I FORWARD 1 -d 192.168.122.33 -j ACCEPT
```

## Lessons

- When adding FORWARD rules on a host running libvirt, always use `-I FORWARD 1` (insert at top), never `-A FORWARD` (append)
- Diagnosing: `iptables -L FORWARD --line-numbers` to see rule positions and the libvirt chain location
- libvirt's REJECT rule is by design — it prevents unmanaged traffic from crossing between libvirt networks
