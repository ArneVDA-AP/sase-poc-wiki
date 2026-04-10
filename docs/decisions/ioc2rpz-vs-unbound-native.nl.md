---
title: "Beslissing: ioc2rpz vs Unbound native RPZ"
tags: [decision, ioc2rpz, dns, rpz, network]
---

# Beslissing: ioc2rpz vs Unbound native RPZ

**Status:** Geïmplementeerd  
**Datum:** April 2026 (Verslag24)

## Context

DNS-bedreigingsinformatie vereist een bron die feed-URL's (URLhaus, ThreatFox) aggregeert in een RPZ-zone en deze actueel houdt. Twee benaderingen: ioc2rpz gebruiken als een speciale feed-aggregator, of Unbound configureren om zonebestanden rechtstreeks van statische URL's te laden.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Unbound native RPZ** | Geen extra component; Unbound 1.24.2 ondersteunt native `rpz:`-blok | Unbound laadt zonebestanden alleen van schijf of via zone-transfer — geen ingebouwde HTTP-feed-ophaling; cron-job zou nodig zijn om feeds te downloaden en te converteren |
| **ioc2rpz** | Speciaal gebouwde feed-aggregator; HTTP-pull + RPZ-zone-compilatie; GUI voor beheer; NOTIFY naar downstream-resolvers | Extra Docker-container; ioc2rpz.gui heeft een JavaScript-loginbug |

## Beslissing

ioc2rpz als RPZ-zonebron, gedeployed als Docker-container op mgmt01.

ioc2rpz verwerkt de feed-levenscyclus: het haalt periodiek URLhaus- en ThreatFox-RPZ-feeds op via HTTP, voegt ze samen tot één zone (`threat-intel.rpz.sase`) en stuurt DNS NOTIFY naar downstream-servers wanneer de zone wordt bijgewerkt. Zonder ioc2rpz zouden cron-jobs nodig zijn om feeds op te halen, te converteren en zone-bestanden opnieuw te laden — het opnieuw implementeren van wat ioc2rpz al doet.

De GUI-JavaScript-bug is een bekend upstream-probleem met een eenvoudige sed-fix die na het starten van de container wordt toegepast. Zie [Bevinding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md).

## Gevolgen

- ioc2rpz Docker-container moet actief zijn op mgmt01 voor zone-updates. Initiële zone-lading blijft aanwezig in BIND/Unbound, ook als ioc2rpz tijdelijk uitvalt.
- ioc2rpz stuurt NOTIFY naar `192.168.122.13:53` (Unbound-poort, niet BIND-poort `53530`). BIND ontdekt updates alleen bij het volgende SOA-pollinterval (3600 s). Handmatige trigger: `rndc retransfer threat-intel.rpz.sase`. Productiefix: `pf rdr` om NOTIFY van `192.168.122.23` op poort 53 naar poort 53530 om te leiden.
- Het SSL-certificaat gebundeld in de ioc2rpz.gui Docker-image verliep in juli 2022. Vervang met een nieuw zelfondertekend certificaat vóór containerstart.
- ioc2rpz's GUI wordt geproxied door Caddy op `https://ioc2rpz.sandbox.local`.
