---
title: "Finding: Suricata Netmap IPS fails on virtio NICs"
tags: [finding, suricata, workaround]
---

# Finding: Suricata Netmap IPS fails on virtio NICs

**Component:** [Suricata](../components/suricata.md)  
**Severity:** Blocker (for IPS mode)

## What happened

IPS mode was enabled in OPNsense (Intrusion Detection → Settings → IPS mode ✔, pattern matcher: Netmap). Suricata restarted but processed 0 packets. All previously active alert categories stopped firing. The system appeared to be working (no crash) but was completely blind to all traffic.

Reverting to IDS mode (PCAP) restored normal alert activity.

## Root cause

Netmap IPS mode requires NIC drivers with native Netmap support: Intel igb, ixgbe; Broadcom bge. QEMU's virtio NIC driver (`vtnet`) has no Netmap support.

When Netmap is enabled on an unsupported NIC, Suricata starts but the Netmap ring buffers are never populated. Suricata sits waiting for packets that never arrive. No error is logged — from Suricata's perspective, the interface exists and Netmap initialized, but the ring is simply empty.

`sysctl hw.model` on QEMU VMs shows a generic processor string and does not help identify NIC support. The reliable check is `dmesg | grep -i features` for SSE4.2 (needed for Hyperscan, not for Netmap), and verifying the NIC driver via `dmesg | grep vtnet`.

## Resolution / workaround

Remain in IDS mode (PCAP). The differentiated drop/alert policy table is configured and ready. On physical hardware with Intel igb/ixgbe or Broadcom bge NICs, IPS mode can be activated by enabling the toggle — the policies are already correct.

## Lessons

- Netmap support is NIC-driver-specific, not a kernel capability
- virtio NICs (the QEMU default) do not support Netmap — this is a hard limitation
- When IPS mode produces 0 packets with no error, the first suspect is Netmap/NIC incompatibility
- Always test IPS mode activation in a lab before production: reverting to IDS mode restores previous state immediately
