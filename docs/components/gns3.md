---
title: "GNS3 — Virtualization and Lab Topology"
tags: [network, architecture, sase]
---

# GNS3 — Virtualization and Lab Topology

**Role:** Lab infrastructure platform — hosts all SASE stack VMs and simulates the network topology (WAN segment, DC-LAN, Site-LAN) on a single physical server using nested QEMU/KVM virtualization.  
**Status:** ✅ Fully operational  
**Config location:** GNS3 project on `poc-1a` (Ubuntu 24.04 VM at `10.158.10.67`), Proxmox VM-103

---

## How it works in this stack

GNS3 runs as a systemd service on an Ubuntu 24.04 VM (`poc-1a`) inside Proxmox. The GNS3 Server manages QEMU/KVM virtual machines and Ethernet switch nodes. Team members connect with the GNS3 GUI application from their own laptops to `10.158.10.67:3080` — multiple people can work on the same topology simultaneously, each on their own VM console.

**Why GNS3 and not EVE-NG:** EVE-NG has a single web interface — two users working on the same node block each other. GNS3's client-server model allows all four team members to work concurrently. EVE-NG also uses OVA imports that are incompatible with the QCOW2 images that OPNsense, VyOS, and Ubuntu distribute as primary download format.

**Nested virtualization:** GNS3 uses QEMU/KVM for VMs inside the topology. The Proxmox host passes CPU virtualization extensions (VMX/SVM) through to the Ubuntu VM with `cpu type = host`. Verify with `egrep -c '(vmx|svm)' /proc/cpuinfo` — must be > 0.

**libvirt as WAN segment:** GNS3's NAT node attaches to libvirt's default network (`virbr0`, `192.168.122.0/24`). libvirt handles NAT and internet routing automatically — no manual iptables rules needed for internet access from VMs. This replaces the EVE-NG IP forwarding and NAT steps from the handbook.

---

## Topology

### Nodes

| Node | Type | OS | RAM | vCPU |
|------|------|----|-----|------|
| pop01 | QEMU (QCOW2) | OPNsense 25.1 | 8192 MB | 2 |
| mgmt01 | QEMU (QCOW2) | Ubuntu 24.04 | 16384 MB | 4 |
| dc01 | QEMU (QCOW2) | Ubuntu 24.04 | 4096 MB | 2 |
| site01 | QEMU (QCOW2) | VyOS | 1024 MB | 1 |
| sitepc01 | QEMU (QCOW2) | Ubuntu 24.04 | 4096 MB | 2 |

pop01 requires 8 GB RAM for ClamAV + Suricata + Squid running concurrently (handbook specifies 4 GB — this is insufficient).

### Cabling

```
NAT-Internet ──── Switch-WAN        (WAN segment, 192.168.122.0/24)
Switch-WAN   ──── pop01  vtnet0     (pop01 WAN interface)
Switch-WAN   ──── mgmt01 ens3       (mgmt01 WAN interface)
Switch-WAN   ──── site01 eth0       (site01 WAN interface)
Switch-LAN   ──── pop01  vtnet1     (pop01 LAN → DC-LAN)
Switch-LAN   ──── dc01   ens3       (dc01 DC-LAN)
Switch-Site  ──── site01 eth1       (site01 Site-LAN)
Switch-Site  ──── sitepc01 ens3     (sitepc01 Site-LAN)
```

mobile01 is a VMware VM on a team member's laptop — not a GNS3 node. It connects directly via its own network adapter, simulating a real BYOD user outside the GNS3 topology. It reaches the SASE stack exclusively through the NetBird WireGuard tunnel.

### IP addressing

**WAN segment (192.168.122.0/24, libvirt NAT gateway .1):**

| Node | WAN IP | External access |
|------|--------|-----------------|
| pop01 | `192.168.122.13` | SSH: 7022, WebUI: 7443 |
| mgmt01 | `192.168.122.23` | SSH: 7023 |
| site01 | `192.168.122.33` | SSH: 7033 |

**Isolated segments:**

| Segment | CIDR | Gateway | Nodes |
|---------|------|---------|-------|
| DC-LAN | `10.0.0.0/24` | `10.0.0.1` (pop01 vtnet1) | dc01: `10.0.0.100` |
| Site-LAN | `172.16.10.0/24` | `172.16.10.1` (site01 eth1) | sitepc01: `172.16.10.50` (no OS) |

**NetBird overlay (100.64.0.0/10):**

| Node | Overlay IP |
|------|-----------|
| pop01 | `100.70.154.79` |
| mgmt01 | `100.70.135.241` |
| mobile01 | `100.70.95.98` |

### External access port forwards

Port forwards via iptables DNAT on the GNS3 host (`10.158.10.67`). Rule ordering is critical — see [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

```bash
iptables -t nat -A PREROUTING -p tcp --dport 7022 -j DNAT --to 192.168.122.13:22
iptables -t nat -A PREROUTING -p tcp --dport 7443 -j DNAT --to 192.168.122.13:443
iptables -t nat -A PREROUTING -p tcp --dport 7023 -j DNAT --to 192.168.122.23:22
iptables -t nat -A PREROUTING -p tcp --dport 7033 -j DNAT --to 192.168.122.33:22
# FORWARD rules: use -I FORWARD 1, not -A FORWARD
iptables -I FORWARD 1 -d 192.168.122.0/24 -j ACCEPT
```

The GNS3 host also runs nginx SNI stream passthrough on port 443, routing `netbird.sandbox.local` → `192.168.122.23:443`.

---

## Snapshots

GNS3 snapshots cover the entire project — all VMs simultaneously. Unlike VMware/Proxmox per-VM snapshots, a GNS3 snapshot captures the full topology state.

**Before creating a snapshot:** Stop all nodes first (Edit → Stop all nodes, wait until all nodes go grey).

**Rollback:** File → Snapshots → [name] → Restore. GNS3 automatically stops all running nodes.

Current snapshots:

| Name | Contents |
|------|----------|
| `Fase2-ZTNA-Complete` | Full ZTNA stack operational (NetBird, Zitadel, Entra ID federation) |

---

## Known issues / gotchas

**"Stop Node" = QEMU SIGKILL → filesystem corruption on OPNsense** — GNS3's stop button sends SIGKILL directly to QEMU. OPNsense runs on FreeBSD UFS with soft updates, which writes asynchronously. An abrupt kill can produce an inconsistent filesystem state or lost config. Always shut down OPNsense properly before stopping the GNS3 node:
```bash
# From OPNsense console or SSH:
shutdown -h now
# Wait for node to stop in GNS3, then proceed
```
Mitigation: add `fsck_y_enable="YES"` to `/etc/rc.conf` on pop01 — FreeBSD then runs fsck automatically on next boot after unclean shutdown.

**iptables FORWARD ordering** — libvirt places its own REJECT rules in the FORWARD chain. Rules appended with `-A FORWARD` land after those REJECT rules and never match. Always use `-I FORWARD 1` for ACCEPT rules. See [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

**ubridge learning bridge limits traffic visibility** — Switch-WAN uses ubridge, a learning L2 bridge. After MAC learning, unicast frames only go to the correct port — not flooded. Mgmt01 in promiscuous mode on ens3 does not see pop01's traffic to the internet. This impacts any future Zeek/RITA deployment: validate with `tcpdump -i ens3 -n host 8.8.8.8` on mgmt01 before deploying.

**sitepc01 has no OS installed** — the node exists in the topology and is cabled to Switch-Site, but Ubuntu 24.04 has not been installed. It is not operationally active.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: VyOS](vyos.md)
- [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md)
- [Decision: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)
