---
title: "Runbook: Zeek & RITA, netwerkbrede behavioral analysis"
tags: [runbook, zeek, rita, gre, network, beaconing]
---

# Runbook: Zeek & RITA, netwerkbrede behavioral analysis

**Bron:** `rayan_zeek-rita-documentation.md` (apr 2026), `Verslag07` (mei 2026)
**Node(s):** mgmt01 (`192.168.122.20`) Zeek-cluster + RITA; VyOS-1 + OPNsense-1 GRE-mirror-bronnen
**Vereisten:** GNS3 lab-topologie operationeel; mgmt01 met een tweede NIC voor monitoring
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending

> **Scope.** Deze runbook bouwt de Zeek/RITA-monitoring op de **parallelle stack**. Het is een
> PoC-within-the-PoC: end-to-end gevalideerd, maar nog niet in het sandbox-geheel geïntegreerd.
> mgmt01 is hier `192.168.122.20` (de parallelle stack), niet de mgmt01 van de sandbox.

---

## Wat dit opbouwt

Zeek als multi-worker cluster op mgmt01, dat drie segmenten tegelijk bekijkt en uurlijkse logs in
RITA's ClickHouse-backend voedt voor beaconing-analyse. Capture gebruikt twee mechanismen: een
GNS3 **Hub** floodt het core-segment naar een aparte monitor-NIC, en **GRE-tunnels** dragen
kernel-gemirrorde frames van de remote segmenten.

```
Core (Hub1) ──flood──► ens4 ──► worker-core ┐
Site1 ──VyOS mirror──► GRE ──► gre-site1 ──► worker-site1 ├─► manager ─► /opt/zeek/logs/ ─► RITA
DC ────(partieel)────► GRE ──► gre-dc ─────► worker-dc   ┘
```

---

## Vereistenchecklist

- [ ] GNS3-topologie draait, core-segment bereikbaar
- [ ] mgmt01 heeft twee NIC's: `ens3` (management, heeft IP) en `ens4` (monitor, geen IP)
- [ ] Docker-daemon draait op mgmt01 (Zeek en RITA draaien als containers)
- [ ] Root/sudo op mgmt01, VyOS-1 en OPNsense-1

---

## Stap 1: vervang de core-switch door een Hub

De Ethernet Switch van GNS3 (`ubridge`) forwardt geleerde unicast alleen naar de bestemmingspoort
en heeft geen SPAN-functie, dus een monitor-NIC erop ziet niets tussen andere nodes. Vervang hem
door een GNS3 **Hub** (`Hub1`), die elk frame naar elke poort floodt. De monitor-NIC ontvangt dan
elke conversatie.

Zie [Beslissing: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.nl.md) voor waarom dit
het functionele equivalent van een SPAN-poort is.

**Gebruikte Hub1-poortindeling:**

```
Ethernet0  NAT1            nat0
Ethernet1  Windows11-1     eth0
Ethernet2  VyOS-1          eth0
Ethernet3  OPNsense-1      eth0
Ethernet4  mgmt01          ens3   (management, heeft IP)
Ethernet5  mgmt01          ens4   (monitor, geen IP)
```

> **Waarom twee NIC's:** `ens3` doet mee aan het netwerk (IP, routing, SSH, Docker). `ens4`
> luistert alleen: geen IP, geen routing, geen identiteit. De kernel geeft elk ontvangen frame aan
> Zeek via `libpcap`/`AF_PACKET` zonder TCP/IP-verwerking. Dat voorkomt dat de kernel antwoordt met
> TCP RST's / ICMP unreachables / ARP op verkeer dat hij nooit hoorde te verwerken, en houdt het
> eigen management-verkeer van mgmt01 (SSH keepalives, Docker pulls, NTP, apt) uit de
> RITA-baseline.

---

## Stap 2: configureer de monitor-interface (ens4)

Zet `ens4` op in promiscuous mode en schakel alle hardware-offloading uit, anders geeft de kernel
Zeek herassembleerde jumbo frames in plaats van de echte wire frames.

```bash
sudo ip link set ens4 up
sudo ip link set ens4 promisc on
sudo ethtool -K ens4 rx off tx off sg off tso off gso off gro off lro off
```

Maak het persistent over reboots met een systemd oneshot-unit:

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

**Verificatie:** de interface moet `PROMISC` tonen:

```bash
ip link show ens4 | grep PROMISC
# 3: ens4: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 9216 ...
```

---

## Stap 3: mirror het Site1-segment vanaf VyOS (GRE)

VyOS doet native kernel mirroring: kopieer elk frame op `eth1` (het Site1-segment) in een
GRE-tunnel gericht op mgmt01.

> **Gotcha: dit kost twee commits.** VyOS valideert het mirror-doel op commit-tijd, en de tunnel
> bestaat nog niet binnen dezelfde commit. Commit eerst de tunnel, dan de mirror. Zie
> [Bevinding: VyOS GRE two-step commit](../findings/vyos-gre-two-step-commit.nl.md).

```bash
configure

# Commit 1: maak de tunnel
set interfaces tunnel tun0 encapsulation 'gre'
set interfaces tunnel tun0 source-address '192.168.122.31'
set interfaces tunnel tun0 remote '192.168.122.20'
commit

# Commit 2: voeg de mirror toe nu tun0 bestaat
set interfaces ethernet eth1 mirror ingress 'tun0'
set interfaces ethernet eth1 mirror egress 'tun0'
commit
save
exit
```

> **Parameternamen verschillen per VyOS-versie.** Deze stack had `source-address` en `remote`
> nodig, niet `local-ip` / `remote-ip`. Als een parameter geweigerd wordt, draai
> `set interfaces tunnel tun0 ?`.

---

## Stap 4: zet de DC GRE-tunnel op vanaf OPNsense (partieel)

```bash
ifconfig gre0 create
ifconfig gre0 tunnel 192.168.122.11 192.168.122.20
ifconfig gre0 up
```

> **Verwacht hier geen gemirrord verkeer.** OPNsense is FreeBSD, dat geen kernel-level interface
> mirror heeft. De tunnel komt op en Zeek ontdekt hem, maar het inner DC-segment (`10.0.0.x`) wordt
> er niet in gekopieerd. DC-verkeer wordt nog steeds op het core-niveau gezien wanneer het de
> WAN-interface van OPNsense passeert. Dit is een geaccepteerde PoC-beperking. Zie
> [Bevinding: DC inner-segment mirror-limiet](../findings/dc-segment-mirror-limit.nl.md).

De OPNsense-tunnelconfig is niet persistent. Draai de drie `ifconfig`-regels opnieuw na reboot of
voeg een `/etc/rc.local`-entry toe.

---

## Stap 5: ontvang de GRE-tunnels op mgmt01

Laad de GRE-kernelmodule en maak de twee receiver-interfaces:

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

Persistent maken met een systemd oneshot-unit (geordend na `ens4-monitor.service`):

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

## Stap 6: configureer het Zeek-cluster

Zeek/RITA zijn geïnstalleerd via de officiële `activecm/rita` v5.1.1 Ansible-installer; de wrappers
staan op `/usr/local/bin/zeek` en `/usr/local/bin/rita`. Twee configbestanden zijn van belang.

**`/opt/zeek/etc/networks.cfg`**: vertel Zeek welke subnets intern zijn:

```
192.168.122.0/24    Core Management LAN (Hub1)
10.0.0.0/24         DC LAN (OPNsense-1 → Switch2 → Ubuntu-Server-DC-1)
172.16.10.0/24      Site1 LAN (VyOS-1 → Switch3 → Ubuntu-Server-Site-1)
```

**`/opt/zeek/etc/node.cfg`**: één worker per capture-interface:

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

> **Gotcha: cluster-modus verandert het log-pad.** In standalone-modus schrijft Zeek naar
> `/opt/zeek/spool/zeek/`. In cluster-modus aggregeert de manager naar `/opt/zeek/spool/manager/`.
> Wijs de `current`-symlink opnieuw:
> ```bash
> sudo rm -f /opt/zeek/logs/current
> sudo ln -s /opt/zeek/spool/manager /opt/zeek/logs/current
> ```

---

## Stap 7: start Zeek en de RITA-backend

```bash
# Zeek (wrapper draait de container met --network=host, --cap-add=NET_ADMIN,NET_RAW)
sudo zeek start
sudo zeek enable      # auto-start bij reboot

# RITA-backend: ClickHouse + syslog-ng
cd /opt/rita
sudo docker compose up -d

# Wacht tot ClickHouse healthy meldt
docker inspect rita-clickhouse --format='{{.State.Health.Status}}'
# Verwacht: healthy
```

**Verifieer het cluster:** alle vijf processen draaien:

```bash
sudo zeek status
# manager / proxy-1 / worker-core / worker-site1 / worker-dc  → running
```

**Verifieer GRE-decapsulatie:** Zeeks `tunnel.log` moet beide tunnels ontdekken:

```
192.168.122.20 → 192.168.122.11    Tunnel::GRE    Tunnel::DISCOVER
192.168.122.20 → 192.168.122.31    Tunnel::GRE    Tunnel::DISCOVER
```

> Een `DISCOVER`-regel bevestigt de tunnel, niet dat er verkeer in loopt. `gre-dc` ontdekt wel maar
> blijft leeg (Stap 4).

---

## Stap 8: importeer logs in RITA

```bash
# Snelle test: één dag in een wegwerp-database
sudo rita import -l /opt/zeek/logs/2026-04-21/ --database sase_test

# Volledige rolling baseline: meerdaags venster
sudo rita import -l /opt/zeek/logs/ --database sase_poc --rolling
```

> **Gotcha: geen koppeltekens in RITA v5 database-namen.** Gebruik `sase_poc`, niet `sase-poc`.

Een rolling import van ~12 dagen verwerkte in ongeveer 7,5 minuten. Afgekapte logs van uren waarin
Zeek mid-hour herstartte geven niet-fatale waarschuwingen en worden alsnog verwerkt.

---

## Stap 9: draai beacon-analyse

```bash
sudo rita view sase_poc --beacon            # beaconing (primaire use case)
sudo rita view sase_poc --long-connection   # langdurige connecties (mogelijke C2)
sudo rita view sase_poc --c2-over-dns       # DNS-gebaseerde C2-indicatoren
sudo rita view sase_poc --threat-intel      # threat-intel feed-matches
```

**Hoe "werkend" eruitziet.** Zonder malware in het lab zijn de top-beacons legitieme periodieke
services. Dat is precies het bewijs dat de detectie-pipeline werkt:

| Source | Destination | Beacon score | Lezing |
|--------|-------------|--------------|--------|
| 192.168.122.20 | pkgs.netbird.io | 100.00% | NetBird update-check, perfect regelmatig |
| 192.168.122.20 | 185.125.190.56 | 99.00% | Ubuntu NTP (Canonical) |
| 192.168.122.20 | 8.8.8.8 | 73.40% | Google DNS, periodiek |
| 192.168.122.11 | 212.132.97.26 | 68.10% | NTP vanaf OPNsense |

Een echte C2-beacon zou in deze zelfde lijst verschijnen; de analist onderzoekt elk patroon dat hij
niet herkent. Dat RITA deze regelmatige flows correct naar boven haalt bevestigt dat de
[behavioral-analysis](../concepts/behavioral-analysis.nl.md) pipeline operationeel is.

Deze beacon-view is ook de input voor de geautomatiseerde DNS-blokkeringspipeline. Zie
[Runbook 15: RITA → RPZ integratie](15-rita-rpz-integration.nl.md).

---

## Dekkingsoverzicht

| Segment | Dekking | Methode | Wat zichtbaar is |
|---------|---------|---------|------------------|
| Core (Hub1) | Volledig | Hub flood → ens4 → worker-core | Al het inter-node verkeer |
| Site1 | Volledig | VyOS eth1 kernel mirror → GRE → worker-site1 | Al het Site1-interne + Site1 ↔ internet (pre-routing) |
| DC | Partieel | Routed verkeer op core; GRE op maar geen actieve mirror | DC ↔ internet op core; `10.0.0.x` inner-perspectief niet gecapteerd (FreeBSD-limiet) |

Geschatte totale topologie-dekking: **~90%**. De ontbrekende ~10% is DC inner-segment-verkeer, dat
geen praktische impact heeft in de single-server DC.

---

## Reboot-gedrag

Deze starten automatisch: `ens4-monitor.service`, `gre-tunnels.service`, de `zeek`-container
(`sudo zeek enable`), en `rita-clickhouse` / `rita-syslog-ng` (`restart: unless-stopped`). VyOS
houdt zijn config via `save`; OPNsense heeft de `ifconfig gre0 ...`-regels opnieuw nodig (of een
`/etc/rc.local`-entry).

---

## Checklist

- [ ] Core-switch vervangen door Hub1; ens4 op een aparte Hub-poort
- [ ] `ens4` PROMISC, offloading uit, gepersisteerd via systemd
- [ ] VyOS GRE-tunnel + mirror in twee stappen gecommit
- [ ] OPNsense GRE-tunnel aangemaakt (DC partieel, verwacht)
- [ ] `gre-site1` en `gre-dc` op en promiscuous op mgmt01
- [ ] `node.cfg` heeft worker-core / worker-site1 / worker-dc; `current`-symlink → manager
- [ ] `sudo zeek status` toont alle 5 processen running
- [ ] `tunnel.log` ontdekt beide GRE-tunnels
- [ ] RITA-import in `sase_poc` (underscore-naam) slaagt
- [ ] `rita view sase_poc --beacon` toont de legitieme periodieke services

---

## Gerelateerd

- [Component: Zeek](../components/zeek.nl.md)
- [Component: RITA](../components/rita.nl.md)
- [Component: GNS3](../components/gns3.nl.md)
- [Component: VyOS](../components/vyos.nl.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.nl.md)
- [Beslissing: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.nl.md)
- [Bevinding: VyOS GRE two-step commit](../findings/vyos-gre-two-step-commit.nl.md)
- [Bevinding: DC inner-segment mirror-limiet](../findings/dc-segment-mirror-limit.nl.md)
- [Runbook 15: RITA → RPZ integratie](15-rita-rpz-integration.nl.md)
