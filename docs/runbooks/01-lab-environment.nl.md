---
title: "Runbook: Labomgeving"
tags: [runbook, gns3, proxmox, virtualization]
---

# Runbook: Labomgeving

**Bron:** `raw/Doc5_Virtualisatie_Topologie.md`
**Node(s):** GNS3-host (poc-1a, `10.158.10.67`) op Proxmox
**Vereisten:** Proxmox hypervisor met ondersteuning voor geneste virtualisatie, QCOW2-images voor alle VM's
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] Proxmox-server beschikbaar (`10.133.0.21:8006`)
- [ ] Ubuntu 24.04 Server ISO of image
- [ ] QCOW2-images: OPNsense 25.1, Ubuntu 24.04, VyOS
- [ ] DHCP-reservering op schoolnetwerk VLAN 158 voor `10.158.10.67`

---

## Stap 1: Proxmox Ubuntu VM aanmaken

Maak VM-103 (`poc-1a`) aan op Proxmox met de volgende resources:

```
RAM:    50 GB
vCPU:   12
Schijf: 325 GB
IP:     10.158.10.67 (DHCP-reservering op VLAN labo158)
Gebruiker: admin-1a
```

Stel het Proxmox CPU-type in op `host` — dit stuurt VMX/SVM-instructies door naar de VM, wat vereist is voor geneste QEMU binnen GNS3.

**Verificatie:**

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
# Moet > 0 zijn

lsmod | grep kvm
# Moet kvm_intel of kvm_amd tonen
```

---

## Stap 2: GNS3 Server installeren

```bash
sudo add-apt-repository ppa:gns3/ppa -y
sudo apt update
sudo apt install -y gns3-server qemu-kvm libvirt-daemon-system \
    virtinst bridge-utils ubridge
```

Tijdens de installatie: beantwoord **Ja** op beide dialoogvensters (pakketopname + wireshark).

```bash
sudo usermod -aG kvm,libvirt,ubridge,wireshark admin-1a
```

Activeer het libvirt standaardnetwerk — dit levert het WAN-segment (`192.168.122.0/24`) met NAT:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

**Verificatie:**

```bash
ip addr show virbr0
# Moet inet 192.168.122.1/24 tonen

systemctl status gns3
# Moet active (running) tonen
```

Zie [Component: GNS3](../components/gns3.nl.md) voor de volledige systemd-serviceconfiguratie.

---

## Stap 3: Topologie aanmaken in GNS3 GUI

Verbind de GNS3 GUI vanaf je laptop: `Edit → Preferences → Server → Remote server: 10.158.10.67:3080`.

**Nodes aanmaken op het canvas:**

| Node | Type | Naam |
|------|------|------|
| NAT-node | GNS3 ingebouwd | `NAT-Internet` |
| Laag-2-switch | Ethernet switch | `Switch-WAN` |
| Laag-2-switch | Ethernet switch | `Switch-LAN` |
| Laag-2-switch | Ethernet switch | `Switch-Site` |
| OPNsense 25.1 | QEMU (QCOW2) | `pop01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `mgmt01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `dc01` |
| VyOS | QEMU (QCOW2) | `site01` |
| Ubuntu 24.04 | QEMU (QCOW2) | `sitepc01` |

`mobile01` (Windows 11) draait als VMware VM op de laptop van een teamlid — het is **geen** GNS3-node. Het maakt uitsluitend verbinding via de NetBird WireGuard-tunnel.

**Bekabeling:**

```
NAT-Internet ──── Switch-WAN        (WAN-segment, 192.168.122.0/24)
Switch-WAN   ──── pop01  vtnet0     (WAN-interface)
Switch-WAN   ──── mgmt01 ens3       (WAN-interface)
Switch-WAN   ──── site01 eth0       (WAN-interface)
Switch-LAN   ──── pop01  vtnet1     (LAN naar DC-LAN)
Switch-LAN   ──── dc01   ens3       (DC-LAN)
Switch-Site  ──── site01 eth1       (Site-LAN)
Switch-Site  ──── sitepc01 ens3     (Site-LAN)
```

**Verificatie:** Alle nodes verschijnen op het canvas met de juiste bekabeling. Klik met rechts → Start elke node; controleer consoletoegang.

---

## Stap 4: VM-resources configureren

Klik met rechts op elke node → Configure → General settings:

| VM | RAM | vCPU | Adapters | Opmerking |
|----|-----|------|----------|-----------|
| pop01 (OPNsense) | 8192 MB | 2 | 3 (vtnet0, vtnet1, vtnet2) | **8 GB vereist** — zie waarschuwing hieronder |
| mgmt01 (Ubuntu) | 16384 MB | 4 | 1 | Docker-stack |
| dc01 (Ubuntu) | 4096 MB | 2 | 1 | Datacentersimulatie |
| site01 (VyOS) | 1024 MB | 1 | 2 (eth0, eth1) | SD-WAN-gateway |
| sitepc01 (Ubuntu) | 4096 MB | 2 | 1 | Nog geen OS geïnstalleerd |

> **Valkuil: pop01 heeft 8 GB nodig, niet 4 GB.** Het handboek schrijft 4 GB voor, maar ClamAV (~1,2 GB) + Suricata (~760 MB + 4 GB Hyperscan-compilatiepiek) + Squid (~400 MB) gelijktijdig overschrijden 6 GB. Bij 4 GB beëindigt de OOM-killer van FreeBSD processen zonder logboekregels te schrijven.
> Zie [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.nl.md) voor geheugenanalyse.

---

## Stap 5: IP-adressering en poortdoorsturing configureren

**WAN-segment (192.168.122.0/24, libvirt NAT-gateway .1):**

| Node | WAN-IP | Externe toegang | Services |
|------|--------|-----------------|---------|
| pop01 | `192.168.122.13` | SSH: 7022, WebUI: 7443 | OPNsense + SASE-stack |
| mgmt01 | `192.168.122.23` | SSH: 7023 | Docker: NetBird, ioc2rpz, DLP |
| site01 | `192.168.122.33` | SSH: 7033 | VyOS |

**Geïsoleerde segmenten:**

| Segment | CIDR | Gateway | Nodes |
|---------|------|---------|-------|
| DC-LAN | `10.0.0.0/24` | `10.0.0.1` (pop01 vtnet1) | dc01: `10.0.0.100` |
| Site-LAN | `172.16.10.0/24` | `172.16.10.1` (site01 eth1) | sitepc01: `172.16.10.50` |

**Poortdoorsturing op GNS3-host** (`10.158.10.67`):

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

> **Valkuil: Gebruik altijd `-I FORWARD 1`, nooit `-A FORWARD`.** libvirt plaatst zijn eigen FORWARD-ketingregels met een REJECT voor niet-libvirt-verkeer. Een toegevoegde (`-A`) ACCEPT-regel komt na de libvirt REJECT en wordt nooit bereikt. Voeg op positie 1 in.
> Zie [Finding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.nl.md).

```bash
sudo iptables -I FORWARD 1 -d 192.168.122.13 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.23 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.33 -j ACCEPT
```

Bewaar met `iptables-save` en `netfilter-persistent`.

**SNI-gebaseerde routering voor NetBird Dashboard:** De GNS3-host draait een nginx stream-module op poort 443 die op basis van SNI-hostnaam routeert:

```
netbird.sandbox.local  →  192.168.122.23:443
```

**Verificatie:**

```bash
ssh admin-1a@10.158.10.67 -p 7022   # bereikt pop01
ssh mgmt@10.158.10.67 -p 7023       # bereikt mgmt01
```

---

## Stap 6: Snapshots en veilig afsluiten

> **Valkuil: GNS3 "Stop Node" = harde QEMU-kill = bestandssysteemcorruptie op OPNsense.** GNS3's "Stop node" stuurt SIGKILL naar QEMU. OPNsense op FreeBSD UFS met soft updates kan configuratie verliezen of niet meer opstarten na een onregelmatige afsluiting.
> Zie [Finding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.nl.md) voor gerelateerde UFS-corruptieproblemen.

**Correcte afsluitprocedure:**

```bash
# Vanuit OPNsense-console of SSH:
shutdown -h now
# Wacht tot de node grijs wordt in GNS3 (15-30 sec)
# DAN: GNS3 → Edit → Stop all nodes (of rechts klikken → Stop)
```

Veiligheidsnet: voeg `fsck_y_enable="YES"` toe aan `/etc/rc.conf` op pop01 voor automatische bestandssysteemcontrole bij opstarten.

**Snapshots aanmaken:**

```
GNS3-menu → Edit → Stop all nodes
Wacht tot alle nodes grijs zijn (15-30 sec)
→ File → Snapshots → Create snapshot
```

Alle nodes **moeten** gestopt zijn vóór het aanmaken van een snapshot.

**Huidige snapshots:**

| Naam | Inhoud |
|------|--------|
| `Fase2-ZTNA-Complete` | Volledige ZTNA-stack operationeel |
| `Fase3-Security-Complete` | Gepland — na volledige beveiligingsstack |

---

## Eindverificatie

- [ ] `egrep -c '(vmx|svm)' /proc/cpuinfo` > 0 op poc-1a
- [ ] `systemctl status gns3` toont active
- [ ] `ip addr show virbr0` toont `192.168.122.1/24`
- [ ] GNS3 GUI verbindt met `10.158.10.67:3080` en toont alle nodes
- [ ] Alle nodes starten en consoletoegang werkt
- [ ] Poortdoorsturing werkt: SSH via 7022/7023/7033
- [ ] Snapshot succesvol aangemaakt

---

## Gerelateerd

- [Component: GNS3](../components/gns3.nl.md)
- [Component: VyOS](../components/vyos.nl.md)
- [Beslissing: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.nl.md)
- [Finding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.nl.md)
- [Architectuur](../overview/architecture.nl.md)
