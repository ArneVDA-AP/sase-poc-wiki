---
title: "GNS3 — Virtualisatie en Lab-topologie"
tags: [network, architecture, sase]
---

# GNS3 — Virtualisatie en Lab-topologie

**Rol:** Lab-infrastructuurplatform — host alle SASE-stack-VMs en simuleert de netwerktopologie (WAN-segment, DC-LAN, Site-LAN) op één fysieke server met geneste QEMU/KVM-virtualisatie.  
**Status:** ✅ Volledig operationeel  
**Configuratielocatie:** GNS3-project op `poc-1a` (Ubuntu 24.04 VM op `10.158.10.67`), Proxmox VM-103

---

## Hoe het werkt in deze stack

GNS3 draait als een systemd-service op een Ubuntu 24.04 VM (`poc-1a`) binnen Proxmox. De GNS3 Server beheert QEMU/KVM-virtuele machines en Ethernet-switch-nodes. Teamleden verbinden via de GNS3 GUI-applicatie van hun eigen laptop naar `10.158.10.67:3080` — meerdere mensen kunnen gelijktijdig aan dezelfde topologie werken, elk op hun eigen VM-console.

**Waarom GNS3 en niet EVE-NG:** EVE-NG heeft één webinterface — twee gebruikers die aan dezelfde node werken blokkeren elkaar. Het client-server-model van GNS3 stelt alle vier teamleden in staat gelijktijdig te werken. EVE-NG gebruikt ook OVA-imports die incompatibel zijn met de QCOW2-images die OPNsense, VyOS en Ubuntu als primair downloadformaat distribueren.

**Geneste virtualisatie:** GNS3 gebruikt QEMU/KVM voor VMs binnen de topologie. De Proxmox-host geeft CPU-virtualisatie-extensies (VMX/SVM) door naar de Ubuntu VM met `cpu type = host`. Verifieer met `egrep -c '(vmx|svm)' /proc/cpuinfo` — moet > 0 zijn.

**libvirt als WAN-segment:** De NAT-node van GNS3 koppelt aan het standaardnetwerk van libvirt (`virbr0`, `192.168.122.0/24`). libvirt verwerkt NAT en internetrouting automatisch — geen handmatige iptables-regels nodig voor internettoegang van VMs.

---

## Topologie

### Nodes

| Node | Type | OS | RAM | vCPU |
|------|------|----|-----|------|
| pop01 | QEMU (QCOW2) | OPNsense 25.1 | 8192 MB | 2 |
| mgmt01 | QEMU (QCOW2) | Ubuntu 24.04 | 16384 MB | 4 |
| dc01 | QEMU (QCOW2) | Ubuntu 24.04 | 4096 MB | 2 |
| site01 | QEMU (QCOW2) | VyOS | 1024 MB | 1 |
| sitepc01 | QEMU (QCOW2) | Ubuntu 24.04 | 4096 MB | 2 |

pop01 vereist 8 GB RAM voor ClamAV + Suricata + Squid die gelijktijdig draaien (het handboek specificeert 4 GB — dit is onvoldoende).

### Bekabeling

```
NAT-Internet ──── Switch-WAN        (WAN-segment, 192.168.122.0/24)
Switch-WAN   ──── pop01  vtnet0     (pop01 WAN-interface)
Switch-WAN   ──── mgmt01 ens3       (mgmt01 WAN-interface)
Switch-WAN   ──── site01 eth0       (site01 WAN-interface)
Switch-LAN   ──── pop01  vtnet1     (pop01 LAN → DC-LAN)
Switch-LAN   ──── dc01   ens3       (dc01 DC-LAN)
Switch-Site  ──── site01 eth1       (site01 Site-LAN)
Switch-Site  ──── sitepc01 ens3     (sitepc01 Site-LAN)
```

mobile01 is een VMware VM op de laptop van een teamlid — geen GNS3-node. Het verbindt direct via zijn eigen netwerkadapter en simuleert een echte BYOD-gebruiker buiten de GNS3-topologie.

### IP-adressering

**WAN-segment (192.168.122.0/24, libvirt NAT-gateway .1):**

| Node | WAN-IP | Externe toegang |
|------|--------|-----------------|
| pop01 | `192.168.122.13` | SSH: 7022, WebUI: 7443 |
| mgmt01 | `192.168.122.23` | SSH: 7023 |
| site01 | `192.168.122.33` | SSH: 7033 |

**Geïsoleerde segmenten:**

| Segment | CIDR | Gateway | Nodes |
|---------|------|---------|-------|
| DC-LAN | `10.0.0.0/24` | `10.0.0.1` (pop01 vtnet1) | dc01: `10.0.0.100` |
| Site-LAN | `172.16.10.0/24` | `172.16.10.1` (site01 eth1) | sitepc01: `172.16.10.50` (geen OS) |

**NetBird-overlay (100.64.0.0/10):**

| Node | Overlay-IP |
|------|-----------|
| pop01 | `100.70.154.79` |
| mgmt01 | `100.70.135.241` |
| mobile01 | `100.70.95.98` |

### Port-forwards voor externe toegang

Port-forwards via iptables DNAT op de GNS3-host (`10.158.10.67`). Volgorde van regels is kritiek — zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

```bash
iptables -t nat -A PREROUTING -p tcp --dport 7022 -j DNAT --to 192.168.122.13:22
iptables -t nat -A PREROUTING -p tcp --dport 7443 -j DNAT --to 192.168.122.13:443
iptables -t nat -A PREROUTING -p tcp --dport 7023 -j DNAT --to 192.168.122.23:22
iptables -t nat -A PREROUTING -p tcp --dport 7033 -j DNAT --to 192.168.122.33:22
# FORWARD-regels: gebruik -I FORWARD 1, niet -A FORWARD
iptables -I FORWARD 1 -d 192.168.122.0/24 -j ACCEPT
```

De GNS3-host draait ook nginx SNI-stream-passthrough op poort 443, routeert `netbird.sandbox.local` naar `192.168.122.23:443`.

---

## Snapshots

GNS3-snapshots omvatten het volledige project — alle VMs tegelijk. In tegenstelling tot VMware/Proxmox-snapshots per VM legt een GNS3-snapshot de volledige topologiestatus vast.

**Voor het maken van een snapshot:** Stop eerst alle nodes (Edit → Stop all nodes, wacht tot alle nodes grijs zijn).

**Terugzetten:** File → Snapshots → [naam] → Restore. GNS3 stopt automatisch alle actieve nodes.

Huidige snapshots:

| Naam | Inhoud |
|------|--------|
| `Fase2-ZTNA-Complete` | Volledige ZTNA-stack operationeel (NetBird, Zitadel, Entra ID-federatie) |

---

## Bekende problemen / valkuilen

**"Stop Node" = QEMU SIGKILL → bestandssysteemcorruptie op OPNsense** — de stopknop van GNS3 stuurt SIGKILL direct naar QEMU. OPNsense draait op FreeBSD UFS met soft updates, dat asynchroon schrijft. Een abrupte kill kan een inconsistente bestandssysteemstatus of verloren configuratie veroorzaken. Sluit OPNsense altijd correct af:
```bash
shutdown -h now
```
Oplossing: voeg `fsck_y_enable="YES"` toe aan `/etc/rc.conf` op pop01.

**iptables FORWARD-volgorde** — libvirt plaatst zijn eigen REJECT-regels in de FORWARD-keten. Regels toegevoegd met `-A FORWARD` komen na die REJECT-regels terecht en matchen nooit. Gebruik altijd `-I FORWARD 1` voor ACCEPT-regels. Zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

**ubridge learning bridge beperkt verkeerzichtbaarheid** — Switch-WAN gebruikt ubridge, een lerende L2-brug. Na MAC-learning gaan unicast-frames alleen naar de juiste poort — niet geflood. mgmt01 in promiscuous mode op ens3 ziet het verkeer van pop01 naar internet niet.

**sitepc01 heeft geen OS geïnstalleerd** — de node bestaat in de topologie en is bekabeld naar Switch-Site, maar Ubuntu 24.04 is niet geïnstalleerd. Hij is niet operationeel actief.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: VyOS](vyos.md)
- [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md)
- [Beslissing: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)
