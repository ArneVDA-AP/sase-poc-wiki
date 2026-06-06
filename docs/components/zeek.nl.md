---
title: "Zeek: netwerkverkeer-analyse (behavioral telemetrie)"
tags: [zeek, rita, network, behavioral-analysis, beaconing, c2, gns3, docker, sase]
---

# Zeek: netwerkverkeer-analyse (behavioral telemetrie)

**Rol:** Passieve netwerksensor die elke flow op de SASE_POC-segmenten in gestructureerde protocollogs (conn, dns, ssl, http, ntp) ontleedt en zo de ruwe telemetrie produceert die [RITA](rita.md) op behavioral dreigingen analyseert.  
**Versie:** Zeek 6.2.1 (`activecm/zeek` Docker-image, `--network=host`)  
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending  
**Configuratielocatie:** `/opt/zeek/etc/node.cfg` (cluster), `/opt/zeek/etc/networks.cfg` (interne subnetten) op mgmt01 (`192.168.122.20`)

> **Sidenote, parallelle-stack scope.** Zeek draait op de **SASE_POC**-topologie van Rayan, op een eigen mgmt01 (`192.168.122.20`). Het is daar gevalideerd maar **nog niet in het sandbox-geheel geïntegreerd**: geen NATS-publicatie, geen koppeling met de [Control Daemon](control-daemon.md). Het staat naast de signature-gebaseerde [Suricata IDS](suricata.md) van de sandbox als additieve, behavioral laag, niet als vervanging. Waar de twee overlappen, is de sandbox de bron van waarheid.

---

## Hoe het werkt in deze stack

Zeek staat niet inline. Het leest frames van interfaces in promiscuous mode, reconstrueert TCP-streams, identificeert het applicatieprotocol en schrijft één gestructureerde logregel per event. Het velt geen oordeel en blokkeert niets. De taak is om ruwe packets om te zetten in de `conn.log`, `dns.log`, `ssl.log`, `http.log` en `ntp.log` records die RITA later afgraaft op beaconing- en C2-patronen. De detectie is gesplitst: Zeek ziet, RITA beslist.

De host is **mgmt01**, de management/operations-node van de SASE_POC-topologie, bewust gekozen:

- **Uitkijkpositie.** mgmt01 zit op het core-segment en termineert de GRE-tunnels van de remote gateways, waardoor het het breedste zicht in de topologie heeft.
- **Scheiding van planes.** De sensor op de management plane houdt hem weg van de workload-hosts. Als dc01 (de gesimuleerde datacenterserver) gecompromitteerd raakt, blijft de monitoring overeind.
- **Co-locatie.** Wazuh en de NetBird-management draaien al op mgmt01, dus de Zeek/RITA-telemetrie staat lokaal klaar voor correlatie zonder netwerk-hop.

### Drie segmenten capturen

De topologie heeft drie segmenten en geen enkele interface ziet ze allemaal. Zeek draait als **multi-worker cluster** zodat één proces per segment parallel kan capturen:

| Segment | Capture-methode | Worker | Interface |
|---------|-----------------|--------|-----------|
| Core (Hub1) | Hub floodt elk frame naar een aparte monitor-NIC | `worker-core` | `ens4` |
| Site1 | VyOS kernel-mirror kopieert eth1 in een GRE-tunnel naar mgmt01 | `worker-site1` | `gre-site1` |
| DC | GRE-tunnel opgezet, geen actieve inner-mirror (FreeBSD-beperking) | `worker-dc` | `gre-dc` |

Het core-segment is zichtbaar omdat Switch1 vervangen werd door een **GNS3 Hub** (`Hub1`). De Ethernet Switch van GNS3 gebruikt `ubridge`, een learning L2-bridge zonder SPAN of port mirroring, waardoor een monitorpoort erop na MAC-learning niets meer ziet. Een Hub floodt alle frames naar alle poorten per ontwerp, wat op het verkeersvolume van het lab functioneel gelijk is aan een productie-SPAN-poort. Zie [Beslissing: Hub vs. switch-zichtbaarheid](../decisions/hub-vs-switch-visibility.md).

De remote segmenten bereiken mgmt01 via **GRE-tunnels**. VyOS biedt native kernel-mirroring (`mirror ingress`/`mirror egress`), zodat elk frame op de Site1-interface in een GRE-tunnel wordt gekopieerd en aan de andere kant door Zeek wordt gedecapsuleerd. Het DC-segment kan dit niet vanaf OPNsense (FreeBSD mist een gelijkwaardige mirror-primitive). Zie [Bevinding: DC-segment mirror-limiet](../findings/dc-segment-mirror-limit.md).

### Twee NIC's op mgmt01

mgmt01 heeft een tweede adapter die in GNS3 puur voor monitoring is toegevoegd, en de splitsing is belangrijk:

- **ens3** heeft het IP `192.168.122.20`, routeert verkeer, draait SSH/Docker/management. Het is een deelnemer op het netwerk.
- **ens4** heeft geen IP, geen routing, geen identiteit. Het luistert alleen. De kernel geeft elk ontvangen frame direct aan Zeek via `libpcap`/`AF_PACKET`, zonder TCP/IP-stackverwerking.

Een stille listener voorkomt dat de kernel reageert op verkeer dat hij nooit had moeten verwerken (geen losse TCP RST's, ICMP unreachables of ARP-antwoorden van de sensor) en houdt mgmt01's eigen managementverkeer (SSH-keepalives, Docker pulls, NTP, apt) uit de RITA-baseline.

---

## Configuratie

### Monitor-interface (ens4)

ens4 wordt promiscuous gezet met alle hardware offloading uitgeschakeld, zodat Zeek echte on-wire frames ziet in plaats van door de NIC samengevoegde jumbo-segmenten:

```bash
sudo ip link set ens4 up
sudo ip link set ens4 promisc on
sudo ethtool -K ens4 rx off tx off sg off tso off gso off gro off lro off
```

Dit wordt persistent gemaakt via een `ens4-monitor.service` systemd-unit. Een correcte staat toont `<BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP>` op de link.

### Interne netwerken (networks.cfg)

Zeek moet weten welke subnetten intern zijn zodat zijn logs lokaal-vs-remote correct labelen:

```
192.168.122.0/24    Core Management LAN (Hub1)
10.0.0.0/24         DC LAN
172.16.10.0/24      Site1 LAN
```

### Cluster (node.cfg)

Een manager, een proxy en drie workers, één worker per capture-interface:

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

De manager aggregeert de output van elke worker in `/opt/zeek/spool/manager/`, waar RITA van importeert. Zie de log-pad-valkuil hieronder.

### GRE-ontvangers

De `ip_gre` kernelmodule wordt persistent geladen en de twee tunnel-endpoints worden aangemaakt en promiscuous gezet, persistent via een `gre-tunnels.service` unit:

```bash
sudo ip tunnel add gre-site1 mode gre local 192.168.122.20 remote 192.168.122.31 ttl 255
sudo ip link set gre-site1 up && sudo ip link set gre-site1 promisc on
```

Zeek's `tunnel.log` bevestigt de decapsulatie met `Tunnel::GRE  Tunnel::DISCOVER` entries voor beide remote endpoints.

### Starten en persistent maken

```bash
sudo zeek start
sudo zeek enable   # auto-start bij reboot
```

De `zeek`-wrapper beheert de Docker-container met `--network=host` en `--cap-add=NET_ADMIN,NET_RAW`.

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [RITA](rita.md) | → (voedt) | Zeek schrijft `conn/dns/ssl/http/ntp` logs naar `/opt/zeek/logs/`; RITA importeert ze in ClickHouse voor behavioral scoring. Dit is de hele bestaansreden van Zeek hier. |
| [GNS3-topologie](gns3.md) | afhankelijkheid | Core-zichtbaarheid leunt op de Hub1-swap; remote zichtbaarheid leunt op GRE-tunnels van VyOS/OPNsense. |
| [Suricata](suricata.md) | aanvullend | Zelfde wire, andere vraag: Suricata matcht signatures per packet/flow; Zeek registreert flows voor RITA's behavioral analyse over tijd. Behavioral vs. signature. |

---

## Bekende problemen / valkuilen

**Cluster mode verandert het log-pad.** In standalone mode schrijft Zeek naar `/opt/zeek/spool/zeek/`; in cluster mode aggregeert de manager naar `/opt/zeek/spool/manager/`. De `current`-symlink moet meeschuiven:

```bash
sudo rm -f /opt/zeek/logs/current
sudo ln -s /opt/zeek/spool/manager /opt/zeek/logs/current
```

**VyOS-mirror vereist een twee-staps commit.** VyOS valideert het mirror-*target* op commit-moment, dus de GRE-tunnel moet al bestaan voordat de mirror-regels ernaar verwijzen. Commit eerst de tunnel, daarna de `mirror ingress/egress` regels. Eén gecombineerde commit faalt de validatie. Zie [Beslissing: Hub vs. switch-zichtbaarheid](../decisions/hub-vs-switch-visibility.md).

**DC inner-segment verkeer wordt niet gecaptured.** De `gre-dc`-worker ziet tunnel-discovery maar geen gespiegelde frames, omdat OPNsense/FreeBSD geen kernel-mirror heeft die gelijk is aan VyOS. DC-verkeer is nog steeds zichtbaar op core-niveau wanneer het OPNsense's WAN passeert, maar het `10.0.0.x ↔ 10.0.0.x` inner-perspectief ontbreekt. Aanvaardbaar hier omdat het DC exact één workload-host heeft, dus er is geen lateral movement om waar te nemen. Zie [Bevinding: DC-segment mirror-limiet](../findings/dc-segment-mirror-limit.md).

**Schakel NIC-offloading uit of flows zien er verkeerd uit.** Met offloading aan geeft de kernel Zeek hersamengevoegde super-frames en degradeert de streamreconstructie. De `ethtool -K` regel hierboven is niet optioneel voor een monitor-interface.

**Niet geïntegreerd met de sandbox-eventbus.** Zeek/RITA-bevindingen worden niet naar NATS gepubliceerd en voeden de [Control Daemon](control-daemon.md) niet. Detecties leven in RITA's CLI-views (en, in de RITA→RPZ-pipeline, in een feed-bestand). Er is vanaf deze sensor nog geen automatisch quarantaine-pad op de sandbox.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: RITA](rita.md)
- [Component: Suricata](suricata.md)
- [Component: GNS3](gns3.md)
- [Concept: Behavioral analyse](../concepts/behavioral-analysis.md)
- [Beslissing: Hub vs. switch-zichtbaarheid](../decisions/hub-vs-switch-visibility.md)
- [Bevinding: DC-segment mirror-limiet](../findings/dc-segment-mirror-limit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
