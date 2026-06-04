---
title: "Control Daemon"
tags: [control-daemon, nats, netbird, quarantine, threat-scoring]
---

# Control Daemon

**Rol:** Python-daemon op mgmt01 die de NATS event bus consumeert, per-peer threat scores bijhoudt via sliding-window decay, en peers in quarantaine plaatst door ze te verwijderen uit beleiddragende persona-groepen (deny-by-default).  
**Versie:** Python 3.12 (nats-py >= 2.13.0, redis >= 5.0.0, httpx >= 0.27.0)  
**Configuratielocatie:** `config/mgmt01/control-daemon/` in de repo (gezaghebbend — Addendum J §J.6.7 is verouderd)

## Hoe het werkt in deze stack

De control daemon is de real-time handhavingsmotor. Het abonneert zich op `security.alert.>` en `identity.>` op de NATS-bus en verwerkt events in drie fasen:

1. **Verrijking:** Koppelt overlay-IP's aan peeridentiteit via een interne `identity.>` cache (gevuld vanuit Identity Bridge events)
2. **Scoring:** Dispatcht op het `producer`-veld (niet het subject). Elk eventtype heeft een configureerbaar gewicht. Scores gebruiken sliding-window decay via Redis sorted sets.
3. **Beslissing:** Wanneer de score van een peer de quarantainedrempel overschrijdt (standaard: 80), verwijdert de daemon de peer uit alle beleiddragende persona-groepen via de NetBird Groups API. Zonder lidmaatschap van een persona-groep blokkeert deny-by-default alle connectiviteit.

**Huidige staat:** `ENFORCE=false` (dry-run) tot demovoorbereiding (Sessie 11).

## Configuratie

- **Scoring-gewichten:** `proxy_block` verwijderd uit scoring (ambient OS-ruis veroorzaakte false positives — Windows NCSI/telemetry). IDS-events zijn enkel log (C2 beacon response hoort bij Zeek/RITA, niet bij eigen correlatie).
- **Quarantainemechanisme:** Strip persona-groepen van peer → deny-by-default. NIET via een aparte deny-groep.
- **Dequarantaine:** Herstelt originele groepen vanuit Redis-backup. Opgelost: NetBird retourneert `peers:null` voor lege groepen (niet `[]`), wat dequarantaine deed crashen.
- **Redis:** Threat score opslag + sessiestatus op `redis:7-alpine`, poort 6379 enkel localhost.
- **Beleidsgroepen:** `NETBIRD_POLICY_GROUPS=Studenten,Docenten,Admins` — enkel deze worden gestript tijdens quarantaine. Infrastructuurgroepen zijn structureel onaantastbaar.

## Integratiepunten

| Interface | Richting | Details |
|-----------|----------|---------|
| NATS JetStream | Inkomend (consumer) | Durable + DeliverPolicy.NEW consumer op `security.alert.>` (overleeft restarts, geen herafspelen van historische events); ephemeral + DeliverPolicy.ALL consumer op `identity.>` (herbouwt de in-memory identity-map bij elke start) |
| NetBird Management API | Uitgaand | Groups API GET/PUT voor quarantaine/dequarantaine |
| Redis | Bidirectioneel | Threat scores, sessiestatus, groepsbackup |

## Bekende problemen / valkuilen

- **`peers:null` coercion:** NetBird retourneert `null` in plaats van `[]` voor groepen zonder peers. Opgelost met `(group.get("peers") or [])` coercion (V35).
- **`proxy_block` false positives:** Ambient Windows NCSI/telemetry-verkeer genereert proxy block events. Verwijderd uit scoring-gewichten (V35).
- **IDS-correlatie niet geimplementeerd:** De malware-branch dekt dezelfde real-time quarantainecapaciteit met native attributie; C2 beacon response hoort bij het Zeek/RITA-domein. Zie [Beslissing: Control Daemon Scope](../decisions/control-daemon-scope.md).

## Gerelateerd

- [Beslissing: Control Daemon Scope](../decisions/control-daemon-scope.md)
- [Component: NATS JetStream](nats-jetstream.md)
- [Component: NetBird](netbird.md)
