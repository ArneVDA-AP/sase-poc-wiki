---
title: "VyOS: SASE-gateway (site01)"
tags: [network, sase, sd-wan, nat, zero-trust]
---

# VyOS: SASE-gateway (site01)

**Rol:** SASE-gateway voor de remote site die WAN-connectiviteit en NAT biedt voor het Site-LAN, invulling gevend aan de SD-WAN-pijler via een Zero Trust Branch-model. Equivalent aan een Zscaler Zero Trust Branch edge-appliance.  
**Versie:** VyOS rolling 2026.02.16  
**Status:** ✅ Operationeel (site01 op `192.168.122.33`, sitepc01 (Tiny11) ge-enrolld in sandbox)  
**Configuratielocatie:** `/etc/vyos/config.boot` op site01

---

## Hoe het werkt in deze stack

VyOS draait als `site01` met twee interfaces: eth0 op het WAN-segment (`192.168.122.33`) en eth1 op het Site-LAN (`172.16.10.1/24`). De verantwoordelijkheden zijn bewust beperkt tot connectiviteit: het fungeert als NAT-gateway zodat apparaten op het Site-LAN het internet kunnen bereiken, maar het handhaaft zelf geen beveiligingsbeleid.

De branch volgt een **Zero Trust Branch-model**: de gehele site wordt behandeld als een niet-vertrouwde locatie, niet anders dan een Wi-Fi-netwerk in een koffiebar. Er zijn geen IPsec-tunnels tussen site01 en pop01. In plaats daarvan draait sitepc01 een [NetBird](netbird.md)-client en treedt toe tot de WireGuard-overlay met een identiteitsgebaseerde verbinding, en volgt exact hetzelfde pad als elke mobiele gebruiker. Alle beveiligingsinspectie vindt plaats op de PoP, niet aan de branch-rand.

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

### QoS-shaper (DSCP-gebaseerd)

VyOS past op eth0 egress een QoS-shaper toe die Microsoft Teams-media classificeert op DSCP-markering (de QoS-laag van SD-WAN binnen het project). De DSCP-waarden worden op het endpoint gezet (Intune NetworkQoSPolicy CSP / GPO); VyOS ziet ze in cleartext op het **split-tunnel-pad**, waar Teams Optimize-media rechtstreeks naar eth1 wordt gerouteerd (via een Intune-`/14`-route) in plaats van door de WireGuard-overlay. Al het overige verkeer blijft binnen de versleutelde overlay, waar app-bewuste QoS niet mogelijk is.

Op VyOS rolling 2026.02.16 gebruikt de shaper de `qos policy shaper`-commandostructuur (niet de oudere `traffic-policy`):

```
set qos policy shaper SDWAN-WAN bandwidth '20mbit'
set qos policy shaper SDWAN-WAN default bandwidth '50%'
set qos policy shaper SDWAN-WAN default queue-type 'fq-codel'

# class 10 — Teams audio (DSCP EF, strict priority)
set qos policy shaper SDWAN-WAN class 10 bandwidth '10%'
set qos policy shaper SDWAN-WAN class 10 ceiling '30%'
set qos policy shaper SDWAN-WAN class 10 priority '0'
set qos policy shaper SDWAN-WAN class 10 match TEAMS-AUDIO ip dscp 'EF'

# class 15 — Teams video (DSCP AF41)
set qos policy shaper SDWAN-WAN class 15 priority '1'
set qos policy shaper SDWAN-WAN class 15 match TEAMS-VIDEO ip dscp 'AF41'

# class 18 — Teams screen-sharing (DSCP AF21)
set qos policy shaper SDWAN-WAN class 18 priority '2'
set qos policy shaper SDWAN-WAN class 18 match TEAMS-SHARING ip dscp 'AF21'

set qos interface eth0 egress 'SDWAN-WAN'
```

De 20 Mbit root-bandbreedte is een bewust demo-plafond: het libvirt-pad heeft geen echte WAN-bottleneck, dus een laag plafond is nodig om contention te fabriceren voor de onder-last-test. Productie aligneert dit op de werkelijke WAN-capaciteit.

**Verificatie:** het operationele commando `show qos interface eth0` bestaat niet op deze rolling-build. Gebruik in plaats daarvan de onderliggende Linux-`tc`:

```
sudo tc -s class show dev eth0
```

**Bewezen (V43):** een synthetische DSCP-EF-stroom markeerde 300/300 pakketten in class 10 met 0 drops (contract B4). Onder last (Test #5) hield de EF-class 0 drops / 0 overlimits terwijl de bulk-default-class 26 drops + ~17.000 overlimits opliep. Real-time-media wordt beschermd onder congestie. Zie [Beslissing: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

> **Build-opmerking:** op deze rolling-build mapt een percentage-`ceiling` verkeerd naar `tc` (`ceiling '30%'` wordt `ceil 3Gbit`, effectief onbegrensd). Classificatie en prioriteit werken correct; alleen ceiling-handhaving is getroffen. Dit is benign voor de PoC-demo.

### WAN-link-monitoring

Een health-check-script (`/config/scripts/wan-health-check.sh`) draait op een VyOS-task-scheduler-interval van 30 seconden en pingt pop01 (`192.168.122.13`), de internet-gateway (`192.168.122.1`) en `8.8.8.8`. Bij falen logt het een `CRITICAL`-regel naar `/var/log/sdwan-health.log`.

**Bewezen (V43, Test #6):** het neerhalen van pop01's interface produceerde een `CRITICAL`-detectie binnen 30 seconden, met herstel gelogd zodra de interface terugkwam. Dit is eerlijke single-WAN-framing: detectie en alerting, geen automatische dual-WAN-switch (het lab heeft één WAN). Productie zou multi-PoP-failover, BFD/SNMP en fail-closed-handhaving toevoegen.

---

## Integratiepunten

- **Verkeerspad sitepc01:** sitepc01 → NAT via site01 → NetBird-overlay → pop01 (SWG-inspectie) voor Default/Allow-verkeer. Dit is hetzelfde pad dat mobile01 volgt; er bestaat geen branch-specifieke tunnel. Teams Optimize-media neemt in plaats daarvan het split-tunnel-pad rechtstreeks via eth0 (zie QoS hierboven).
- **Site-LAN-segment:** `172.16.10.0/24`. sitepc01 zit op `172.16.10.10` met site01 als default gateway. Het draagt ook een tijdelijke tweede NIC op het WAN-segment (`192.168.122.50`), gebruikt als setup-/RDP-steiger (te verwijderen zodra overlay-RDP bewezen is).
- **Geen directe WireGuard-tunnel** tussen site01 en pop01. Alle beveiligingshandhaving is gecentraliseerd op de PoP.
- **Identiteit sitepc01 (twee onafhankelijke lagen):** ge-enrolld in NetBird als `docent1` (Docenten-persona); afzonderlijk Entra ID joined en Intune enrolled via `student1`, het enige Intune-gelicenseerde sandbox-account (`docent1`/`admin1` zijn bewust ongelicenseerd). Gedocumenteerd in V44.
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

### Klassieke IPsec-SD-WAN uit scope (QoS en failover zijn geïmplementeerd)

Enkel de *klassieke* IPsec/uCPE-SD-WAN-aanpak is uit scope gehaald: de oorspronkelijke acceptatietests F12–F14 (IPsec-tunnel, site-to-site datacentertoegang) zijn als N/A gemarkeerd. De SD-WAN-QoS- en failover-detectie-mogelijkheden zijn **geïmplementeerd** onder het Zero Trust Branch-model (de DSCP-shaper en WAN-health-check hierboven, bewezen in V43 Test #5 en Test #6). De rationale is gedocumenteerd in [Beslissing: SD-WAN uit scope](../decisions/sdwan-descoped.md) en [Beslissing: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: GNS3](gns3.md)
- [Concept: SASE](../concepts/sase.md)
- [Beslissing: SD-WAN uit scope](../decisions/sdwan-descoped.md)
- [Beslissing: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)
- [Bevinding: DC-LAN-isolatie route ACL](../findings/dc-lan-isolation-route-acl.md)
