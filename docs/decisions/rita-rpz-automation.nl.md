---
title: "Beslissing: RITA-beacons als derde dynamische RPZ-feed"
tags: [decision, rita, rpz, ioc2rpz, dns, beaconing]
---

# Beslissing: RITA-beacons als derde dynamische RPZ-feed

**Status:** Geïmplementeerd (PoC-gevalideerd op de parallelle stack, sandbox-integratie pending)  
**Datum:** Mei 2026 (Verslag07, Deel II)

> **Scope.** Gebouwd en end-to-end bewezen op de **parallelle stack** (`mgmt01` = `192.168.122.20`,
> `pop01` = `192.168.122.11`). De pipeline voedt dezelfde ioc2rpz → BIND → Unbound keten die de
> sandbox gebruikt, maar als parallel-stack-experiment. De eigen RPZ van de sandbox blijft de
> source of truth.

## Context

De DNS threat-intelligence keten (zie [Runbook 06](../runbooks/06-dns-threat-intel.nl.md))
blokkeert domeinen die al in een gepubliceerde feed staan: URLhaus en ThreatFox. Beide zijn
statische IOC-lijsten. Ze zijn uitstekend voor bekend-kwaadaardige domeinen en nutteloos tegen
een C2-domein dat nog op niemands lijst staat.

RITA dicht dat gat vanaf de andere kant. Het scoort src→dst-paren op beaconing (regelmatige,
machine-achtige verbindingsintervallen) over Zeeks `conn.log`. Een hoge beacon score is
gedragsmatig bewijs van C2, los van enige reputatielijst. De vraag was hoe je die
gedragsdetectie omzet in een echte blokkering in plaats van een dashboard-entry waar een analist
handmatig op moet reageren.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Handmatige analist-review** | Menselijk oordeel op elke hit; geen false-positive-blokkering | Traag; een beacon blijft bereikbaar tot iemand kijkt; ondermijnt het punt van geautomatiseerde detectie |
| **RITA → file-feed → ioc2rpz (gekozen)** | Hergebruikt de bestaande ioc2rpz-aggregatie, TSIG-transfer en Unbound-enforcement; een gedetecteerd domein wordt binnen minuten NXDOMAIN; volledig onbeheerd | Gedragsmatige false positives (NetBird, NTP) hebben een whitelist nodig; RITA-scoring is gevoelig voor het dataset-window |
| **RITA → directe Unbound-zoneschrijf** | Eén hop minder | Omzeilt de feed-merging en serial-beheer van ioc2rpz; zou herimplementeren wat ioc2rpz al doet; verliest één audit-punt voor alle drie de feeds |

## Beslissing

Koppel RITA als **derde dynamische bron** naast URLhaus en ThreatFox, via een `file:`-source die
ioc2rpz al ondersteunt.

Een cron job (`/opt/scripts/rita_beacon_feed.sh`, elke 15 minuten) draait `rita view --beacon`,
houdt rijen met **beacon score ≥ 0.8**, strips een whitelist van bekend-goede domeinen, en
schrijft de overblijvers naar `/opt/ioc2rpz/cfg/rita_beacons.txt`. ioc2rpz leest dat bestand als
de `rita_beacons`-source, merget het in de `threat-intel.rpz.sase` zone naast de twee statische
feeds, verhoogt het zone-serial, en het bestaande TSIG-AXFR → BIND → Unbound-pad maakt van het
domein een authoritative NXDOMAIN.

Filteren op **score**, niet op RITA's **severity**, was bewust. RITA's severity-classificatie is
conservatief: `httpbin.org` met 333 connecties en score 0.629 kreeg slechts "Medium". De ruwe
score is de betrouwbaardere trigger, en 0.8 was de gekozen drempel.

## Gevolgen

- Het blokkeerpad is volledig geautomatiseerd: Zeek capteert → RITA scoort → cron extraheert →
  ioc2rpz merget → BIND transfereert → Unbound enforcet. Een fake beacon
  (`fake-c2-beacon-test.example.org`) is handmatig geplaatst en bevestigd tot NXDOMAIN door de
  hele keten; een live beacon (`grafana.com` vanuit de lokale monitoring-stack) is opgevangen,
  als Critical gescoord, en op dezelfde manier geblokkeerd.
- Een whitelist (`rita_whitelist.txt`) is verplicht en is het eerste controle-punt voor false
  positives. Zonder die zou de RPZ NetBird (100% beacon score) en Ubuntu NTP (99%) blokkeren,
  legitieme periodieke services. Zie de
  [RITA → RPZ integratie-runbook](../runbooks/15-rita-rpz-integration.nl.md).
- RITA-scoring hangt af van het analysevenster. Een rolling import verwerkt het meest recente
  uur-chunk, dus een domein dat alleen in een eerder venster beaconde kan laag scoren bij een
  latere import. Een beacon moet lang genoeg binnen het huidige venster lopen om betrouwbaar
  opgevangen te worden.
- De propagatie-latency wordt begrensd door de SOA refresh, verlaagd naar 300 s zodat Unbound de
  zone binnen vijf minuten opnieuw ophaalt. Of dat de behoefte aan een handmatige
  `rm zone + restart` op `pop01` volledig wegneemt, was ten tijde van het verslag nog niet
  end-to-end geverifieerd.
- Dit voegt een gedragsmatige feed toe aan een stack die verder reputatie-only was, zonder de
  enforcement-laag aan te raken. ioc2rpz, BIND en Unbound weten of merken niet dat de derde bron
  gedragsmatig is in plaats van een download.

## Gerelateerd

- [Component: RITA](../components/rita.nl.md)
- [Component: ioc2rpz + BIND + Unbound](../components/ioc2rpz.nl.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.nl.md)
- [Runbook: RITA → RPZ integratie](../runbooks/15-rita-rpz-integration.nl.md)
- [Runbook: DNS Threat Intelligence](../runbooks/06-dns-threat-intel.nl.md)
