---
title: "Beslissing: SD-WAN-functies geschrapt (F12, F13, F14)"
tags: [sd-wan, ztna, architecture, decision]
---

# Beslissing: SD-WAN-functies geschrapt (F12, F13, F14)

**Status:** Geïmplementeerd  
**Datum:** maart 2026 (Herziening v3), bevestigd april 2026 (Research SD-WAN-sessie)

## Context

Het oorspronkelijke Handboek bevatte drie SD-WAN-acceptatietests:

- **F12** — IPsec-tunnelverbinding tussen site01 en het datacenter
- **F13** — QoS-verkeersclassificatie op VyOS
- **F14** — Datacentertoegang van sitegebruikers via de SD-WAN-tunnel

De implementatie hiervan zou vereisen: een IPsec-tunnel op VyOS, een NetBird-container op VyOS als uCPE (customer premises equipment) host, en VyOS QoS-configuratie. Alle drie werden uitgesteld en vervolgens expliciet geschrapt.

## Overwogen opties

| Optie | Pro | Contra |
|-------|-----|--------|
| Implementeer IPsec + NetBird uCPE op VyOS | Voldoet aan originele Handboek-specificatie | Repliceert netwerkcentrische patronen die SASE juist vervangt; site-to-site-tunnel verleent impliciete subnettoegang, in strijd met Zero Trust; dupliceert de ZTNA-overlay |
| Schrap SD-WAN-tests; schrijf sitepc01 individueel in bij NetBird | Sluit aan op Zero Trust-architectuur; consistent met Zscaler-model; eenvoudiger | F12/F13/F14 moeten als N.v.t. worden gemarkeerd in alle testmatrices |
| Implementeer alleen QoS (zonder IPsec) | Gedeeltelijke naleving | QoS-configuratie op VyOS zonder bijbehorende IPsec-tunnel is contextloos en niet zinvol voor de PoC |

## Beslissing

SD-WAN-tests F12, F13 en F14 zijn geschrapt. VyOS blijft in de topologie als de site01 WAN-gateway en NAT-apparaat voor Site-LAN, maar is niet geconfigureerd als IPsec-router of uCPE NetBird-host.

Sitegebruikers (sitepc01) zullen het datacenter bereiken via individuele NetBird-inschrijving — hetzelfde mechanisme als mobile01 — in plaats van via een site-to-site-tunnel.

## Gevolgen

- **F12, F13, F14** zijn als N.v.t. gemarkeerd in alle acceptatietestmatrices. Dit is een expliciete architectuurbeslissing, geen implementatieleemte.
- **F15-stappen 7–8** zijn N.v.t. om dezelfde reden (stap 7 vereist sitepc01 met NetBird; stap 8 vereist QoS op VyOS).
- **VyOS** blijft in de topologie, maar de configuratie is minimaal — alleen WAN-connectiviteit en NAT. Zie [VyOS](../components/vyos.md).
- **sitepc01** datacentertoegang is gepland via NetBird-inschrijving (nog niet uitgevoerd per april 2026).
- De architectuur sluit direct aan op het "Zero Trust SD-WAN"-model van Zscaler: vestigingen behandeld als niet-vertrouwde netwerken (zoals cafés), elk apparaat authenticeert individueel, geen site-to-site-tunnels.

## Onderbouwing

**IPsec repliceert wat ZTNA vervangt.** Een site-to-site IPsec-tunnel is netwerkcentrisch: hij verleent alle hosts op Site-LAN impliciete toegang tot het datacenter op basis van netwerklocatie. Dit is precies het perimeter-model dat SASE is ontworpen te vervangen. NetBird biedt al een Zero Trust-overlay waarbij elke peer individueel authenticeert en ACL-beleidsregels per-peer toegang afdwingen.

**Zscaler-afstemming.** Het schrappen sluit de architectuur af op de publiek gedocumenteerde Zero Trust SD-WAN-aanpak van Zscaler: geen impliciet vertrouwen op basis van netwerklocatie, elke vestiging behandeld als een niet-vertrouwd netwerk, cloudinspectie voor al het verkeer.

**Complexiteitsreductie.** Het verwijderen van IPsec, uCPE en QoS vermindert het implementatieoppervlak en houdt de PoC gericht op de SASE-beveiligingscomponenten.

## Gerelateerd

- [VyOS](../components/vyos.md)
- [NetBird](../components/netbird.md)
- [Zero Trust](../concepts/zero-trust.md)
- [SASE](../concepts/sase.md)
- [Testen: Acceptatietests](../testing/acceptance-tests.md)
