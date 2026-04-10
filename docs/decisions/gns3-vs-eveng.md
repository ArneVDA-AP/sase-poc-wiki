---
title: "Decision: GNS3 vs EVE-NG"
tags: [decision, network, architecture]
---

# Decision: GNS3 vs EVE-NG

**Status:** Implemented  
**Date:** Early project (before Verslag18)

## Context

The handbook (v4) specified EVE-NG as the virtualization platform. The team needed to run multiple VMs simultaneously with four people working concurrently on different components.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **EVE-NG** | Handbook-specified; web interface (no client install) | Single web interface — two users working the same node block each other; OVA image format incompatible with QCOW2 (OPNsense/VyOS native format) |
| **GNS3** | Client-server model — multiple GUI clients, same server; supports QCOW2/IMG/ISO natively; multi-user concurrent | Requires desktop GUI client install on each team member's laptop |

## Decision

GNS3, running as a systemd service on an Ubuntu 24.04 VM (poc-1a, `10.158.10.67`) inside Proxmox.

The multi-user requirement was the decisive factor. With four team members each responsible for a different component, sequential access (EVE-NG's limitation) was impractical. GNS3's client-server model lets all four members connect to the same server at `10.158.10.67:3080` and work independently on their VMs simultaneously.

The QCOW2 format compatibility was a secondary factor — OPNsense, VyOS, and Ubuntu 24.04 Server all distribute QCOW2 as primary format. EVE-NG's OVA import requirement would have required format conversion.

## Consequences

- GNS3 runs on Ubuntu 24.04 VM (nested QEMU/KVM) — requires Proxmox CPU `type = host` for VMX/SVM pass-through
- libvirt default network (`virbr0`, `192.168.122.0/24`) serves as the WAN segment and provides NAT — handbook's EVE-NG iptables NAT procedures do not apply
- Handbook references `192.168.100.0/24` as the WAN subnet; in this sandbox it is `192.168.122.0/24` (libvirt default)
- GNS3 "Stop node" = QEMU SIGKILL — OPNsense must be shut down via `shutdown -h now` before GNS3 stops the node (FreeBSD UFS soft-updates risks filesystem corruption on unclean shutdown)
- The handbook's EVE-NG-specific procedures (IP forwarding, NAT setup, OVA imports) are all replaced by the GNS3 equivalents documented in Addendum C

See also: [Component: GNS3](../components/gns3.md)
