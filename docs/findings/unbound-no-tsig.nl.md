---
title: "Bevinding: Unbound 1.24.2 ondersteunt geen TSIG voor zonetransfers"
tags: [finding, network, ioc2rpz]
---

# Bevinding: Unbound 1.24.2 ondersteunt geen TSIG voor zonetransfers

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Ernst:** Blokker

## Wat er gebeurde

Een poging om Unbound te configureren voor het rechtstreeks overdragen van de RPZ-zone van ioc2rpz met TSIG-authenticatie mislukte. Het `rpz:`-blok van Unbound accepteert een `primary:`-adres maar heeft geen optie voor het opgeven van een TSIG-sleutel.

ioc2rpz vereist TSIG-authenticatie voor autorisatie van zonetransfer. Zonder dit worden AXFR-verzoeken door ioc2rpz geweigerd.

## Oorzaak

Unbound 1.24.2 implementeert TSIG niet voor inkomende zonetransfers (`rpz: primary:`-configuratie). Dit is een gedocumenteerde ontbrekende functie in NLnetLabs/unbound GitHub issue #336, open sinds oktober 2020.

## Oplossing / workaround

BIND 9.20 (OPNsense `os-bind`-plugin) invoegen als tussenpersoon. BIND verwerkt de TSIG-geauthenticeerde AXFR van ioc2rpz en presenteert de zone aan Unbound via loopback op `127.0.0.1:53530` zonder authenticatie.

Zie [Beslissing: BIND als TSIG-tussenpersoon](../decisions/bind-tsig-intermediary.md).

## Lessen

- Controleer altijd de huidige functieondersteuning in release notes vóór het ontwerpen van een transferketen — ontbrekende functies in langlopende open issues zijn mogelijk niet opgelost binnen uw tijdlijn
- BIND is een goed ondersteund secundair zonemechanisme; het gebruiken als TSIG-capable tussenpersoon is een schone workaround met minimale overhead
- NLnetLabs/unbound issue #336 is de gezaghebbende referentie als deze beperking toekomstige projecten beïnvloedt
