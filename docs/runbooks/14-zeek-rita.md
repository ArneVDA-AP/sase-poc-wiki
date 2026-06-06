---
title: "Runbook: Zeek & RITA — Network-Wide Behavioral Analysis"
tags: [runbook, zeek, rita, gre, network, beaconing]
---

# Runbook: Zeek & RITA — Network-Wide Behavioral Analysis

**Source:** `rayan_zeek-rita-documentation.md` (Apr 2026), `Verslag07` (May 2026)
**Node(s):** mgmt01 (`192.168.122.20`) Zeek cluster + RITA; VyOS-1 + OPNsense-1 GRE mirror sources
**Prerequisites:** GNS3 lab topology operational; mgmt01 with a second NIC for monitoring
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending

> **Scope.** This runbook builds the Zeek/RITA monitoring on the **parallel stack**. It is a
> PoC-within-the-PoC: validated end-to-end, but not yet wired into the main sandbox topology.
> mgmt01 here is `192.168.122.20` (the parallel stack), not the sandbox's mgmt01.

---

## What this builds

Zeek as a multi-worker cluster on mgmt01, watching three segments at once, feeding hourly logs
into RITA's ClickHouse backend for beaconing analysis. Capture uses two mechanisms: a GNS3 **Hub**
floods the core segment to a dedicated monitor NIC, and **GRE tunnels** carry kernel-mirrored
frames from the remote segments.

```
Core (Hub1) ──flood──► ens4 ──► worker-core ┐
Site1 ──VyOS mirror──► GRE ──► gre-site1 ──► worker-site1 ├─► manager ─► /opt/zeek/logs/ ─► RITA
DC ────(partial)─────► GRE ──► gre-dc ─────► worker-dc   ┘
```

---

## Prerequisites checklist

- [ ] GNS3 topology running, core segment reachable
- [ ] mgmt01 has two NICs: `ens3` (management, has IP) and `ens4` (monitor, no IP)
- [ ] Docker daemon running on mgmt01 (Zeek and RITA run as containers)
- [ ] Root/sudo on mgmt01, VyOS-1, and OPNsense-1

---

## Step 1: Replace the core Switch with a Hub

GNS3's Ethernet Switch (`ubridge`) forwards learned unicast only to the destination port and has
no SPAN feature, so a monitor NIC on it sees nothing between other nodes. Swap it for a GNS3 **Hub**
(`Hub1`), which floods every frame to every port. The monitor NIC then receives every conversation.

See [Decision: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.md) for why this is the
functional equivalent of a SPAN port.

**Hub1 port assignment used:**

```
Ethernet0  NAT1            nat0
Ethernet1  Windows11-1     eth0
Ethernet2  VyOS-1          eth0
Ethernet3  OPNsense-1      eth0
Ethernet4  mgmt01          ens3   (management, has IP)
Ethernet5  mgmt01          ens4   (monitor, no IP)
```

> **Why two NICs:** `ens3` participates in the network (IP, routing, SSH, Docker). `ens4` only
> listens — no IP, no routing, no identity. The kernel hands every received frame to Zeek via
> `libpcap`/`AF_PACKET` without TCP/IP processing. That stops the kernel from answering with TCP
> RSTs / ICMP unreachables / ARP on traffic it was never meant to handle, and keeps mgmt01's own
> management chatter (SSH keepalives, Docker pulls, NTP, apt) out of the RITA baseline.

---

## Step 2: Configure the monitor interface (ens4)

Bring `ens4` up in promiscuous mode and turn off all hardware offloading, otherwise the kernel
gives Zeek reassembled jumbo frames instead of the actual wire frames.

```bash
sudo ip link set ens4 up
sudo ip link set ens4 promisc on
sudo ethtool -K ens4 rx off tx off sg off tso off gso off gro off lro off
```

Persist it across reboots with a systemd oneshot unit:

```bash
sudo tee /etc/systemd/system/ens4-monitor.service > /dev/null <<'EOF'
[Unit]
Description=Configure ens4 as promiscuous monitor interface for Zeek
After=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/ip link set ens4 up
ExecStart=/sbin/ip link set ens4 promisc on
ExecStart=/bin/sh -c '/usr/sbin/ethtool -K ens4 rx off tx off sg off tso off gso off gro off lro off 2>/dev/null || true'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ens4-monitor.service
```

**Verify** — the interface must show `PROMISC`:

```bash
ip link show ens4 | grep PROMISC
# 3: ens4: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 9216 ...
```

---

## Step 3: Mirror the Site1 segment from VyOS (GRE)

VyOS does native kernel mirroring: copy every frame on `eth1` (the Site1 segment) into a GRE
tunnel aimed at mgmt01.

> **Gotcha: this takes two commits.** VyOS validates the mirror target at commit time, and the
> tunnel does not exist yet inside the same commit. Commit the tunnel first, then the mirror.
> See [Finding: VyOS GRE two-step commit](../findings/vyos-gre-two-step-commit.md).

```bash
configure

# Commit 1 — create the tunnel
set interfaces tunnel tun0 encapsulation 'gre'
set interfaces tunnel tun0 source-address '192.168.122.31'
set interfaces tunnel tun0 remote '192.168.122.20'
commit

# Commit 2 — add the mirror now that tun0 exists
set interfaces ethernet eth1 mirror ingress 'tun0'
set interfaces ethernet eth1 mirror egress 'tun0'
commit
save
exit
```

> **Parameter names vary by VyOS version.** This stack needed `source-address` and `remote`, not
> `local-ip` / `remote-ip`. If a parameter is rejected, run `set interfaces tunnel tun0 ?`.

---

## Step 4: Set up the DC GRE tunnel from OPNsense (partial)

```bash
ifconfig gre0 create
ifconfig gre0 tunnel 192.168.122.11 192.168.122.20
ifconfig gre0 up
```

> **Expect no mirrored traffic here.** OPNsense is FreeBSD, which has no kernel-level interface
> mirror. The tunnel comes up and Zeek discovers it, but the inner DC segment (`10.0.0.x`) is not
> copied into it. DC traffic is still seen at the core when it transits OPNsense's WAN. This is an
> accepted PoC limitation — see
> [Finding: DC inner-segment mirror limit](../findings/dc-segment-mirror-limit.md).

OPNsense tunnel config is not persistent — re-run the three `ifconfig` lines after reboot or add
an `/etc/rc.local` entry.

---

## Step 5: Receive the GRE tunnels on mgmt01

Load the GRE kernel module and create the two receiver interfaces:

```bash
sudo modprobe ip_gre
echo "ip_gre" | sudo tee /etc/modules-load.d/gre.conf

# Site1 receiver
sudo ip tunnel add gre-site1 mode gre local 192.168.122.20 remote 192.168.122.31 ttl 255
sudo ip link set gre-site1 up
sudo ip link set gre-site1 promisc on

# DC receiver
sudo ip tunnel add gre-dc mode gre local 192.168.122.20 remote 192.168.122.11 ttl 255
sudo ip link set gre-dc up
sudo ip link set gre-dc promisc on
```

Persist with a systemd oneshot unit (ordered after `ens4-monitor.service`):

```bash
sudo tee /etc/systemd/system/gre-tunnels.service > /dev/null <<'EOF'
[Unit]
Description=GRE tunnels for Zeek remote segment monitoring
After=network-online.target ens4-monitor.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/ip tunnel add gre-site1 mode gre local 192.168.122.20 remote 192.168.122.31 ttl 255
ExecStart=/sbin/ip link set gre-site1 up
ExecStart=/sbin/ip link set gre-site1 promisc on
ExecStart=/sbin/ip tunnel add gre-dc mode gre local 192.168.122.20 remote 192.168.122.11 ttl 255
ExecStart=/sbin/ip link set gre-dc up
ExecStart=/sbin/ip link set gre-dc promisc on
ExecStop=/sbin/ip tunnel del gre-site1
ExecStop=/sbin/ip tunnel del gre-dc

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gre-tunnels.service
```

---

## Step 6: Configure the Zeek cluster

Zeek/RITA were installed via the official `activecm/rita` v5.1.1 Ansible installer; the wrappers
live at `/usr/local/bin/zeek` and `/usr/local/bin/rita`. Two config files matter.

**`/opt/zeek/etc/networks.cfg`** — tell Zeek which subnets are internal:

```
192.168.122.0/24    Core Management LAN (Hub1)
10.0.0.0/24         DC LAN (OPNsense-1 → Switch2 → Ubuntu-Server-DC-1)
172.16.10.0/24      Site1 LAN (VyOS-1 → Switch3 → Ubuntu-Server-Site-1)
```

**`/opt/zeek/etc/node.cfg`** — one worker per capture interface:

```ini
[manager]
type=manager
host=localhost

[proxy-1]
type=proxy
host=localhost

[worker-core]
type=worker
host=localhost
interface=ens4

[worker-site1]
type=worker
host=localhost
interface=gre-site1

[worker-dc]
type=worker
host=localhost
interface=gre-dc
```

> **Gotcha: cluster mode changes the log path.** In standalone mode Zeek writes to
> `/opt/zeek/spool/zeek/`. In cluster mode the manager aggregates into `/opt/zeek/spool/manager/`.
> Re-point the `current` symlink:
> ```bash
> sudo rm -f /opt/zeek/logs/current
> sudo ln -s /opt/zeek/spool/manager /opt/zeek/logs/current
> ```

---

## Step 7: Start Zeek and the RITA backend

```bash
# Zeek (wrapper runs the container with --network=host, --cap-add=NET_ADMIN,NET_RAW)
sudo zeek start
sudo zeek enable      # auto-start on reboot

# RITA backend: ClickHouse + syslog-ng
cd /opt/rita
sudo docker compose up -d

# Wait for ClickHouse to report healthy
docker inspect rita-clickhouse --format='{{.State.Health.Status}}'
# Expected: healthy
```

**Verify the cluster** — all five processes running:

```bash
sudo zeek status
# manager / proxy-1 / worker-core / worker-site1 / worker-dc  → running
```

**Verify GRE decapsulation** — Zeek's `tunnel.log` should discover both tunnels:

```
192.168.122.20 → 192.168.122.11    Tunnel::GRE    Tunnel::DISCOVER
192.168.122.20 → 192.168.122.31    Tunnel::GRE    Tunnel::DISCOVER
```

> A `DISCOVER` line confirms the tunnel, not that traffic flows inside it. `gre-dc` will discover
> but stay empty (Step 4).

---

## Step 8: Import logs into RITA

```bash
# Quick test — a single day into a throwaway database
sudo rita import -l /opt/zeek/logs/2026-04-21/ --database sase_test

# Full rolling baseline — multi-day window
sudo rita import -l /opt/zeek/logs/ --database sase_poc --rolling
```

> **Gotcha: no hyphens in RITA v5 database names.** Use `sase_poc`, not `sase-poc`.

A rolling import of ~12 days processed in about 7.5 minutes. Truncated logs from hours where Zeek
was restarted mid-hour produce non-fatal warnings and are processed anyway.

---

## Step 9: Run beacon analysis

```bash
sudo rita view sase_poc --beacon            # beaconing (primary use case)
sudo rita view sase_poc --long-connection   # long-duration connections (possible C2)
sudo rita view sase_poc --c2-over-dns       # DNS-based C2 indicators
sudo rita view sase_poc --threat-intel      # threat-intel feed matches
```

**What "working" looks like.** With no malware in the lab, the top beacons are legitimate periodic
services — that is exactly the proof the detection pipeline works:

| Source | Destination | Beacon score | Reading |
|--------|-------------|--------------|---------|
| 192.168.122.20 | pkgs.netbird.io | 100.00% | NetBird update check, perfectly regular |
| 192.168.122.20 | 185.125.190.56 | 99.00% | Ubuntu NTP (Canonical) |
| 192.168.122.20 | 8.8.8.8 | 73.40% | Google DNS, periodic |
| 192.168.122.11 | 212.132.97.26 | 68.10% | NTP from OPNsense |

A real C2 beacon would appear in this same list; the analyst investigates any pattern they do not
recognise. RITA correctly surfacing these regular flows confirms the
[behavioral-analysis](../concepts/behavioral-analysis.md) pipeline is operational.

This beacon view is also the input for the automated DNS-blocking pipeline — see
[Runbook 15: RITA → RPZ integration](15-rita-rpz-integration.md).

---

## Coverage summary

| Segment | Coverage | Method | What is visible |
|---------|----------|--------|-----------------|
| Core (Hub1) | Full | Hub flood → ens4 → worker-core | All inter-node traffic |
| Site1 | Full | VyOS eth1 kernel mirror → GRE → worker-site1 | All Site1 internal + Site1 ↔ internet (pre-routing) |
| DC | Partial | Routed traffic at core; GRE up but no active mirror | DC ↔ internet at core; `10.0.0.x` inner perspective not captured (FreeBSD limit) |

Estimated overall topology coverage: **~90%**. The missing ~10% is DC inner-segment traffic, which
has no practical impact in the single-server DC.

---

## Reboot behaviour

These auto-start: `ens4-monitor.service`, `gre-tunnels.service`, the `zeek` container
(`sudo zeek enable`), and `rita-clickhouse` / `rita-syslog-ng` (`restart: unless-stopped`). VyOS
keeps its config via `save`; OPNsense needs the `ifconfig gre0 ...` lines re-run (or an
`/etc/rc.local` entry).

---

## Checklist

- [ ] Core switch replaced with Hub1; ens4 on a dedicated Hub port
- [ ] `ens4` PROMISC, offloading off, persisted via systemd
- [ ] VyOS GRE tunnel + mirror committed in two steps
- [ ] OPNsense GRE tunnel created (DC partial, expected)
- [ ] `gre-site1` and `gre-dc` up and promiscuous on mgmt01
- [ ] `node.cfg` has worker-core / worker-site1 / worker-dc; `current` symlink → manager
- [ ] `sudo zeek status` shows all 5 processes running
- [ ] `tunnel.log` discovers both GRE tunnels
- [ ] RITA import into `sase_poc` (underscore name) succeeds
- [ ] `rita view sase_poc --beacon` lists the legitimate periodic services

---

## Related

- [Component: Zeek](../components/zeek.md)
- [Component: RITA](../components/rita.md)
- [Component: GNS3](../components/gns3.md)
- [Component: VyOS](../components/vyos.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Decision: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.md)
- [Finding: VyOS GRE two-step commit](../findings/vyos-gre-two-step-commit.md)
- [Finding: DC inner-segment mirror limit](../findings/dc-segment-mirror-limit.md)
- [Runbook 15: RITA → RPZ integration](15-rita-rpz-integration.md)
