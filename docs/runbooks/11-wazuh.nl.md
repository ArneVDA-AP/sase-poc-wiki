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

> **Valkuil: CPU-piek bij opstarten.** De Wazuh Manager en Indexer veroorzaken aanzienlijke CPU-belasting tijdens de initiële opstart door glibc-compatibiliteitsproblemen in de Docker-omgeving. Dit is een bekend probleem — wacht 2-3 minuten tot de stack stabiliseert. Zie [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.nl.md).

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

Deploy een Python-script op mgmt01 dat NATS-events doorstuurt naar Wazuh:

1. De forwarder abonneert zich op `security.alert.*`-subjects op NATS
2. Voor elk ontvangen event stuurt het door naar het Wazuh API `/events`-endpoint
3. Dit creëert een **dual-write-architectuur**: beveiligingsevents stromen onafhankelijk naar zowel de Control Daemon (voor geautomatiseerde respons) als Wazuh (voor logging, correlatie en onderzoek)

> **Opmerking: Dual-write, niet serieel.** De NATS-forwarder en Control Daemon zijn onafhankelijke subscribers. Als Wazuh uitvalt, blijft de Control Daemon events verwerken en andersom. Geen van beiden is afhankelijk van de ander.

Controleer of de forwarder draait en doorstuurt:

```bash
# Trigger een testevent (bijv. Suricata-alert)
# Controleer vervolgens de Wazuh API voor het event
curl -k -u <wazuh-api-gebruiker>:<wachtwoord> https://localhost:55000/security/events?limit=5
```

---

## Stap 3: Wazuh-agent deployen op pop01

Installeer en configureer de Wazuh-agent op pop01 (agent-ID 001):

1. Installeer het Wazuh-agentpakket op pop01
2. Configureer de agent om verbinding te maken met de Wazuh Manager op mgmt01
3. Registreer de agent (agent-ID: 001)

Configureer directe logverzameling — de agent monitort deze bestanden:

| Logbestand | Bron | Doel |
|------------|------|------|
| `/var/log/suricata/eve.json` | Suricata IDS | IDS-alerts en flowgegevens |
| `/var/log/squid/access.log` | Squid proxy | Proxy-toegangsevents |

4. Maak aangepaste decoders voor het SASE-eventformaat zodat Wazuh de gestructureerde velden uit Suricata- en Squid-logs kan parsen

Herstart de agent en verifieer registratie:

```bash
# Op Wazuh Manager (mgmt01)
docker exec wazuh-manager /var/ossec/bin/agent_control -l
# Verwacht: agent 001 (pop01) — Active
```

---

## Stap 4: Aangepaste Wazuh-regels configureren

Maak aangepaste regels in het 100500-100600-bereik voor SASE-specifieke detecties:

| Regelbereik | Categorie | Voorbeelden |
|-------------|-----------|-------------|
| 100500-100519 | IDS-correlatie | Hoge-severity Suricata-alerts, herhaalde alerts van dezelfde bron |
| 100520-100539 | DLP-alerts | Gevoelige datapatroonmatches, geblokkeerde uploads |
| 100540-100559 | DNS RPZ-blokkades | RPZ-geblokkeerde domeinen, herhaalde blokkeerpogingen |
| 100560-100599 | Gereserveerd | Toekomstig gebruik |

Plaats aangepaste regels in de lokale regelmap van de Wazuh Manager. Herstart de Wazuh Manager na het toevoegen van regels:

```bash
docker exec wazuh-manager /var/ossec/bin/wazuh-control restart
```

---

## Stap 5: CASB Layer 2 configureren — M365 API-integratie

Stel de ms-graph-module van Wazuh in om de Microsoft 365 Management Activity API te pollen voor cloudbeveiligingsevents:

1. Configureer de `ms-graph`-module in de Wazuh Manager-configuratie:
   - API-endpoint: Microsoft 365 Management Activity API
   - Authenticatie: App registration-referenties voor aplab.be-tenant
   - Polling-interval: geconfigureerd volgens API-vereisten

2. Maak aangepaste regels in de 100600-familie voor M365-schendingen:

| Regel | Detectie | Respons |
|-------|----------|---------|
| 100600 | SharePoint anonieme deellink aangemaakt | Alert + Active Response |
| 100601 | OneDrive externe share gedetecteerd | Alert |
| 100602 | Verdachte aanmelding vanuit ongebruikelijke locatie | Alert |

3. Deploy Active Response-script `sharepoint_remediate.sh`:
   - Getriggerd door regel 100600
   - Trekt anonieme/externe deellinks in via de Microsoft Graph API
   - Logt de herstelactie terug naar Wazuh

> **Opmerking:** Dit is CASB Layer 2 in de drielaagse CASB-architectuur. Zie [Beslissing: CASB drie lagen](../decisions/casb-three-layers.nl.md) voor het volledige ontwerp.

---

## Stap 6: Events verifiëren in Wazuh Dashboard

1. Open het Wazuh Dashboard in een browser
2. Navigeer naar **Discover**
3. Zoek naar SASE-events:

```
rule.groups: "sase" OR data.alert.signature_id: *
```

**Verwacht:** Events van Suricata, Squid, DNS RPZ en DLP verschijnen in het dashboard met correct geparseerde velden.

> **Valkuil: Dashboard kan aanvankelijk niet laden.** Als het Wazuh Dashboard een verbindingsfout toont gerelateerd aan `airgate`-hostnaamresolutie, is dit een bekend DNS-probleem in de Docker-omgeving. Zie [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.nl.md).

---

## Stap 7: CASB Active Response verifiëren

Test de SharePoint-herstelworkflow:

1. Maak een anonieme deellink aan op een SharePoint-document in de aplab.be-tenant
2. Wacht tot de ms-graph-module pollt en de schending detecteert
3. Wazuh moet regel 100600 triggeren
4. Active Response-script `sharepoint_remediate.sh` moet uitvoeren
5. Verifieer: de anonieme deellink is ingetrokken (niet meer toegankelijk)

Controleer het Active Response-log:

```bash
docker exec wazuh-manager cat /var/ossec/logs/active-responses.log | tail -20
```

---

## Checklist

- [ ] Wazuh Manager v4.14.5 draait op mgmt01
- [ ] Wazuh Indexer (OpenSearch) draait en is gezond
- [ ] Wazuh Dashboard toegankelijk via browser
- [ ] NATS-naar-Wazuh-forwarder geabonneerd op `security.alert.*`
- [ ] Wazuh-agent 001 (pop01) geregistreerd en actief
- [ ] Aangepaste decoders parsen SASE-eventformaat
- [ ] Aangepaste regels 100500-100600 gedeployd
- [ ] M365 ms-graph-module pollt Management Activity API
- [ ] Aangepaste regels 100600-familie detecteren SharePoint-schendingen
- [ ] Active Response `sharepoint_remediate.sh` trekt deellinks in
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
