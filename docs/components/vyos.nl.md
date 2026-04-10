---
title: "VyOS — SD-WAN-gateway (site01)"
tags: [network, sase, firewall]
---

# VyOS — SD-WAN-gateway (site01)

**Rol:** SASE-gateway voor de remote site — biedt WAN-connectiviteit en NAT voor het Site-LAN-segment, invulling gevend aan de SD-WAN-pijler van de SASE-stack. Equivalent aan een Zscaler Zero Trust Branch edge-appliance.  
**Status:** ✅ Gedeployed — site01 operationeel op `192.168.122.33`. sitepc01 (clientnode op Site-LAN) heeft geen OS geïnstalleerd.  
**Configuratielocatie:** `/etc/vyos/config.boot` op site01

<!-- TODO: insufficient source detail — VyOS-configuratiestappen zijn niet gedocumenteerd in de bronmaterialen, buiten topologie/adressering -->

---

## Hoe het werkt in deze stack

VyOS draait als `site01` op het Switch-WAN-segment (`192.168.122.0/24`, eth0) met een tweede interface op Switch-Site (`172.16.10.0/24`, eth1 gateway `172.16.10.1`). Het biedt:

- WAN-connectiviteit voor het Site-LAN via NAT
- Routing voor sitepc01 (wanneer gedeployed) richting internet

De site-branch wordt beschouwd als een niet-vertrouwde locatie — equivalent aan een café-netwerk. Alle beveiligingshandhaving vindt plaats op pop01, niet bij de site. sitepc01 zal een NetBird-client gebruiken voor identiteitsgebaseerde connectiviteit, identiek aan mobile01. VyOS biedt alleen de WAN-uplink — er wordt geen beveiligingsbeleid toegepast bij de branch.

**SD-WAN-equivalent:** Dit weerspiegelt de Zero Trust SD-WAN-architectuur van Zscaler waarbij de branch-appliance connectiviteit biedt terwijl de Zero Trust Exchange inspectie verzorgt. Er bestaan geen IPsec-tunnels tussen site01 en pop01 — sitepc01 bereikt resources uitsluitend via de NetBird WireGuard-overlay.

**IP-adressering:**
- eth0 (WAN): `192.168.122.33` (gateway `192.168.122.1`)
- eth1 (Site-LAN): `172.16.10.1/24`
- Externe toegang via GNS3-host: SSH port-forward naar `10.158.10.67:7033`

**Hosts-vermelding voor NetBird:** VyOS lost `netbird.sandbox.local` op via statische host-mapping:
```
set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23
```

---

## Bekende status

sitepc01 is bekabeld op Switch-Site (`172.16.10.50/24`) maar heeft geen OS geïnstalleerd. De VyOS-gateway en Site-LAN zijn topologisch compleet; de endpoint-node wacht op OS-installatie en NetBird-inschrijving.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: GNS3](gns3.md)
- [Component: NetBird](netbird.md)
- [Concept: SASE](../concepts/sase.md)
- [Beslissing: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)
