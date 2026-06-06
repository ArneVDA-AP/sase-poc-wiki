---
title: "Concept: behavioral analyse"
tags: [behavioral-analysis, beaconing, c2, zeek, rita, suricata, sase]
---

# Concept: behavioral analyse

**Definitie:** Dreigingen detecteren op basis van hoe verkeer zich *over tijd gedraagt* (regelmatige intervallen, abnormale duur, geautomatiseerde call-home patronen) in plaats van de *inhoud* van afzonderlijke packets te matchen tegen bekende kwaadaardige signatures.

## Hoe het hier van toepassing is

Deze stack draait twee detectiefilosofieën naast elkaar, en ze beantwoorden verschillende vragen.

**Signature-gebaseerde** detectie vraagt "heb ik dit exacte slechte ding al eerder gezien?" [Suricata](../components/suricata.md) matcht elk packet of elke flow tegen tienduizenden regels die bekende malware, exploits en C2-indicatoren beschrijven. Het is snel en precies, maar het is blind voor alles wat nog niet in een ruleset zit: een vers C2-domein, een custom implant, een onbekende beacon.

**Behavioral** detectie vraagt "*gedraagt* deze host zich gecompromitteerd, wat de payload ook is?" Dat is het werk van het [Zeek](../components/zeek.md) + [RITA](../components/rita.md) duo op de SASE_POC parallelle stack. De redenering is eenvoudig: een mens die surft is onregelmatig, maar malware die incheckt bij zijn controller is metronomisch. Je hoeft de malware niet te herkennen om het ritme op te merken.

De splitsing tussen de twee tools is bewust: **Zeek ziet, RITA beslist.** Zeek registreert passief elke flow in gestructureerde logs; RITA importeert die logs en scoort gedrag over het hele capture-venster. In dit pad wordt niets tegen een signature gematcht: het oordeel komt uit de *vorm* van de communicatie.

### De drie gedragingen die hier gedetecteerd worden

- **Beaconing:** periodiek, regelmatig call-home verkeer. RITA scoort elk `source → destination` paar op hoe regelmatig het interval en de payloadgrootte zijn en produceert een beacon score van 0 tot 1. Legitieme periodieke diensten valideren de methode: NetBird update-checks landden op 100% en Ubuntu NTP op 99% op een known-good netwerk, wat precies is wat een werkende detector zou moeten flaggen. Een echte C2-beacon zou in dezelfde lijst verschijnen, en de analist onderzoekt wat hij niet herkent.
- **C2-over-DNS:** command-and-control getunneld in DNS-queries. Dit is belangrijk omdat [DNS-niveau RPZ-blokkering](rpz.md) het op zichzelf niet kan stoppen: de kwaadaardige payload rijdt mee in de querynamen naar een domein dat misschien op geen enkele blocklist staat. Behavioral analyse spot het abnormale querypatroon dat signatures en domein-blocklists missen.
- **Lange verbindingen:** sessies die veel langer openstaan dan legitiem verkeer rechtvaardigt, een klassieke indicator van een interactief C2-kanaal.

Omdat dit behavioral is en niet signature, kan het een bestemming flaggen die geen enkele feed ooit heeft vermeld. Wanneer dat gebeurt (bijvoorbeeld RITA die `grafana.com` in testen als Critical beacon op score 0,998 scoort) kan de detectie in de [RITA→RPZ-pipeline](../decisions/rita-rpz-automation.md) worden geduwd zodat een behavioral ontdekt domein een DNS-blok (NXDOMAIN) wordt dat op clients wordt afgedwongen. Behavioral detectie voedt signature-achtige enforcement.

## Waar het in de stack voorkomt

**[Zeek](../components/zeek.md):** de sensor. Ontleedt passief flows op de core-, Site1- en DC-segmenten in `conn/dns/ssl/http/ntp` logs. Produceert de ruwe telemetrie; velt zelf geen oordeel.

**[RITA](../components/rita.md):** de analyser. Importeert Zeek-logs in ClickHouse en scoort ze op beaconing, C2-over-DNS, lange verbindingen en threat-intel matches. Hier wordt het behavioral oordeel geveld.

**[Suricata](../components/suricata.md):** het contrast. De signature-gebaseerde IDS van de stack op pop01. Behavioral analyse is additief aan Suricata, geen vervanging: Suricata vangt het bekende, behavioral analyse vangt het *abnormale*. Waar de Suricata van de sandbox en de Zeek/RITA van de parallelle stack hetzelfde terrein bestrijken, is de sandbox de bron van waarheid.

**[ioc2rpz / RPZ](../components/ioc2rpz.md):** de enforcement-uitgang. Een behavioral detectie met hoge zekerheid (beacon score ≥ 0,8) kan in de RPZ-feed worden geëxporteerd, waardoor een behavioral bevinding een DNS-niveau blok wordt. Zie [Beslissing: RITA→RPZ-automatisering](../decisions/rita-rpz-automation.md).

## Belangrijke onderscheidingen

**Behavioral vs. signature:** signature-detectie matcht *inhoud* tegen een bekende kwaadaardige lijst en is blind voor nieuwigheid; behavioral detectie matcht *patronen over tijd* en kan een bestemming flaggen die geen enkele feed heeft vermeld, ten koste van een analist die de intentie moet beoordelen. Ze zijn complementair, niet inwisselbaar. Suricata is signature; Zeek/RITA is behavioral.

**Detectie vs. enforcement:** behavioral analyse *detecteert* hier alleen. Zeek en RITA blokkeren niets. Enforcement gebeurt elders: een detectie wordt pas een blok wanneer ze in RPZ wordt gevoed, waar Unbound NXDOMAIN retourneert. Houd "wat RITA vond" gescheiden van "wat geblokkeerd raakte".

**Score vs. severity:** RITA geeft zowel een numerieke beacon score als een categorische severity. De score is het betrouwbare signaal voor automatisering; RITA's severity-labeling is conservatief (een duidelijk periodieke bestemming kreeg slechts "Medium"). De RITA→RPZ-extractie filtert om die reden op beacon score ≥ 0,8.

**Beaconing ≠ kwaadaardig op zichzelf:** de meeste beacons op een gezond netwerk zijn legitieme automatisering (updates, NTP, telemetrie). Het patroon is verdacht, niet het oordeel. Daarom gaat een whitelist vooraf aan elke geautomatiseerde blokkering van RITA-output.

## Bronnen

- `rayan_zeek-rita-documentation.md`: Zeek/RITA-implementatie, multi-worker cluster, beacon-validatieresultaten.
- `Verslag07_DNS_ThreatIntel_RITA_RPZ_v2.md`: RITA→RPZ-pipeline, beacon-score extractie, `grafana.com` end-to-end blok.
