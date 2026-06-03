---
title: "Beslissing: IDS-correlatiebranch verwijderd uit Control Daemon"
tags: [decision, control-daemon, ids, nats]
---

# Beslissing: IDS-correlatiebranch verwijderd uit Control Daemon

**Status:** Geïmplementeerd  
**Datum:** Mei 2026 (Verslag35)

## Context

Het threat scoring-model van de Control Daemon bevatte aanvankelijk een IDS-correlatiebranch ontworpen om te reageren op C2 (Command and Control) beacon-detectie. Wanneer Suricata een IDS-alert afvuurde die overeenkwam met een C2-signatuur, zou de daemon het alert correleren met gebruikersidentiteit en quarantaine triggeren. Deze branch werd toegevoegd naast de malware-branch (die reageert op ClamAV/ICAP malware-detecties) en de DLP-branch (die reageert op data-exfiltratiepatronen).

Tijdens de implementatie onthulde analyse dat de IDS-correlatiebranch de quarantainemogelijkheid dupliceerde die al door de malware-branch werd geboden, maar met slechtere attributie. De malware-branch mapt overlay IP-adressen native naar peer-identiteit via de API van NetBird, wat schone gebruikersattributie biedt. De IDS-correlatiebranch zou dezelfde mapping moeten uitvoeren maar voegde geen unieke handhavingsactie toe — beide branches resulteren in dezelfde quarantaine-uitkomst.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **IDS-correlatiebranch behouden** | Voegt expliciete C2 beacon-responsmogelijkheid toe; apart scoringsgewicht voor IDS-events | Dupliceert quarantainemogelijkheid van de malware-branch met slechtere attributie. C2 beacon-detectie hoort meer thuis bij Zeek/RITA (gedragsanalyse over tijd), niet bij op maat gemaakte single-event correlatie |
| **IDS-correlatiebranch verwijderen** | Eenvoudiger scoringsmodel; malware-branch dekt al real-time quarantaine met native attributie; C2-respons uitgesteld naar geschikt tooling (Zeek/RITA) | IDS-events worden log-only in de daemon — geen geautomatiseerde respons op IDS-alerts |

## Beslissing

De IDS-correlatiebranch is verwijderd uit de Control Daemon. IDS-events van Suricata worden nog steeds opgenomen via NATS JetStream maar dragen alleen bij aan logging, niet aan de threat score of geautomatiseerde quarantaineacties.

De malware-branch dekt dezelfde real-time quarantainemogelijkheid met native attributie: wanneer ClamAV malware detecteert in de ICAP-pipeline, resolvet de daemon het overlay IP naar een NetBird-peeridentiteit en voert quarantaine uit. C2 beacon-respons wordt uitgesteld naar Zeek/RITA, dat gedragsanalyse over tijd uitvoert in plaats van single-event correlatie.

Daarnaast zijn `proxy_block`-events verwijderd uit het scoringsmodel. Omgevingsruis van het besturingssysteem (Windows-telemetrie, updatecontroles, certificaatintrekkingsopzoekingen) veroorzaakte frequente proxy-blocks die threat scores opbliezen zonder werkelijke dreigingen aan te duiden, wat leidde tot vals-positieve quarantainetriggers.

## Gevolgen

- IDS-alerts van Suricata zijn log-only in de Control Daemon. Ze verschijnen in NATS en in daemon-logs maar beïnvloeden de threat score niet en triggeren geen quarantaine.
- Het threat scoring-model is vereenvoudigd tot twee actieve branches: malware (ClamAV-detecties via ICAP) en DLP (data-exfiltratiepatronen).
- `proxy_block`-events zijn uitgesloten van scoring. Alleen expliciete malware-detecties en DLP-schendingen dragen bij aan de threat score.
- C2 beacon-detectie en -respons blijft een open capability gap. Als Zeek/RITA in de toekomst aan de architectuur wordt toegevoegd, kan C2-respons via dat pad worden herintroduceerd in plaats van via de Control Daemon.
- Het vals-positief percentage door omgevingsruis van het OS is geëlimineerd, waardoor quarantaineacties betrouwbaardere indicatoren zijn van werkelijke dreigingen.

Zie ook: [Component: Control Daemon](../components/control-daemon.md), [Component: Suricata](../components/suricata.md), [Component: NATS](../components/nats-jetstream.md), [Beslissing: Drielaags CASB-architectuur](casb-three-layers.md)
