---
title: "GNS3: Virtualisatie en Lab-topologie"
tags: [network, architecture, sase]
---

# GNS3: Virtualisatie en Lab-topologie

**Rol:** Lab-infrastructuurplatform dat alle SASE-stack-VMs host en de netwerktopologie simuleert (WAN-segment, DC-LAN, Site-LAN) op één fysieke server met geneste QEMU/KVM-virtualisatie.  
**Status:** ✅ Volledig operationeel  
**Configuratielocatie:** GNS3-project op `poc-1a` (Ubuntu 24.04 VM op `10.158.10.67`), Proxmox VM-103

---

## Hoe het werkt in deze stack

GNS3 draait als een systemd-service op een Ubuntu 24.04 VM (`poc-1a`) binnen Proxmox. De GNS3 Server beheert QEMU/KVM-virtuele machines en Ethernet-switch-nodes. Teamleden verbinden via de GNS3 GUI-applicatie van hun eigen laptop naar `10.158.10.67:3080`: meerdere mensen kunnen gelijktijdig aan dezelfde topologie werken, elk op hun eigen VM-console.

**Waarom GNS3 en niet EVE-NG:** EVE-NG heeft één webinterface: twee gebruikers die aan dezelfde node werken blokkeren elkaar. Het client-server-model van GNS3 stelt alle vier teamleden in staat gelijktijdig te werken. EVE-NG gebruikt ook OVA-imports die incompatibel zijn met de QCOW2-images die OPNsense, VyOS en Ubuntu als primair downloadformaat distribueren.

**Geneste virtualisatie:** GNS3 gebruikt QEMU/KVM voor VMs binnen de topologie. De Proxmox-host geeft CPU-virtualisatie-extensies (VMX/SVM) door naar de Ubuntu VM met `cpu type = host`. Verifieer met `egrep -c '(vmx|svm)' /proc/cpuinfo` (moet > 0 zijn).

**libvirt als WAN-segment:** De NAT-node van GNS3 koppelt aan het standaardnetwerk van libvirt (`virbr0`, `192.168.122.0/24`). libvirt verwerkt NAT en internetrouting automatisch; geen handmatige iptables-regels nodig voor internettoegang van VMs. Dit vervangt de EVE-NG IP-forwarding- en NAT-stappen uit het handboek.

---

## Topologie

### Nodes

| Node | Type | OS | RAM | vCPU |
|------|------|----|-----|------|
| pop01 | QEMU (QCOW2) | OPNsense 25.1 | 8192 MB | 2 |
| mgmt01 | QEMU (QCOW2) | Ubuntu 24.04 | 16384 MB | 4 |
| dc01 | QEMU (QCOW2) | Ubuntu 24.04 | 4096 MB | 2 |
| site01 | QEMU (QCOW2) | VyOS | 1024 MB | 1 |
| sitepc01 | QEMU (QCOW2) | Tiny11 (Windows 11) | 4096 MB | 2 |

pop01 vereist 8 GB RAM voor ClamAV + Suricata + Squid die gelijktijdig draaien (het handboek specificeert 4 GB, dit is onvoldoende).

### Bekabeling

```
NAT-Internet ──── Switch-WAN        (WAN-segment, 192.168.122.0/24)
Switch-WAN   ──── pop01  vtnet0     (pop01 WAN-interface)
Switch-WAN   ──── mgmt01 ens3       (mgmt01 WAN-interface)
Switch-WAN   ──── site01 eth0       (site01 WAN-interface)
Switch-LAN   ──── pop01  vtnet1     (pop01 LAN → DC-LAN)
Switch-LAN   ──── dc01   ens3       (dc01 DC-LAN)
Switch-Site  ──── site01 eth1       (site01 Site-LAN)
Switch-Site  ──── sitepc01 Ethernet (sitepc01 Site-LAN)
```

```
NAT-Internet ──── mobile01 NIC1     (aparte TAP — simulatie van extern netwerk)
```

mobile01 is een VMware VM op de laptop van een teamlid, geen GNS3-node. Het verbindt direct via zijn eigen netwerkadapter en simuleert een echte remote gebruiker buiten de GNS3-topologie. Het bereikt de SASE-stack uitsluitend via de NetBird WireGuard-tunnel.

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
| Site-LAN | `172.16.10.0/24` | `172.16.10.1` (site01 eth1) | sitepc01: `172.16.10.10` |

**NetBird-overlay (100.64.0.0/10):**

| Node | Overlay-IP | Rol |
|------|-----------|-----|
| pop01 | `100.70.154.79` | Data plane, exit node, DNS-primary |
| mgmt01 | `100.70.135.241` | Management plane, WPAD-server |
| mobile01 | `100.70.95.98` | Remote client (managed Windows) |

### Port-forwards voor externe toegang

Port-forwards via iptables DNAT op de GNS3-host (`10.158.10.67`). Volgorde van regels is kritiek; zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

```bash
iptables -t nat -A PREROUTING -p tcp --dport 7022 -j DNAT --to 192.168.122.13:22
iptables -t nat -A PREROUTING -p tcp --dport 7443 -j DNAT --to 192.168.122.13:443
iptables -t nat -A PREROUTING -p tcp --dport 7023 -j DNAT --to 192.168.122.23:22
iptables -t nat -A PREROUTING -p tcp --dport 7033 -j DNAT --to 192.168.122.33:22
# FORWARD-regels: gebruik -I FORWARD 1, niet -A FORWARD
iptables -I FORWARD 1 -d 192.168.122.0/24 -j ACCEPT
```

De GNS3-host draait ook nginx SNI-stream-passthrough op poort 443, routeert op basis van hostnaam:

| SNI-hostnaam | Doel |
|-------------|------|
| `netbird.sandbox.local` | `192.168.122.23:443` (sandbox mgmt01) |
| `netbird.sase.local` | `192.168.122.20:443` (teamproject mgmt01) |

Twee SNI-entries vermijden poortconflicten tussen de sandbox- en de teamprojectstack die op dezelfde GNS3-host draaien.

---

## Snapshots

GNS3-snapshots omvatten het volledige project (alle VMs tegelijk). In tegenstelling tot VMware/Proxmox-snapshots per VM legt een GNS3-snapshot de volledige topologiestatus vast.

**Voor het maken van een snapshot:** Stop eerst alle nodes (Edit → Stop all nodes, wacht tot alle nodes grijs zijn).

**Terugzetten:** File → Snapshots → [naam] → Restore. GNS3 stopt automatisch alle actieve nodes.

Huidige snapshots:

| Naam | Inhoud |
|------|--------|
| `Fase2-ZTNA-Complete` | Volledige ZTNA-stack operationeel (NetBird, Zitadel, Entra ID-federatie) |
| `Fase3-Security-Complete` | Gepland, na Zeek/RITA-uitrol |

---

## Bekende problemen / valkuilen

**"Stop Node" = QEMU SIGKILL → bestandssysteemcorruptie op OPNsense:** de stopknop van GNS3 stuurt SIGKILL direct naar QEMU. OPNsense draait op FreeBSD UFS met soft updates, dat asynchroon schrijft. Een abrupte kill kan een inconsistente bestandssysteemstatus of verloren configuratie veroorzaken. Sluit OPNsense altijd correct af voordat je de GNS3-node stopt:
```bash
# Vanuit OPNsense-console of SSH:
shutdown -h now
# Wacht tot de node stopt in GNS3, ga dan verder
```
Mitigatie: voeg `fsck_y_enable="YES"` toe aan `/etc/rc.conf` op pop01. FreeBSD draait dan automatisch fsck bij de volgende boot na een onregelmatige afsluiting.

De OOM-killer kan dezelfde abrupte kill zonder waarschuwing aan elke QEMU-guest toebrengen: `poc-1a` committeert guest-RAM over en draait met 0 swap, dus een geheugenpiek kan de kernel een willekeurige guest laten beëindigen, pop01 inbegrepen. Zie [Bevinding: RAM-overcommit op de GNS3-host](../findings/gns3-host-ram-overcommit.md).

**iptables FORWARD-volgorde:** libvirt plaatst zijn eigen REJECT-regels in de FORWARD-keten. Regels toegevoegd met `-A FORWARD` komen na die REJECT-regels terecht en matchen nooit. Gebruik altijd `-I FORWARD 1` voor ACCEPT-regels. Zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

**ubridge learning bridge beperkt verkeerzichtbaarheid:** Switch-WAN gebruikt ubridge, een lerende L2-brug. Na MAC-learning gaan unicast-frames alleen naar de juiste poort, niet geflood. mgmt01 in promiscuous mode op ens3 ziet het verkeer van pop01 naar internet niet. Dit heeft impact op elke toekomstige Zeek/RITA-uitrol: valideer met `tcpdump -i ens3 -n host 8.8.8.8` op mgmt01 vóór uitrol.

**sitepc01 draait nu Tiny11 (Windows 11):** aanvankelijk aangemaakt als een lege node (geen OS), werd het later geïmporteerd als een Tiny11-image en ge-enrolld in de NetBird-overlay als `docent1` (Entra ID joined + Intune enrolled). Het is operationeel actief.

**Twee projecten delen deze GNS3-host: verifieer op project, niet op node-naam.** De sandbox en het teamproject draaien beide guests met de naam `mgmt01`/`VyOS-1`, dus een node-naam alleen is dubbelzinnig. De project-UUID's zijn `97e2bce6` = de sandbox (`PoC_Sandbox`: `mgmt01`, `pop01`, `VyOS-1`) en `b3b179f6` = het teamproject (`SASE_POC`: `Ubuntu-mgmt01-1`, `OPNsense-1`, `linuxpop01`, en andere). Bevestig de node-naar-project-mapping in de GNS3-GUI (projectnaam ↔ UUID ↔ node-set) voordat je een node stopt of op een guest ingrijpt. De `SASE_POC_IP_Referentie.md`-referentie is onbetrouwbaar: die dateert van vóór de sandbox (16 maart 2026) en benoemt de project-ID's omgekeerd.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: VyOS](vyos.md)
- [Component: Zeek](zeek.nl.md) — parallel-stack-sensor gehost op de SASE_POC-topologie
- [Component: Telemetry-stack](telemetry-stack.nl.md) — parallel-stack-observability (incl. GNS3-nodemonitoring)
- [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md)
- [Bevinding: RAM-overcommit op de GNS3-host](../findings/gns3-host-ram-overcommit.md)
- [Beslissing: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)
- [Runbook: Labomgeving](../runbooks/01-lab-environment.md)
