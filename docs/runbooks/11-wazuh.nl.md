---
title: "Runbook: Wazuh SIEM"
tags: [runbook, wazuh, siem, docker, nats-jetstream]
---

# Runbook: Wazuh SIEM

**Node(s):** mgmt01 (Docker — Wazuh-stack + NATS-forwarder), pop01 (Wazuh-agent)
**Vereisten:** [Runbook 10: NATS JetStream](10-nats-jetstream.nl.md) afgerond (NATS operationeel), Docker op mgmt01
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] NATS JetStream operationeel op mgmt01 ([Runbook 10](10-nats-jetstream.nl.md))
- [ ] Docker en Docker Compose geïnstalleerd op mgmt01
- [ ] SSH-toegang tot pop01 voor agentinstallatie
- [ ] Voldoende schijfruimte op mgmt01 voor OpenSearch-indices

---

## Stap 1: Wazuh-stack deployen op mgmt01

Voeg de Wazuh-stack toe aan Docker Compose op mgmt01. De stack bestaat uit drie services:

| Service | Versie | Rol |
|---------|--------|-----|
| Wazuh Manager | v4.14.5 | Alertverwerking, rule engine, active response |
| Wazuh Indexer | (OpenSearch) | Eventopslag en zoekmogelijkheden |
| Wazuh Dashboard | (OpenSearch Dashboards) | Web-UI voor visualisatie en onderzoek |

> **Valkuil: Indexer vereist host-CPU-passthrough.** De Wazuh Indexer vereist glibc x86-64-v2 CPU-features; het standaard QEMU `kvm64`-model crasht hem met illegal-instruction-fouten. Stel het GNS3-node-CPU-model in op `host` voor mgmt01 en zet `vm.max_map_count=262144`. Het opstarten duurt dan 2-3 minuten om te stabiliseren (tijdelijke `port 9200`-fouten tijdens herinitialisatie zijn normaal). Zie [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.nl.md).

Start de stack:

```bash
docker compose up -d
```

Wacht tot alle drie containers gezond melden:

```bash
docker compose ps
# Alle drie services moeten "healthy" of "running" tonen
```

---

## Stap 2: NATS-naar-Wazuh-forwarder deployen

Deploy een Python-container op mgmt01 die NATS-events naar Wazuh overbrugt:

1. De forwarder is een **durable PULL-consumer** (`DeliverPolicy.NEW`, `AckPolicy.EXPLICIT`) op `security.alert.>`
2. Voor elk ontvangen event schrijft het een regel NDJSON naar het gedeelde `wazuh_nats_ingest`-volume (`/ingest/security_alerts.json`). De Wazuh manager tailt dat bestand via een `<localfile><log_format>json</log_format></localfile>`-blok — dit is het werkende ingestiepad; rechtstreeks naar de manager-socket schrijven en naar de Wazuh-API posten zijn beide geëvalueerd en verworpen.
3. Dit creëert een **dual-write-architectuur**: beveiligingsevents stromen onafhankelijk naar zowel de Control Daemon (voor geautomatiseerde respons) als Wazuh (voor logging, correlatie en onderzoek)

> **Opmerking: Dual-write, niet serieel.** De NATS-forwarder en Control Daemon zijn onafhankelijke consumers. Als Wazuh uitvalt, blijft de Control Daemon events verwerken en andersom. Geen van beiden is afhankelijk van de ander.

> **Valkuil: reconnect-resilience.** Een `--force-recreate` van de NATS-container wijzigt zijn IP en verbreekt elke hangende consumerverbinding. De forwarder moet opnieuw verbinden met `max_reconnect_attempts=-1` en DNS opnieuw resolven (geen gecachete IP).

Controleer of de forwarder draait en doorstuurt:

```bash
# Trigger een testevent (bijv. browser EICAR-download → security.alert.malware)
# Bevestig vervolgens dat de NDJSON-regel in het gedeelde ingestiebestand is beland:
docker exec single-node-wazuh.manager-1 tail -n 5 /ingest/security_alerts.json
# En bevestig dat het geïndexeerd is: Wazuh Dashboard → Discover, index wazuh-alerts-*, query rule.groups:"sase"
```

---

## Stap 3: Wazuh-agent deployen op pop01

Installeer en configureer de Wazuh-agent op pop01 (agent-ID 001):

1. Installeer het Wazuh-agentpakket op pop01
2. Configureer de agent om verbinding te maken met de Wazuh Manager op mgmt01
3. Registreer de agent (agent-ID: 001)

Configureer logverzameling op hostniveau. De agent levert de OPNsense-**host**telemetrie (hij monitort **niet** Suricata/Squid/c-icap/Unbound — die bereiken Wazuh via hun eigen NATS-producers en de forwarder, dus de agent heeft `intrusion_detection_events=false` om dubbel-ingest te vermijden):

| Feed | Bron | Doel |
|------|------|------|
| audit | OPNsense-auditlog | Configuratie-/adminacties |
| configd | OPNsense configd | Service-/configevents |
| filter | OPNsense-firewall (filterlog) | Packet-filter-events |
| kernel | OPNsense-kernellog | Systeem-/kernelevents |
| pkg | OPNsense-packagemanager | Package-installatie-/updateevents |

4. De agent heeft ook `active_response=false` — Active Response (handhaving) is eigendom van de Control Daemon (Laag 3), niet van de agent op pop01.

Herstart de agent en verifieer registratie:

```bash
# Op Wazuh Manager (mgmt01)
docker exec single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
# Verwacht: agent 001 (pop01) — Active
```

---

## Stap 4: Aangepaste Wazuh-regels configureren

Maak aangepaste regels voor de busproducers. Ze hangen allemaal aan één basisregel (100500) die matcht op het `producer`-veld; de kinderen classificeren en kennen severity toe:

| Regel | Producer / conditie | Niveau |
|-------|---------------------|--------|
| 100500 | basis — `producer` ∈ {suricata, squid, dlp, c-icap, unbound} | 0 |
| 100501 | Squid proxy-event | 5 |
| 100510 | Suricata IDS-alert | 6 |
| 100511 | Suricata IDS, severity 1 | 10 |
| 100513 | Suricata IDS, severity 3 | 3 |
| 100520 | DLP-match | 10 |
| 100530 | malware (ClamAV / c-icap) | 12 |
| 100540 | DNS RPZ-blokkade | 8 |

> **Valkuil:** de kinderen hangen aan de basis via `if_sid 100500`, dus de pcre2-alternatie van de basis moet elke producer bevatten. `unbound` ontbrak aanvankelijk, waardoor regel 100540 nooit afvuurde totdat het werd toegevoegd (V38.9).

Plaats aangepaste regels in `config/mgmt01/wazuh/local_rules.xml` (gemount in de manager). Herstart na het bewerken van regels de manager — nooit `docker restart`:

```bash
docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control restart
```

---

## Stap 5: CASB Laag 2 configureren — M365 API-integratie

CASB Laag 2 pollt de Microsoft 365-auditactiviteit en remedieert riskante shares. Er is **geen** "Wazuh office365-module" in deze opstelling — een aangepaste producer doet de polling:

1. Deploy de `o365_producer`-container op mgmt01:
   - Pollt de **Office 365 Management Activity API** (`contentType=Audit.SharePoint`) met app-registration-referenties voor de aplab.be-tenant
   - Normaliseert elke audit-blob en publiceert die naar NATS-subject `security.alert.casb` (NATS-rol `casb-pub`)
   - Vanaf daar draagt de NATS→Wazuh forwarder het de manager in zoals elk ander busevent

2. Maak aangepaste regels in de 100600-familie (een zelfstandige basis, los van de 100500-basis van de busproducers):

| Regel | Detectie (`producer` = `o365`) | Niveau |
|-------|--------------------------------|--------|
| 100600 | basis — `producer` = `o365` | 0 |
| 100601 | `Operation` = AnonymousLinkCreated | 10 |
| 100602 | `Operation` = SharingLinkCreated AND `SharingLinkScope` = Anyone | 10 |
| 100603 | `Operation` = SharingSet AND `TargetUserOrGroupType` = Guest | 8 |

3. Deploy de Active Response-scripts (geport uit het CASB-project van het team), achter een `ENFORCE`-gate die **standaard op detect-only staat** (`ENFORCE=false`):
   - `sharepoint_remediate.sh` — getriggerd door regels **100601, 100602**; trekt de anonieme / anyone-link in via de Microsoft Graph API
   - `guest_remediate.sh` — getriggerd door regel **100603**; verwijdert de gasttoekenning
   - Beide lezen Graph-referenties uit `graph.env` (in het `wazuh_etc`-volume, nooit gecommit) en loggen de herstelactie

> **Valkuil: jq-afhankelijkheid.** De Active Response-scripts hebben `jq` nodig, dat niet in de standaard manager-image zit — zonder dit faalt het script stilletjes bij de eerste aanroep. Installeer het persistent via de `install_deps.sh`-entrypointwrapper plus een compose `entrypoint:`-override (V39); een gewone `yum install` overleeft een recreate niet.

> **Opmerking:** Dit is CASB Laag 2 in de drielaagse CASB-architectuur. Zie [Beslissing: CASB drie lagen](../decisions/casb-three-layers.nl.md) voor het volledige ontwerp.

---

## Stap 6: Events verifiëren in Wazuh Dashboard

1. Open het Wazuh Dashboard in een browser
2. Navigeer naar **Discover**
3. Zoek naar SASE-events:

```
rule.groups: "sase" OR data.alert.signature_id: *
```

**Verwacht:** Events van Suricata, Squid, DNS RPZ en DLP verschijnen in het dashboard met correct geparseerde velden.

> **Opmerking:** Het dashboard toonde eerder "Status: Offline" door een lege manager-UUID (gewist door `docker compose down -v`) — opgelost op 2 juni 2026 via in-place bump naar 4.14.5. De `Error checking updates` CTI-500 (air-gap, cosmetisch) blijft aanwezig maar is non-blocking in 4.14.5+. App-modules en Discover werken beide volledig. Zie [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.nl.md).

---

## Stap 7: CASB Active Response verifiëren

**Active Response staat standaard op detect-only (`ENFORCE=false`).** In detect-only-modus vuurt de regel af en logt het herstelscript wat het *zou* doen, maar er wordt geen Microsoft Graph API-aanroep gedaan. Zet `ENFORCE=true` (de gate in het script) enkel wanneer je live remediatie beoogt.

Test de SharePoint-herstelworkflow:

1. Maak een anonieme deellink aan op een SharePoint-document in de aplab.be-tenant
2. Wacht tot de `o365_producer` de Management Activity API pollt en het auditevent publiceert naar `security.alert.casb`
3. Wazuh moet regel 100601 (`AnonymousLinkCreated`) triggeren
4. Active Response-script `sharepoint_remediate.sh` moet uitvoeren (enkel loggen tenzij `ENFORCE=true`)
5. Verifieer met `ENFORCE=true` dat de anonieme deellink wordt ingetrokken via de Microsoft Graph API

> **Status: de revoke-keten is bewezen tot HTTP-204 via een offline stub; live intrekking tegen een echt OneDrive-/SharePoint-bestand wacht op provisioning van een testaccount** (een externe Microsoft-afhankelijkheid). Detect-only-modus en de regel→Active-Response-trigger zijn gevalideerd; live end-to-end intrekking is nog niet gedemonstreerd op de sandbox.

Controleer het Active Response-log:

```bash
docker exec single-node-wazuh.manager-1 cat /var/ossec/logs/active-responses.log | tail -20
```

---

## Checklist

- [ ] Wazuh Manager v4.14.5 draait op mgmt01
- [ ] Wazuh Indexer (OpenSearch) draait en is gezond
- [ ] Wazuh Dashboard toegankelijk via browser (Discover + app-modules werken beide; `Error checking updates` CTI-500 cosmetisch in air-gap, non-blocking)
- [ ] NATS-naar-Wazuh-forwarder consumeert `security.alert.>` → NDJSON naar `wazuh_nats_ingest`
- [ ] Wazuh-agent 001 (pop01) geregistreerd en actief
- [ ] Aangepaste busregels 100500–100540 gedeployd (`local_rules.xml`)
- [ ] Busregels classificeren proxy/IDS/DLP/malware/DNS-RPZ-events (kinderen hangen aan via `if_sid 100500`)
- [ ] `o365_producer` pollt Office 365 Management Activity API → `security.alert.casb`
- [ ] Aangepaste regels 100600-familie detecteren SharePoint-schendingen
- [ ] Active Response `sharepoint_remediate.sh` (ENFORCE detect-only standaard; live revoke pending)
- [ ] Events zichtbaar in Wazuh Dashboard → Discover

---

## Gerelateerd

- [Component: Wazuh](../components/wazuh.nl.md)
- [Component: NATS JetStream](../components/nats-jetstream.nl.md)
- [Component: Control Daemon](../components/control-daemon.nl.md)
- [Beslissing: CASB drie lagen](../decisions/casb-three-layers.nl.md)
- [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.nl.md)
- [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.nl.md)
- [Runbook 10: NATS JetStream](10-nats-jetstream.nl.md)
