---
title: "Beslissing: BIND als TSIG-tussenstap"
tags: [decision, dns, network, ioc2rpz]
---

# Beslissing: BIND als TSIG-tussenstap

**Status:** Geïmplementeerd  
**Datum:** April 2026 (Verslag25)

## Context

ioc2rpz authenticeert zone-transfers met TSIG (Transaction Signature, hmac-sha256). Unbound 1.24.2 ondersteunt het laden van RPZ-zones via zone-transfer (`rpz: primary:`-directive), maar ondersteunt geen TSIG-authenticatie voor die transfers. Dit is een gedocumenteerde ontbrekende functie in NLnetLabs/unbound GitHub issue #336, open sinds oktober 2020.

Zonder TSIG-ondersteuning in Unbound kan de zone-transfer van ioc2rpz niet worden geauthenticeerd.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Unbound direct (geen TSIG)** | Eenvoudigste keten | ioc2rpz vereist TSIG voor zone-transfer-autorisatie; zonder is AXFR geweigerd |
| **ioc2rpz configureren zonder TSIG** | Verwijdert de vereiste | Verwijdert transfer-authenticatie — elke DNS-client zou de RPZ-zone kunnen ophalen |
| **BIND als tussenstap** | BIND ondersteunt TSIG volledig; presenteert zone aan Unbound via loopback zonder auth | Extra component; voegt complexiteit toe; vereist os-bind-plugin op OPNsense |
| **Aangepaste forwarder/script** | Zou het gat kunnen overbruggen | Significante aangepaste code, kwetsbaar |

## Beslissing

BIND 9.20 invoegen (via OPNsense `os-bind`-plugin) als tussenstap tussen ioc2rpz en Unbound.

BIND verwerkt de TSIG-geauthenticeerde AXFR van ioc2rpz en fungeert als secundaire nameserver voor de RPZ-zone. Unbound transfert de zone dan van BIND via loopback (`127.0.0.1:53530`) zonder TSIG-authenticatie. De transfer is niet-geauthenticeerd maar beperkt tot localhost — acceptabel op een single-node-setup.

Dit is een protocol-gap-brug: BIND bestaat uitsluitend omdat Unbound 1.24.2 TSIG-ondersteuning mist. Als NLnetLabs/unbound issue #336 wordt opgelost in een toekomstige release, kan BIND worden verwijderd en kan Unbound direct van ioc2rpz halen.

## Gevolgen

- BIND (os-bind-plugin) moet worden geïnstalleerd en geconfigureerd op pop01 naast Unbound
- BIND luistert alleen op `127.0.0.1:53530` met `loopback_only` ACL's — het is niet blootgesteld als resolver
- ioc2rpz stuurt NOTIFY naar `192.168.122.13:53` (standaard DNS-poort, raakt Unbound, niet BIND op 53530). BIND ontdekt zone-updates alleen via SOA-verversing (3600 s). Handmatige retransfer: `rndc retransfer threat-intel.rpz.sase`
- TSIG-sleutel (`tkey_rpz_transfer`, hmac-sha256) moet expliciet worden gekoppeld aan de RPZ-zone in de ioc2rpz GUI — als leeg gelaten, veroorzaakt dit TSIG-fouten bij elke AXFR-poging

Zie ook: [Bevinding: Unbound geen TSIG](../findings/unbound-no-tsig.md)
