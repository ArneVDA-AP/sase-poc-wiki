---
title: "Finding: GNS3 host RAM overcommit with zero swap"
tags: [finding, gns3, qemu]
---

# Finding: GNS3 host RAM overcommit with zero swap

**Component:** [GNS3](../components/gns3.md)  
**Severity:** Blocker (structural / latent)

## What happened

During a large image pull on the mgmt01 guest, the GNS3 host (`poc-1a`) reported itself near the memory ceiling: 49.01 GB total, 46.33 GB used, 2.68 GB free, swap 0. The pull pushed mgmt01's host-side resident set to roughly 12.3 GB (a ~2.4 GB download plus unpacking into page cache), leaving almost no headroom on the hypervisor.

## Root cause

The sum of guest RAM allocations on `poc-1a` far exceeds physical memory. The sandbox mgmt01 and the team-project mgmt01 each request `-m 16384M`, pop01 takes `8192M`, and the remaining nodes add more — together well past the 49 GB the host has. QEMU allocates guest RAM lazily, so this overcommit stays invisible as long as the guests do not touch their pages. A memory spike that forces those pages to be backed — such as an image pull pulling content into cache — collapses the slack.

With 0 swap configured, there is no overflow surface. When free memory runs out, the Linux OOM-killer steps in and terminates a QEMU process of its own choosing. That victim may be any guest on the host: pop01 (an OPNsense kill risks UFS corruption — see the OPNsense stop-node gotcha on the [GNS3](../components/gns3.md) page), the sandbox mgmt01, or a teammate's node on the shared host. Earlier "go-at-default, plenty free" readings were taken inside the guest and measured the wrong layer; the binding constraint lives on the hypervisor, and the pre-flight had missed it.

The largest single skew is mgmt01's `-m 16384M` against roughly 1.1 GB actually in use — a 16 GB reservation for a guest that needs a fraction of it.

## Resolution / workaround

No structural fix was applied. Headroom was reclaimed for the session by stopping sandbox nodes not needed at the time (`sitepc01`, `dc01`), which dropped the host back below the ceiling. The risk returns whenever guests run in parallel again — particularly a future Zeek/RITA node alongside Wazuh.

A durable fix has two parts, both requiring coordination because they touch the shared host:

- Add a swapfile on `poc-1a` so an OOM event has an overflow surface instead of killing a guest outright. This affects every project on the host, so agree it with the team first.
- Lower the mgmt01 reservation from 16384M toward roughly 8 GB, closing the biggest gap between reserved and used. This needs a separate guest reboot.

## Lessons

- Measure `free -h` on the GNS3 host (`poc-1a`), not only inside the guest — a guest reporting free memory says nothing about hypervisor headroom when allocations are overcommitted.
- QEMU lazy RAM allocation hides overcommit until a spike forces the pages to be backed; a single large image pull is enough to expose it.
- Zero swap turns a memory spike into an OOM-kill of an arbitrary QEMU process — on a shared host that can take down another project's guest, not just yours.
