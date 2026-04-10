---
title: "Bevinding: NetBird primaire naamserver vereist voor externe RPZ-dekking"
tags: [finding, network, ioc2rpz, workaround]
---

# Bevinding: NetBird primaire naamserver vereist voor externe RPZ-dekking

**Component:** [ioc2rpz](../components/ioc2rpz.md), [NetBird](../components/netbird.md)  
**Ernst:** Blokker (voor DNS-bedreigingsinformatie)

## Wat er gebeurde

Unbound RPZ werkte bevestigd voor `*.sandbox.local`-interne domeinen — `testentry.rpz.urlhaus.abuse.ch` retourneerde correct NXDOMAIN wanneer rechtstreeks opgevraagd vanaf pop01. Echter, vanuit mobile01 via NetBird werden externe domeinen van bekende kwaadaardige domeinen niet geblokkeerd. `nslookup testentry.rpz.urlhaus.abuse.ch` op mobile01 retourneerde het echte (niet-geblokkeerde) IP.

## Oorzaak

De Custom DNS Zone van NetBird (`sandbox.local → pop01`) routeert alleen `*.sandbox.local`-query's via Unbound. Voor alle andere domeinen gebruikte mobile01 de standaard-DNS van de lokale netwerkadapter (doorgaans de ISP-resolver of router). Die query's bereikten pop01 Unbound nooit, dus de RPZ evalueerde ze nooit.

## Oplossing / workaround

In NetBird Dashboard → DNS: configureer een primaire naamserver:

```
Primaire naamserver: pop01 (100.70.154.79)
Overeenkomende domeinen: (leeg)
```

Het lege veld voor overeenkomende domeinen is cruciaal — het zorgt ervoor dat NetBird *alle* DNS-query's van de client via pop01 Unbound routeert, niet alleen query's voor specifieke domeinen. Met deze instelling zijn externe query's ook onderworpen aan Unbound RPZ-evaluatie.

Na toepassen: bevestigd via `nslookup testentry.rpz.urlhaus.abuse.ch` op mobile01 dat `Non-existent domain` retourneert en het Unbound-log dat `rpz: applied [ioc2rpz-threat-intel]` toont.

## Lessen

- NetBird Custom DNS Zones beïnvloeden alleen query's die overeenkomen met het opgegeven domeinachtervoegsel
- Om RPZ te beschermen tegen externe bedreigingen, moet al het DNS-verkeer (niet alleen intern) verlopen via de afhandelende resolver
- De instelling Primaire naamserver met lege overeenkomende domeinen is het NetBird-mechanisme hiervoor — het vervangt de adapter-DNS van de client voor alle query's zolang de tunnel actief is
- Dit is architectureel equivalent aan het dwingen van alle DNS via een bedrijfsresolver in een traditionele VPN-opstelling
