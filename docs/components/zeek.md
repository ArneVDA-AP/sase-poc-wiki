---
title: "Zeek — Network Traffic Analysis (Behavioral Telemetry)"
tags: [zeek, rita, network, behavioral-analysis, beaconing, c2, gns3, docker, sase]
---

# Zeek — Network Traffic Analysis (Behavioral Telemetry)

**Role:** Passive network sensor that parses every flow on the SASE_POC segments into structured protocol logs (conn, dns, ssl, http, ntp), producing the raw telemetry that [RITA](rita.md) analyses for behavioral threats.  
**Version:** Zeek 6.2.1 (`activecm/zeek` Docker image, `--network=host`)  
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending  
**Config location:** `/opt/zeek/etc/node.cfg` (cluster), `/opt/zeek/etc/networks.cfg` (internal subnets) on mgmt01 (`192.168.122.20`)

> **Sidenote, parallel-stack scope.** Zeek runs on the **SASE_POC** topology built by Rayan, on its own mgmt01 (`192.168.122.20`). It was validated there but is **not yet wired into the main sandbox** — no NATS event publishing, no [Control Daemon](control-daemon.md) hook. It sits alongside the sandbox's signature-based [Suricata IDS](suricata.md) as an additive, behavioral layer rather than a replacement. Where the two overlap, the sandbox is the source of truth.

---

## How it works in this stack

Zeek does not sit inline. It reads frames off interfaces in promiscuous mode, reconstructs TCP streams, identifies the application protocol, and writes one structured log line per event. It produces no verdict and blocks nothing. Its job is to turn raw packets into the `conn.log`, `dns.log`, `ssl.log`, `http.log` and `ntp.log` records that RITA later mines for beaconing and C2 patterns. Detection is split: Zeek sees, RITA decides.

The host is **mgmt01**, the management/operations node of the SASE_POC topology, chosen deliberately:

- **Vantage point.** mgmt01 sits on the core segment and terminates the GRE tunnels from the remote gateways, so it has the widest view in the topology.
- **Separation of planes.** Running the sensor on the management plane keeps it off the workload hosts. If dc01 (the simulated datacenter server) were compromised, the monitoring would not fall with it.
- **Co-location.** Wazuh and the NetBird management already live on mgmt01, so Zeek/RITA telemetry is local for correlation without a network hop.

### Capturing three segments

The topology has three segments, and no single interface sees all of them. Zeek runs as a **multi-worker cluster** so one process per segment can capture in parallel:

| Segment | Capture method | Worker | Interface |
|---------|----------------|--------|-----------|
| Core (Hub1) | Hub floods every frame to a dedicated monitor NIC | `worker-core` | `ens4` |
| Site1 | VyOS kernel-level mirror copies eth1 into a GRE tunnel to mgmt01 | `worker-site1` | `gre-site1` |
| DC | GRE tunnel established; no active inner mirror (FreeBSD limit) | `worker-dc` | `gre-dc` |

The core segment is visible because Switch1 was swapped for a **GNS3 Hub** (`Hub1`). GNS3's Ethernet Switch uses `ubridge`, a learning L2 bridge with no SPAN or port-mirroring capability, so a monitor port on it sees nothing after MAC learning. A Hub floods all frames to all ports by design, which is functionally equivalent to a production SPAN port at the lab's traffic volume. See [Decision: Hub vs switch visibility](../decisions/hub-vs-switch-visibility.md).

The remote segments reach mgmt01 over **GRE tunnels**. VyOS provides native kernel-level interface mirroring (`mirror ingress`/`mirror egress`), so every frame on its Site1 interface is copied into a GRE tunnel and decapsulated by Zeek at the other end. The DC segment cannot do this from OPNsense (FreeBSD lacks an equivalent mirror primitive) — see [Finding: DC segment mirror limit](../findings/dc-segment-mirror-limit.md).

### Two NICs on mgmt01

mgmt01 has a second adapter added in GNS3 purely for monitoring, and the split matters:

- **ens3** has the IP `192.168.122.20`, routes traffic, runs SSH/Docker/management. It is a participant on the network.
- **ens4** has no IP, no routing, no identity. It only listens — the kernel hands every received frame straight to Zeek via `libpcap`/`AF_PACKET` with no TCP/IP stack processing.

A silent listener prevents the kernel from reacting to traffic it was never meant to process (no stray TCP RSTs, ICMP unreachables, or ARP replies from the sensor) and keeps mgmt01's own management chatter (SSH keepalives, Docker pulls, NTP, apt) out of the RITA baseline.

---

## Configuration

### Monitor interface (ens4)

ens4 is brought up promiscuous with all hardware offloading disabled, so Zeek sees real on-wire frames rather than NIC-reassembled jumbo segments:

```bash
sudo ip link set ens4 up
sudo ip link set ens4 promisc on
sudo ethtool -K ens4 rx off tx off sg off tso off gso off gro off lro off
```

This is made persistent through a `ens4-monitor.service` systemd unit. A correct state shows `<BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP>` on the link.

### Internal networks (networks.cfg)

Zeek needs to know which subnets are internal so its logs label local-vs-remote correctly:

```
192.168.122.0/24    Core Management LAN (Hub1)
10.0.0.0/24         DC LAN
172.16.10.0/24      Site1 LAN
```

### Cluster (node.cfg)

A manager, a proxy and three workers — one worker per capture interface:

```ini
[manager]
type=manager
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

The manager aggregates every worker's output into `/opt/zeek/spool/manager/`, which is where RITA imports from. See the log-path gotcha below.

### GRE receivers

The `ip_gre` kernel module is loaded persistently, and the two tunnel endpoints are created and set promiscuous, persisted via a `gre-tunnels.service` unit:

```bash
sudo ip tunnel add gre-site1 mode gre local 192.168.122.20 remote 192.168.122.31 ttl 255
sudo ip link set gre-site1 up && sudo ip link set gre-site1 promisc on
```

Zeek's `tunnel.log` confirms decapsulation with `Tunnel::GRE  Tunnel::DISCOVER` entries for both remote endpoints.

### Start and persist

```bash
sudo zeek start
sudo zeek enable   # auto-start on reboot
```

The `zeek` wrapper manages the Docker container with `--network=host` and `--cap-add=NET_ADMIN,NET_RAW`.

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [RITA](rita.md) | → (feeds) | Zeek writes `conn/dns/ssl/http/ntp` logs to `/opt/zeek/logs/`; RITA imports them into ClickHouse for behavioral scoring. This is the whole point of Zeek here. |
| [GNS3 topology](gns3.md) | dependency | Core visibility relies on the Hub1 swap; remote visibility relies on GRE tunnels from VyOS/OPNsense. |
| [Suricata](suricata.md) | complementary | Same wire, different question: Suricata matches signatures per packet/flow; Zeek records flows for RITA's behavioral analysis over time. Behavioral vs signature. |

---

## Known issues / gotchas

**Cluster mode changes the log path.** In standalone mode Zeek writes to `/opt/zeek/spool/zeek/`; in cluster mode the manager aggregates into `/opt/zeek/spool/manager/`. The `current` symlink must follow:

```bash
sudo rm -f /opt/zeek/logs/current
sudo ln -s /opt/zeek/spool/manager /opt/zeek/logs/current
```

**VyOS mirror needs a two-step commit.** VyOS validates the mirror *target* at commit time, so the GRE tunnel must already exist before the mirror rules reference it. Commit the tunnel first, then commit the `mirror ingress/egress` rules. A single combined commit fails validation. See [Decision: Hub vs switch visibility](../decisions/hub-vs-switch-visibility.md).

**DC inner-segment traffic is not captured.** The `gre-dc` worker sees tunnel discovery but no mirrored frames, because OPNsense/FreeBSD has no kernel mirror equivalent to VyOS. DC traffic is still visible at the core when it transits OPNsense's WAN, but the `10.0.0.x ↔ 10.0.0.x` inner perspective is missing. Acceptable here because the DC has exactly one workload host, so there is no lateral movement to observe. See [Finding: DC segment mirror limit](../findings/dc-segment-mirror-limit.md).

**Disable NIC offloading or flows look wrong.** With offloading on, the kernel hands Zeek reassembled super-frames and stream reconstruction degrades. The `ethtool -K` line above is not optional for a monitor interface.

**Not integrated with the sandbox event bus.** Zeek/RITA findings are not published to NATS and do not feed the [Control Daemon](control-daemon.md). Detections live in RITA's CLI views (and, in the RITA→RPZ pipeline, in a feed file) — there is no automatic quarantine path on the sandbox from this sensor yet.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: RITA](rita.md)
- [Component: Suricata](suricata.md)
- [Component: GNS3](gns3.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Decision: Hub vs switch visibility](../decisions/hub-vs-switch-visibility.md)
- [Finding: DC segment mirror limit](../findings/dc-segment-mirror-limit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
