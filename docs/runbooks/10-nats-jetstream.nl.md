---
title: "Runbook: NATS JetStream Event Bus"
tags: [runbook, nats-jetstream, docker, redis, event-bus]
---

# Runbook: NATS JetStream Event Bus

**Node(s):** mgmt01 (Docker — NATS + Redis + Control Daemon), pop01 (producers)
**Vereisten:** Docker op mgmt01, SSH-toegang tot pop01, Python 3 op pop01
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] Docker en Docker Compose geïnstalleerd op mgmt01
- [ ] SSH-toegang tot pop01
- [ ] Python 3 geïnstalleerd op pop01
- [ ] Suricata operationeel op pop01 ([Runbook 05](05-ids.nl.md))
- [ ] Squid operationeel op pop01 ([Runbook 03](03-proxy-wpad.nl.md))
- [ ] DNS RPZ operationeel ([Runbook 06](06-dns-threat-intel.nl.md))
- [ ] Python DLP ICAP operationeel ([Runbook 04](04-malware-dlp.nl.md))
- [ ] Identity Bridge operationeel ([Runbook 09](09-identity-bridge.nl.md))

---

## Stap 1: NATS en Redis toevoegen aan Docker Compose op mgmt01

Voeg de volgende services toe aan de Docker Compose-stack op mgmt01:

**NATS 2.14.1:**

- Pin de versie expliciet — gebruik niet `:latest`
- Configureer met het `accounts{}`-auth-model (niet `authorization{}`)
- Stel poort 4222 (clientverbindingen) beschikbaar op `192.168.122.23`
- Stel poort 8222 (monitoring) beschikbaar op `192.168.122.23`
- Koppel een persistent Docker-volume voor JetStream-opslag

> **Valkuil: JetStream-opslag moet op een persistent Docker-volume staan.** Zonder een benoemd volume gaan JetStream-gegevens verloren bij het opnieuw aanmaken van de container. Zie [Finding: NATS store dir](../findings/nats-store-dir.nl.md).

> **Valkuil: Gebruik het `accounts{}`-auth-model, niet `authorization{}`.** Het `authorization{}`-blok ondersteunt geen per-subject publish/subscribe-permissies die nodig zijn voor multi-tenant-isolatie. Zie [Beslissing: NATS accounts auth](../decisions/nats-accounts-auth.nl.md).

**Redis 7.x:**

- Gebruikt voor threat score-opslag door de Control Daemon
- Alleen beschikbaar stellen op het Docker-netwerk (niet extern), tenzij voor debugging

---

## Stap 2: Controleren of NATS draait

Start de Docker Compose-stack en controleer of NATS verbindingen accepteert:

```bash
docker compose up -d

# Controleer vanuit een nats-box-container
docker run --rm -it --network <compose-netwerk> natsio/nats-box nats pub test "hello"
# Verwacht: geen fout, bericht gepubliceerd
```

Controleer het monitoring-endpoint:

```bash
curl http://192.168.122.23:8222/varz
# Verwacht: JSON met serverinfo, JetStream ingeschakeld
```

---

## Stap 3: Connectiviteit verifiëren vanaf pop01

Bevestig vanaf pop01 dat NATS bereikbaar is via het managementnetwerk:

```bash
echo "PING" | nc -w 2 192.168.122.23 4222
# Verwacht: antwoord van NATS-server (PONG of INFO-regel)
```

Als dit faalt, controleer firewallregels op mgmt01 en verifieer dat NATS gebonden is aan `192.168.122.23:4222`, niet alleen `127.0.0.1`.

---

## Stap 4: Producers deployen op pop01

Deploy Python-gebaseerde event producers op pop01. Elke producer tailt een logbestand en publiceert gestructureerde events naar een NATS-subject:

### Producer: Suricata IDS

- Tailt `/var/log/suricata/eve.json`
- Publiceert naar subject: `security.alert.ids`
- Extraheert: alert signature, severity, bron-/doel-IP, SID

### Producer: Squid Proxy

- Tailt `/var/log/squid/access.log`
- Publiceert naar subject: `security.alert.proxy`
- Extraheert: URL, client-IP, HTTP-status, ACL-beslissing

### Producer: DNS RPZ

- Unbound Python-module-integratie
- Publiceert naar subject: `security.alert.dns`
- Extraheert: opgevraagd domein, RPZ-actie (NXDOMAIN/redirect), feedbron

Controleer of elke producer publiceert:

```bash
# Vanuit een nats-box-container, abonneer om events te zien
nats sub "security.alert.>"
# Trigger een testevent (bijv. curl naar testmyids.com via proxy)
# Verwacht: event verschijnt op het abonnement
```

---

## Stap 5: Producers deployen op mgmt01

### Producer: Python DLP

- Geïntegreerd in de ICAP-server (draait al vanuit [Runbook 04](04-malware-dlp.nl.md))
- Publiceert naar subject: `security.alert.dlp`
- Extraheert: gevonden patroon, bestandsnaam, client-IP, severity

### Producer: Identity Bridge

- Geïntegreerd in de Identity Bridge-service ([Runbook 09](09-identity-bridge.nl.md))
- Publiceert naar subjects:
  - `identity.login` — wanneer een nieuwe peer verbindt
  - `identity.group_change` — wanneer de personagroep van een peer wijzigt

---

## Stap 6: Control Daemon deployen op mgmt01

De Control Daemon is de centrale eventprocessor die beveiligingsevents consumeert en geautomatiseerde responsacties onderneemt.

1. Deploy als Docker-container op mgmt01
2. Configureer abonnementen: `security.alert.*` (wildcard)
3. Configureer Redis-verbinding voor threat score-opslag

**Threat scoring:**

- Redis-gebaseerde sliding-window-scoring per peer-IP
- Elk alerttype draagt een configureerbaar scoregewicht bij
- Wanneer de score van een peer de quarantainedrempel overschrijdt: de Control Daemon verwijdert de peer uit zijn personagroep via de NetBird Groups API
- Wanneer de score onder de hersteldrempel daalt: de peer wordt opnieuw aan zijn personagroep toegevoegd

Zie [Component: Control Daemon](../components/control-daemon.nl.md) voor het scoring-algoritme en drempelconfiguratie.

---

## Stap 7: End-to-end-verificatie

Trigger een echte alert en volg deze door de hele pipeline:

1. **Trigger:** Vanuit mobile01, voer uit `curl -x http://100.70.154.79:3128 http://testmyids.com/`
2. **Suricata:** Controleer of eve.json de alert registreert (SID 2100498)
3. **Producer:** Verifieer dat het event verschijnt op het `security.alert.ids`-subject
4. **Redis:** Controleer of de threat score voor het peer-IP is verhoogd:

```bash
docker exec redis redis-cli GET "threat:<peer-ip>"
```

5. **Control Daemon:** Controleer Control Daemon-logs voor het ontvangen event en de scoringsbeslissing

Als de score de quarantainedrempel overschrijdt, verifieer dat de peer is verwijderd uit zijn personagroep op het NetBird Dashboard.

---

## Checklist

- [ ] NATS 2.14.1 draait op mgmt01 met `accounts{}`-auth-model
- [ ] JetStream-opslag op persistent Docker-volume
- [ ] Poort 4222 bereikbaar vanaf pop01
- [ ] Poort 8222 monitoring-endpoint reageert
- [ ] Redis draait op mgmt01
- [ ] Suricata-producer publiceert naar `security.alert.ids`
- [ ] Squid-producer publiceert naar `security.alert.proxy`
- [ ] DNS RPZ-producer publiceert naar `security.alert.dns`
- [ ] DLP-producer publiceert naar `security.alert.dlp`
- [ ] Identity Bridge publiceert naar `identity.login` / `identity.group_change`
- [ ] Control Daemon consumeert `security.alert.*` en scoort in Redis
- [ ] Quarantaine-actie functioneel (peer verwijderd uit groep bij drempeloverschrijding)

---

## Gerelateerd

- [Component: NATS JetStream](../components/nats-jetstream.nl.md)
- [Component: Control Daemon](../components/control-daemon.nl.md)
- [Beslissing: NATS accounts auth](../decisions/nats-accounts-auth.nl.md)
- [Beslissing: Control Daemon scope](../decisions/control-daemon-scope.nl.md)
- [Finding: NATS store dir](../findings/nats-store-dir.nl.md)
- [Runbook 05: IDS](05-ids.nl.md)
- [Runbook 09: Identity Bridge](09-identity-bridge.nl.md)
