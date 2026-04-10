---
title: "Runbook: Lab Environment"
tags: [runbook, gns3, proxmox, virtualization]
---

# Runbook: Lab Environment

**Source:** `raw/Doc5_Virtualisatie_Topologie.md`
**Node(s):** GNS3 host (poc-1a, `10.158.10.67`) on Proxmox
**Prerequisites:** Proxmox hypervisor with nested virtualization support, QCOW2 images for all VMs
**Status:** Operational

---

## Prerequisites checklist

- [ ] Proxmox server available (`10.133.0.21:8006`)
- [ ] Ubuntu 24.04 Server ISO or image
- [ ] QCOW2 images: OPNsense 25.1, Ubuntu 24.04, VyOS
- [ ] DHCP reservation on school network VLAN 158 for `10.158.10.67`

---

## Step 1: Create Proxmox Ubuntu VM

Create VM-103 (`poc-1a`) on Proxmox with these resources:

```
RAM:    50 GB
vCPU:   12
Disk:   325 GB
IP:     10.158.10.67 (DHCP reservation on VLAN labo158)
User:   admin-1a
```

Set the Proxmox CPU type to `host` — this passes VMX/SVM instructions through to the VM, which is required for nested QEMU inside GNS3.

**Verify:**

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
# Must be > 0

lsmod | grep kvm
# Must show kvm_intel or kvm_amd
```

---

## Step 2: Install GNS3 Server

```bash
sudo add-apt-repository ppa:gns3/ppa -y
sudo apt update
sudo apt install -y gns3-server qemu-kvm libvirt-daemon-system \
    virtinst bridge-utils ubridge
```

During installation: answer **Yes** to both dialogs (packet capture + wireshark).

```bash
sudo usermod -aG kvm,libvirt,ubridge,wireshark admin-1a
```

Activate the libvirt default network — this provides the WAN segment (`192.168.122.0/24`) with NAT:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

**Verify:**

```bash
ip addr show virbr0
# Must show inet 192.168.122.1/24

systemctl status gns3
# Must show active (running)
```

See [Component: GNS3](../components/gns3.md) for the full systemd service configuration.

---

## Step 3: Create topology in GNS3 GUI

Connect the GNS3 GUI from your laptop: `Edit → Preferences → Server → Remote server: 10.158.10.67:3080`.

**Create nodes on the canvas:**

| Node | Type | Name |
|------|------|------|
| NAT node | GNS3 built-in | `NAT-Internet` |
| Layer 2 switch | Ethernet switch | `Switch-WAN` |
| Layer 2 switch | Ethernet switch | `Switch-LAN` |
| Layer 2 switch | Ethernet switch | `Switch-Site` |
| OPNsense 25.1 | QEMU (QCOW2) | `pop01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `mgmt01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `dc01` |
| VyOS | QEMU (QCOW2) | `site01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `sitepc01` |

`mobile01` (Windows 11) runs as a VMware VM on a team member's laptop — it is **not** a GNS3 node. It connects exclusively via the NetBird WireGuard tunnel.

**Cabling:**

```
NAT-Internet ──── Switch-WAN        (WAN segment, 192.168.122.0/24)
Switch-WAN   ──── pop01  vtnet0     (WAN interface)
Switch-WAN   ──── mgmt01 ens3       (WAN interface)
Switch-WAN   ──── site01 eth0       (WAN interface)
Switch-LAN   ──── pop01  vtnet1     (LAN to DC-LAN)
Switch-LAN   ──── dc01   ens3       (DC-LAN)
Switch-Site  ──── site01 eth1       (Site-LAN)
Switch-Site  ──── sitepc01 ens3     (Site-LAN)
```

**Verify:** All nodes appear on the canvas with correct cabling. Right-click → Start each node; verify console access.

---

## Step 4: Configure VM resources

Right-click each node → Configure → General settings:

| VM | RAM | vCPU | Adapters | Note |
|----|-----|------|----------|------|
| pop01 (OPNsense) | 8192 MB | 2 | 3 (vtnet0, vtnet1, vtnet2) | **8 GB required** — see warning below |
| mgmt01 (Ubuntu) | 16384 MB | 4 | 1 | Docker stack |
| dc01 (Ubuntu) | 4096 MB | 2 | 1 | Datacenter simulation |
| site01 (VyOS) | 1024 MB | 1 | 2 (eth0, eth1) | SD-WAN gateway |
| sitepc01 (Ubuntu) | 4096 MB | 2 | 1 | No OS installed yet |

> **Gotcha: pop01 needs 8 GB, not 4 GB.** The handbook specifies 4 GB, but ClamAV (~1.2 GB) + Suricata (~760 MB + 4 GB Hyperscan compilation peak) + Squid (~400 MB) running concurrently exceed 6 GB. At 4 GB, FreeBSD's OOM-killer silently terminates processes without log entries.
> See [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md) for memory analysis.

---

## Step 5: Configure IP addressing and port forwards

**WAN segment (192.168.122.0/24, libvirt NAT gateway .1):**

| Node | WAN IP | External access | Services |
|------|--------|-----------------|----------|
| pop01 | `192.168.122.13` | SSH: 7022, WebUI: 7443 | OPNsense + SASE stack |
| mgmt01 | `192.168.122.23` | SSH: 7023 | Docker: NetBird, ioc2rpz, DLP |
| site01 | `192.168.122.33` | SSH: 7033 | VyOS |

**Isolated segments:**

| Segment | CIDR | Gateway | Nodes |
|---------|------|---------|-------|
| DC-LAN | `10.0.0.0/24` | `10.0.0.1` (pop01 vtnet1) | dc01: `10.0.0.100` |
| Site-LAN | `172.16.10.0/24` | `172.16.10.1` (site01 eth1) | sitepc01: `172.16.10.50` |

**Port forwards on GNS3 host** (`10.158.10.67`):

```bash
# Pop01 SSH
iptables -t nat -A PREROUTING -p tcp --dport 7022 -j DNAT --to 192.168.122.13:22
# Pop01 WebUI (HTTPS)
iptables -t nat -A PREROUTING -p tcp --dport 7443 -j DNAT --to 192.168.122.13:443
# Mgmt01 SSH
iptables -t nat -A PREROUTING -p tcp --dport 7023 -j DNAT --to 192.168.122.23:22
# VyOS SSH
iptables -t nat -A PREROUTING -p tcp --dport 7033 -j DNAT --to 192.168.122.33:22
```

> **Gotcha: Always use `-I FORWARD 1`, never `-A FORWARD`.** libvirt places its own FORWARD chain rules with a REJECT for non-libvirt-managed traffic. An appended (`-A`) ACCEPT rule lands after the libvirt REJECT and never matches. Insert at position 1 instead.
> See [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

```bash
sudo iptables -I FORWARD 1 -d 192.168.122.13 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.23 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.33 -j ACCEPT
```

Persist with `iptables-save` and `netfilter-persistent`.

**SNI-based routing for NetBird Dashboard:** The GNS3 host runs an nginx stream module on port 443 that routes based on SNI hostname:

```
netbird.sandbox.local  →  192.168.122.23:443
```

**Verify:**

```bash
ssh admin-1a@10.158.10.67 -p 7022   # reaches pop01
ssh mgmt@10.158.10.67 -p 7023       # reaches mgmt01
```

---

## Step 6: Snapshots and safe shutdown

> **Gotcha: GNS3 "Stop Node" = hard QEMU kill = filesystem corruption on OPNsense.** GNS3's "Stop node" sends SIGKILL to QEMU. OPNsense on FreeBSD UFS with soft updates can lose configuration or fail to boot after an unclean kill.
> See [Finding: NetBird config zero bytes](../findings/netbird-config-zero-bytes.md) for related UFS corruption issues.

**Correct shutdown procedure:**

```bash
# From OPNsense console or SSH:
shutdown -h now
# Wait until the node turns grey in GNS3 (15-30 sec)
# THEN: GNS3 → Edit → Stop all nodes (or right-click → Stop)
```

Safety net: add `fsck_y_enable="YES"` to `/etc/rc.conf` on pop01 for automatic filesystem check on boot.

**Create snapshots:**

```
GNS3 menu → Edit → Stop all nodes
Wait until all nodes turn grey (15-30 sec)
→ File → Snapshots → Create snapshot
```

All nodes **must** be stopped before creating a snapshot.

**Current snapshots:**

| Name | Contents |
|------|----------|
| `Fase2-ZTNA-Complete` | Full ZTNA stack operational |
| `Fase3-Security-Complete` | Planned — after full security stack |

---

## Final verification

- [ ] `egrep -c '(vmx|svm)' /proc/cpuinfo` > 0 on poc-1a
- [ ] `systemctl status gns3` shows active
- [ ] `ip addr show virbr0` shows `192.168.122.1/24`
- [ ] GNS3 GUI connects to `10.158.10.67:3080` and shows all nodes
- [ ] All nodes start and console access works
- [ ] Port forwards work: SSH via 7022/7023/7033
- [ ] Snapshot created successfully

---

## Related

- [Component: GNS3](../components/gns3.md)
- [Component: VyOS](../components/vyos.md)
- [Decision: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)
- [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md)
- [Architecture](../overview/architecture.md)
