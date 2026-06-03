---
title: "Finding: Wazuh indexer crashes on QEMU kvm64 CPU model"
tags: [finding, wazuh, gns3, qemu]
---

# Finding: Wazuh indexer crashes on QEMU kvm64 CPU model

**Component:** [Wazuh](../components/wazuh.md)  
**Severity:** Blocker (during deployment)

## What happened

The Wazuh indexer (built on Amazon Linux 2023) crashed immediately on startup with illegal instruction errors. The container exited before completing initialization, making the entire Wazuh stack non-functional.

## Root cause

The Wazuh indexer requires the glibc x86-64-v2 instruction set (Haswell and later). This includes SSE4.2, POPCNT, and related instructions. The standard QEMU `kvm64` CPU model masks these instructions — it emulates a generic x86-64 baseline CPU that lacks the required feature flags.

When glibc or OpenSearch (the indexer engine) attempts to use these instructions, the CPU raises an illegal instruction exception and the process is terminated.

## Resolution / workaround

Set the QEMU CPU model to `host` in the GNS3 node settings for mgmt01:

```
GNS3 → mgmt01 → Configure → QEMU → CPU model: host
```

The `host` model passes through the physical host CPU features to the guest VM, making all modern instruction sets available. After changing this setting and restarting the node, the Wazuh indexer started successfully.

## Lessons

- When running modern Linux containers or applications in GNS3/QEMU, always verify CPU feature requirements — `kvm64` is too conservative for many current container images
- Amazon Linux 2023 and its derivatives assume x86-64-v2 as a baseline, which is not guaranteed by `kvm64`
- The symptom (illegal instruction crash on startup) is distinctive — if a container crashes immediately on a QEMU guest, check the CPU model first
