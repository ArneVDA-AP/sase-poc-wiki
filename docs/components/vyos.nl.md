---
title: "VyOS — SASE-gateway (site01)"
tags: [network, sase, sd-wan, nat, zero-trust]
---

# VyOS — SASE-gateway (site01)

**Rol:** SASE-gateway voor de remote site — biedt WAN-connectiviteit en NAT voor het Site-LAN, invulling gevend aan de SD-WAN-pijler via een Zero Trust Branch-model. Equivalent aan een Zscaler Zero Trust Branch edge-appliance.  
**Versie:** VyOS 1.4 (rolling)  
**Status:** ✅ Operationeel — site01 op `192.168.122.33`, sitepc01 (Tiny11) ingeschreven in sandbox  
**Configuratielocatie:** `/etc/vyos/config.boot` op site01

---

## Hoe het werkt in deze stack

VyOS draait als `site01` met twee interfaces: eth0 op het WAN-segment (`192.168.122.33`) en eth1 op het Site-LAN (`172.16.10.1/24`). De verantwoordelijkheden zijn bewust beperkt tot connectiviteit — het fungeert als NAT-gateway zodat apparaten op het Site-LAN het internet kunnen bereiken, maar het handhaaft zelf geen beveiligingsbeleid.

De branch volgt een **Zero Trust Branch-model**: de gehele site wordt behandeld als een niet-vertrouwde locatie, niet anders dan een Wi-Fi-netwerk in een koffiebar. Er zijn geen IPsec-tunnels tussen site01 en pop01. In plaats daarvan draait sitepc01 een [NetBird](netbird.md)-client en treedt toe tot de WireGuard-overlay met een identiteitsgebaseerde verbinding, exact hetzelfde pad volgend als elke mobiele gebruiker. Alle beveiligingsinspectie vindt plaats op de PoP, niet aan de branch-rand.

Dit weerspiegelt de Zero Trust SD-WAN-architectuur van Zscaler, waarbij de branch-appliance ruwe connectiviteit biedt terwijl de Zero Trust Exchange (hier: de SWG-keten van pop01) inspectie en beleidshandhaving verzorgt.

**IP-adressering:**

| Interface | Netwerk | Adres |
|-----------|---------|-------|
| eth0 (WAN) | Switch-WAN `192.168.122.0/24` | `192.168.122.33` (gw `192.168.122.1`) |
| eth1 (Site-LAN) | Switch-Site `172.16.10.0/24` | `172.16.10.1` |

Externe SSH-toegang is beschikbaar via een GNS3-host port-forward op `10.158.10.67:7033`.

---

## Configuratie

### NAT masquerade

Site-LAN-verkeer wordt via NAT door eth0 geleid zodat branch-apparaten het WAN kunnen bereiken:

```
set nat source rule 10 outbound-interface name eth0
set nat source rule 10 source address 172.16.10.0/24
set nat source rule 10 translation address masquerade
```

### Statische host-mapping voor NetBird

VyOS biedt een lokale DNS-override zodat branch-apparaten de NetBird-coördinatieserver kunnen resolven voordat de overlay actief is:

```
set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23
```

### DSCP / QoS (gepland)

DSCP EF-markering voor Microsoft Teams-verkeer was gepland om de QoS-laag van SD-WAN te demonstreren, maar is nog niet geïmplementeerd. Klassieke IPsec- en geavanceerde QoS SD-WAN-functies zijn uit scope gehaald ten gunste van de Zero Trust Branch-aanpak; zie [Beslissing: SD-WAN uit scope](../decisions/sdwan-descoped.md).

---

## Integratiepunten

- **Verkeerspad sitepc01:** sitepc01 → NAT via site01 → NetBird-overlay → pop01 (SWG-inspectie). Dit is hetzelfde pad dat mobile01 volgt — er bestaat geen branch-specifieke tunnel.
- **Site-LAN-segment:** `172.16.10.0/24`. sitepc01 zit op `172.16.10.50` met site01 als default gateway.
- **Geen directe WireGuard-tunnel** tussen site01 en pop01. Alle beveiligingshandhaving is gecentraliseerd op de PoP.
- **Identiteit sitepc01:** ingeschreven als `docent1` (Docenten-persona) — Entra ID joined en Intune enrolled (gedocumenteerd in V44).
- **NetBird-coördinatie:** sitepc01 gebruikt de [NetBird](netbird.md)-client om toe te treden tot de overlay, geauthenticeerd via Entra ID.

---

## Bekende problemen / valkuilen

### DNS-bootstrap voor inschrijving sitepc01

sitepc01 had een tijdelijke scaffold-NIC nodig (Ethernet 2 gericht op `8.8.8.8`) tijdens de initiële inschrijving, omdat [Unbound](ioc2rpz.md) op pop01 DNS-verzoeken van niet-overlay IP-adressen blokkeert via de WAN ACL. Zonder de scaffold-NIC kan de machine Entra ID- of NetBird-eindpunten niet resolven om de inschrijving te voltooien.

### ICMP misleidend onder ALLOW-only beleid

Ping naar overlay-peers faalt by design wanneer het firewallbeleidsmodel ALLOW-only is (geen ICMP-regel). Gebruik in plaats daarvan poortgebaseerde connectiviteitstests:

```powershell
Test-NetConnection -ComputerName <target> -Port 3128
```

### Caddy CA-certificaat kip-en-ei-probleem

Het interne CA-certificaat van Caddy moet op sitepc01 geïmporteerd worden voordat NetBird-inschrijving kan slagen, maar het certificaat kan niet via de overlay worden overgedragen omdat de overlay nog niet actief is. De workaround is het certificaat via de scaffold-NIC over te zetten voordat de overlay tot stand is gebracht.

### SD-WAN-functies uit scope

Klassieke SD-WAN-mogelijkheden zoals IPsec-tunnelling en QoS-markering zijn uit scope gehaald voor het project. De rationale is gedocumenteerd in [Beslissing: SD-WAN uit scope](../decisions/sdwan-descoped.md) en vervangen door het Zero Trust Branch-model beschreven in [Beslissing: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: GNS3](gns3.md)
- [Concept: SASE](../concepts/sase.md)
- [Beslissing: SD-WAN uit scope](../decisions/sdwan-descoped.md)
- [Beslissing: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)
- [Bevinding: DC-LAN-isolatie route ACL](../findings/dc-lan-isolation-route-acl.md)
