---
title: "Beslissing: Zero Trust Branch in plaats van klassiek IPsec SD-WAN"
tags: [decision, sd-wan, vyos, netbird, zero-trust]
---

# Beslissing: Zero Trust Branch in plaats van klassiek IPsec SD-WAN

**Status:** Geïmplementeerd  
**Datum:** Mei-juni 2026 (Verslag41, Verslag43)

## Context

Traditioneel SD-WAN (IPsec site-to-site-tunnels, uCPE, QoS) maakte aanvankelijk deel uit van de architectuur. Herziening v3 (maart 2026) schrapte de klassieke IPsec/uCPE-aanpak (zie [Beslissing: SD-WAN geschrapt](sdwan-descoped.md)). De vraag bleef: wat vervangt het branch-connectiviteitsmodel? Simpelweg elk apparaat individueel inschrijven via NetBird is functioneel, maar adresseert geen QoS of failover — twee mogelijkheden die SD-WAN traditioneel biedt.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Klassiek IPsec-gebaseerd SD-WAN** | Traditionele aanpak; site-to-site-tunnel biedt transparante subnettoegang; goed begrepen QoS-modellen | Verleent impliciete subnetniveau-toegang op basis van netwerklocatie — in strijd met Zero Trust. Dupliceert de ZTNA-overlay. F12-F14 testkader meet het verkeerde paradigma |
| **Zero Trust Branch (NetBird-overlay + VyOS QoS)** | Per-apparaat authenticatie; geen impliciet subnetvertrouwen; sluit aan op Zscaler/Netskope ZT-SD-WAN-model; VyOS biedt QoS via DSCP zonder IPsec | VyOS QoS wordt toegepast op de site-gateway, niet end-to-end; failover-detectie is afhankelijk van NetBird relay-infrastructuur |

## Beslissing

Zero Trust Branch-model: VyOS site01 fungeert als SASE-gateway op Site-LAN (172.16.10.0/24), de NetBird-overlay dient als transportlaag, en VyOS past DSCP EF-markering toe voor QoS-classificatie via `tc` op eth0. Dit sluit aan op het Zscaler/Netskope Zero Trust SD-WAN-model waarbij vestigingen worden behandeld als niet-vertrouwde netwerken en elk apparaat individueel authenticeert.

## Bewijs

De ZT-Branch-implementatie is gevalideerd door twee specifieke tests:

- **Test #5 (QoS onder last, V43):** DSCP EF-verkeer: 300/300 pakketten, 0 drops. Bulkverkeer onder dezelfde last: 26 drops + 17.000+ overlimits. Demonstreert effectieve verkeerspriorisering zonder IPsec-tunnels.
- **Test #6 (Failover-detectie, V43):** CRITICAL-niveau failover-detectie in minder dan 30 seconden. Valideert dat de NetBird-overlay voldoende veerkracht biedt voor branch-connectiviteit.

Spoor-1-contract acceptatiecriteria B1-B4 zijn volledig gesloten op basis van deze resultaten.

## Gevolgen

- **F12-F14 (klassieke IPsec-tests) zijn N.v.t.** Deze tests maten een ander paradigma. De ZT-Branch tests (#5 en #6) zijn het correcte validatiekader voor de geïmplementeerde architectuur.
- **VyOS-rol is herdefinieerd:** VyOS site01 is een SASE-gateway met DSCP-markering, geen IPsec-router. De configuratie is `tc`-gebaseerde QoS + NAT, niet IPsec + BGP.
- **Per-apparaat authenticatie is verplicht:** Elk apparaat op Site-LAN moet een eigen NetBird-inschrijving hebben. Er is geen site-niveau vertrouwen — een gecompromitteerd apparaat op Site-LAN verleent geen toegang tot datacenterresources voor andere apparaten.
- **QoS is alleen site-egress:** DSCP-markering op VyOS is van toepassing op verkeer dat de site verlaat. End-to-end QoS is afhankelijk van tussenliggende netwerkhops die DSCP-markeringen respecteren, wat niet gegarandeerd is op het publieke internet.

Zie ook: [Beslissing: SD-WAN geschrapt](sdwan-descoped.md), [Component: VyOS](../components/vyos.md), [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)
