---
title: "Concept: RPZ — DNS Response Policy Zones"
tags: [ioc2rpz, dns, rpz, network, sase]
---

# Concept: RPZ — DNS Response Policy Zones

**Definitie:** Een uitbreiding op de DNS-autoritaire zone-serving die een resolver in staat stelt synthetische antwoorden (NXDOMAIN, NODATA, een walled garden-IP) te retourneren voor overeenkomende querynamen, waardoor DNS-niveau blokkering van bekende kwaadaardige domeinen mogelijk is voordat een netwerkverbinding wordt opgezet.

## Hoe het hier van toepassing is

RPZ onderschept op het vroegst mogelijke punt in de bedreigingslevenscyclus: naamresolutie. Als een BYOD-client de hostnaam van een bekende C2-server probeert op te lossen, retourneert Unbound NXDOMAIN — de TCP-verbinding start nooit, ongeacht poort of protocol.

De RPZ-zone in deze stack bevat ~71 767 records uit twee Abuse.ch-feeds (URLhaus + ThreatFox), bijgewerkt door ioc2rpz uit live threat intelligence-feeds. Wanneer een client een hostnaam opvraagt die overeenkomt met een RPZ-record, vervangt Unbound de geconfigureerde actie (NXDOMAIN) in plaats van het echte antwoord te retourneren.

**Zone-transferketen:**

```
URLhaus + ThreatFox (Abuse.ch-feeds)
    ↓ HTTP-ophalen door ioc2rpz
ioc2rpz → BIND 9.20 (TSIG AXFR) → Unbound (RPZ, respip-module)
                                        ↓
                             Alle client DNS-queries
```

Unbound laadt de RPZ-zone in het geheugen. De prestatieimpact per query is verwaarloosbaar — de zone is al aanwezig.

## Waar het in de stack voorkomt

**[ioc2rpz](../components/ioc2rpz.md)** — de zonebron. Aggregeert URLhaus- en ThreatFox-feeds in één RPZ-zone (`threat-intel.rpz.sase`). Fungeert als een autoritatieve DNS-server waarvan BIND overneemt.

**[NetBird](../components/netbird.md)** — de primaire nameserver-configuratie routeert alle DNS-queries van BYOD-clients via pop01 Unbound. Zonder dit omzeilen externe queries Unbound volledig, waardoor RPZ-bescherming voor niet-interne domeinen teniet wordt gedaan.

**[Suricata](../components/suricata.md)** — aanvullende controle. Suricata detecteert verbindingen naar bekende kwaadaardige IP's (van Abuse.ch SSL IP Blacklist en vergelijkbaar); RPZ voorkomt DNS-resolutie van bekende kwaadaardige domeinen. Beide gebruiken Abuse.ch als bron; ze bestrijken verschillende bedreigingsoppervlakken (verbinding vs. naamresolutie).

## Belangrijke onderscheidingen

**RPZ-actie = NXDOMAIN + autoritatieve vlag** — het RPZ-antwoord van Unbound retourneert NXDOMAIN met de `aa`-vlag (autoritatief antwoord) ingesteld. Dit onderscheidt een lokaal RPZ-getriggerde NXDOMAIN van een echte NXDOMAIN van het internet. De `aa`-vlag in een testantwoord (`drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch`) bevestigt dat de RPZ actief is, niet dat het domein daadwerkelijk niet bestaat.

**RPZ blokkeert geen IP-gebaseerde bedreigingen** — een client die DNS omzeilt (gebruikt een hard-gecodeerd IP) wordt niet beschermd door RPZ. Suricata op vtnet0 dekt dit geval via IP-gebaseerde signatuurregels.

**`respip`-modulepositie in Unbound** — RPZ-handhaving vereist de `respip`-module in Unbound's module-config. Het moet als eerste worden vermeld: `"respip python validator iterator"`. De `python`-module moet blijven (OPNsense gebruikt het voor de ingebouwde DNSBL-functie). Het verwijderen van een van beide breekt ofwel RPZ of de eigen blokkeerlijst van OPNsense.

## Bronnen

- `raw/Doc4_DNS_Threat_Intelligence.md` (volledige RPZ-architectuur, ioc2rpz + BIND + Unbound)
- `raw/Verslag24.md`, `raw/Verslag25.md`, `raw/Verslag26.md` (implementatiesessies)
