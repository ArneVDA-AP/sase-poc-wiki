---
title: "RITA: behavioral threat analytics"
tags: [rita, zeek, behavioral-analysis, beaconing, c2, dns, rpz, docker, sase]
---

# RITA: behavioral threat analytics

**Rol:** Analyseert [Zeek](zeek.md)-logs op behavioral indicatoren die een signature-engine niet ziet (beaconing, C2-over-DNS, lange verbindingen, threat-intel matches) en brengt verdachte domeinen naar boven die in de [RITA→RPZ-pipeline](../decisions/rita-rpz-automation.md) kunnen worden geduwd voor DNS-niveau blokkering.  
**Versie:** RITA v5.1.1 (`ghcr.io/activecm/rita`), ClickHouse 24.1.6 backend  
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending  
**Configuratielocatie:** `/etc/rita/config.hjson` (scoring, filtering, threat intel), `/opt/rita/.env`, `/opt/rita/docker-compose.yml` op mgmt01 (`192.168.122.20`)

> **Sidenote, parallelle-stack scope.** RITA draait op de **SASE_POC**-topologie van Rayan, op mgmt01 (`192.168.122.20`). Het is daar end-to-end gevalideerd maar **niet in het sandbox-geheel geïntegreerd**. De bevindingen bereiken de sandbox alleen via het RITA→RPZ feed-bestand (zie Integratiepunten), niet via NATS of de [Control Daemon](control-daemon.md). RITA is de behavioral tegenhanger van de signature-gebaseerde [Suricata IDS](suricata.md) van de sandbox. Waar de twee overlappen, is de sandbox de bron van waarheid.

---

## Hoe het werkt in deze stack

RITA (Real Intelligence Threat Analytics) leest de logs die [Zeek](zeek.md) produceert en zoekt naar *patronen over tijd* in plaats van naar bekende kwaadaardige inhoud in één packet. Zeek ziet, RITA beslist. Het importeert Zeek's `conn`, `dns`, `ssl`, `http` en `ntp` logs in een **ClickHouse** column store (die MongoDB uit RITA v4 verving) en draait daar de analyse.

De kernvraag die RITA beantwoordt is "praat deze interne host met die bestemming op een manier die geautomatiseerd lijkt in plaats van menselijk?" Een gebruiker die surft is onregelmatig; malware die incheckt bij zijn C2 is metronomisch. RITA scoort elk `source → destination(FQDN/SNI)` paar op hoe regelmatig het interval en de payloadgrootte zijn en produceert een **beacon score** van 0 tot 1.

### Wat RITA detecteert

- **Beaconing:** periodiek, regelmatig call-home gedrag. De primaire use case hier.
- **C2-over-DNS:** command-and-control getunneld door DNS-queries (subdomein-enumeratiepatronen), wat de [DNS-niveau RPZ-blokkering](../concepts/rpz.md) op zichzelf niet kan vangen omdat de queries zelf de payload dragen.
- **Lange verbindingen:** sessies die veel langer openstaan dan normaal, een klassiek C2-signaal.
- **Threat-intel matches:** bestemmingen die op een feed (Feodo tracker) staan die in RITA is geladen.

### Importmodel

Zeek roteert logs elk uur naar `/opt/zeek/logs/YYYY-MM-DD/`. RITA importeert die in een benoemde database. De operationele dataset is een **rolling** import (`--rolling`), die first-seen scoring, prevalence tracking en temporele analyse over het hele venster mogelijk maakt in plaats van één momentopname:

```bash
sudo rita import -l /opt/zeek/logs/ --database sase_poc --rolling
```

De initiële import van 12 dagen (7–21 april 2026) verwerkte in ongeveer 7,5 minuten. Databasenamen mogen geen koppeltekens bevatten in RITA v5: gebruik `sase_poc`, niet `sase-poc`.

### De analyse lezen

```bash
sudo rita view sase_poc --beacon           # beaconing (primair)
sudo rita view sase_poc --long-connection  # langlopende sessies
sudo rita view sase_poc --c2-over-dns      # DNS-getunnelde C2
sudo rita view sase_poc --threat-intel     # feed-matches
```

### Validatie: legitieme beacons bewijzen de pipeline

De eerste validatierun flagde alleen legitieme periodieke diensten, wat net het bewijs is dat de detectie werkt: alles wat regelmatig is duikt op, en de analist onderzoekt wat hij niet herkent.

| Source | Destination | Beacon score | Interpretatie |
|--------|-------------|--------------|---------------|
| `192.168.122.20` | pkgs.netbird.io | 100% | NetBird update-check, perfect regelmatig |
| `192.168.122.20` | 185.125.190.56 | 99% | Ubuntu/Canonical NTP-sync |
| `192.168.122.20` | 8.8.8.8 | 73% | Google DNS, periodiek |

In een gecompromitteerde omgeving zou een kwaadaardige C2-beacon in deze zelfde lijst verschijnen, naast de goedaardige entries. Dat die entries op een known-good netwerk rond 100% landen, bevestigt dat de scoring klopt.

### Het `grafana.com`-voorbeeld: detectie tot blokkering

De end-to-end waarde van RITA komt tot uiting wanneer een detectie de [RITA→RPZ-pipeline](../decisions/rita-rpz-automation.md) voedt. In de threat-intel sessie van mei scoorde RITA `grafana.com` als **Critical (beacon score ≈ 0,998)** op basis van een gesimuleerde periodieke verbinding. Een extractiescript lichtte die FQDN uit de `rita view --beacon` output naar een feed-bestand, ioc2rpz aggregeerde het in de `threat-intel.rpz.sase` zone, en Unbound begon er **NXDOMAIN** op te antwoorden. De behavioral detectie op één stack werd een DNS-blok dat op clients werd afgedwongen:

```
nslookup grafana.com → Non-existent domain   (RITA-gedetecteerde beacon, score 0,998)
```

Dit is de pipeline waarvoor RITA bestaat. Zie de integratienotitie hieronder en [Beslissing: RITA→RPZ-automatisering](../decisions/rita-rpz-automation.md).

---

## Configuratie

De backend bestaat uit drie Docker-containers (RITA, ClickHouse, syslog-ng), gestart met `docker compose up -d` vanuit `/opt/rita`, persistent via `restart: unless-stopped`. Wacht op de ClickHouse-healthcheck voor je importeert:

```bash
docker inspect rita-clickhouse --format='{{.State.Health.Status}}'   # verwacht: healthy
```

Het analysegedrag staat in `/etc/rita/config.hjson`: beacon-scoringsdrempels, verbindingsfiltering en de threat-intel feedlijst. Twee databases bestaan uit de validatie: `sase_poc` (rolling, primair) en `sase_test` (single-day statisch, gebruikt tijdens bring-up).

**Score, niet severity, is het betrouwbare signaal voor automatisering.** RITA's eigen severity-label is conservatief: een bestemming met honderden verbindingen kreeg in testen slechts "Medium". De RITA→RPZ-extractie filtert daarom op **beacon score ≥ 0,8**, niet op het severity-veld. Zie [Beslissing: RITA→RPZ-automatisering](../decisions/rita-rpz-automation.md).

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Zeek](zeek.md) | ← (consumeert) | RITA importeert Zeek's `conn/dns/ssl/http/ntp` logs uit `/opt/zeek/logs/` in ClickHouse. Geen Zeek, geen RITA-input. |
| [ioc2rpz / RPZ](ioc2rpz.md) | → (feed-bestand) | Een cron-extractiescript schrijft FQDN's met hoge score naar een `rita_beacons.txt` feed die ioc2rpz als bron leest, waardoor een behavioral detectie een DNS-NXDOMAIN wordt. Dit is vandaag het enige pad waarlangs RITA-bevindingen de sandbox bereiken. Zie [Beslissing: RITA→RPZ-automatisering](../decisions/rita-rpz-automation.md) en [Runbook: RITA→RPZ-integratie](../runbooks/15-rita-rpz-integration.md). |
| [Suricata](suricata.md) | aanvullend | Suricata beantwoordt "is dit packet bekend-kwaadaardig?"; RITA beantwoordt "gedraagt deze flow zich als C2?". Behavioral vs. signature, op hetzelfde verkeer. |

---

## Bekende problemen / valkuilen

**Rolling import evalueert het venster, maar neemt alleen de laatste chunk op.** Een `rita import --rolling` voor de huidige dag verwerkt de meest recente uur-chunk; een domein dat alleen in een eerdere chunk beaconde scoort laag tot die chunk in het venster zit. Importeer meerdere uur-chunks voor een volledig beeld, of draai een beacon-test die lang genoeg loopt om binnen het actieve analysevenster te vallen.

**CDN-rotatie verdunt beacon scores.** Een bestemming achter een CDN (roterende antwoord-IP's) splitst over veel `src→dst` paren en scoort lager dan zijn werkelijke regelmaat: één testbestemming met 333 verbindingen landde op slechts 0,629. Een vast IP forceren (`curl --resolve`) herstelt een schone hoge-score beacon.

**Whitelist legitieme periodieke diensten voor je automatiseert.** NetBird update-checks (100%) en Ubuntu NTP (99%) zijn leerboek-beacons. RITA-output in RPZ voeden zonder whitelist zou je eigen update- en tijdinfrastructuur NXDOMAIN geven. De whitelist is het eerste false-positive controlepunt in de [RITA→RPZ-pipeline](../decisions/rita-rpz-automation.md).

**Niet op de sandbox-eventbus.** RITA publiceert niets naar NATS en triggert de [Control Daemon](control-daemon.md) niet. De enige sandbox-gerichte output is het RPZ feed-bestand; al het andere blijft in de CLI-views op de parallelle stack.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Component: Zeek](zeek.md)
- [Component: ioc2rpz + RPZ](ioc2rpz.md)
- [Component: Suricata](suricata.md)
- [Concept: Behavioral analyse](../concepts/behavioral-analysis.md)
- [Concept: RPZ](../concepts/rpz.md)
- [Beslissing: RITA→RPZ-automatisering](../decisions/rita-rpz-automation.md)
- [Bevinding: DC-segment mirror-limiet](../findings/dc-segment-mirror-limit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
- [Runbook: RITA→RPZ-integratie](../runbooks/15-rita-rpz-integration.md)
