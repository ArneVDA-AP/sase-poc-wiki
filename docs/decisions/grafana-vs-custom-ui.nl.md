---
title: "Beslissing: Grafana boven een custom React UI"
tags: [decision, grafana, telemetry]
---

# Beslissing: Grafana boven een custom React UI

**Status:** Geïmplementeerd (PoC-gevalideerd op de parallelle stack, sandbox-integratie pending)  
**Datum:** April 2026 (Telemetry build v1)

## Context

Het oorspronkelijke telemetrieplan was om een eigen React/Next.js frontend te bouwen voor de monitoring stack: een custom dashboard om logs en metrics van het GNS3-lab te bekijken en om GNS3 nodes te starten/stoppen. Voordat het team weken frontend-werk vastlegde, herbekeek het of een custom UI iets toevoegde boven een bestaande tool.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Custom React/Next.js UI** | Volledige controle over de layout; kon node start/stop-controls in één scherm inbouwen | Weken werk; eindigt als een slechtere versie van wat Grafana al biedt (log tailing, metric grafieken, alerting); meer code, meer containers, meer onderhoud; de enige unieke feature (node start/stop) zit al in de GNS3 webinterface |
| **Grafana** | Industriestandaard, gratis; log tailing (Loki), metric grafieken (Prometheus) en alerting ingebouwd; dashboards-als-JSON overleven container rebuilds en zijn versiebeheerbaar | Geen eigen node start/stop-controls in het dashboard (overgelaten aan de GNS3 webinterface) |

## Beslissing

Grafana, zonder eigen frontend.

De doorslaggevende redenering: Grafana biedt al alles wat de telemetry stack nodig heeft, namelijk log tailing, metric grafieken en alerting. De enige feature die een custom UI zou toevoegen is het starten en stoppen van GNS3 nodes vanuit het dashboard, en GNS3 heeft daar al een eigen webinterface voor. Een eigen React-app bouwen zou een zwakkere versie van Grafana's bestaande mogelijkheden opleveren en tegelijk code, containers en onderhoudslast toevoegen.

De keuze voor Grafana betekende minder code, minder containers, minder onderhoud en een sneller resultaat.

## Gevolgen

- Dashboards worden als JSON-bestanden geschreven en bij het opstarten van Grafana geprovisioned. Ze overleven container rebuilds en zijn versiebeheerbaar. De keerzijde is dat geprovisionde dashboards en datasources **read-only zijn in de UI**; aanpassingen moeten via de YAML/JSON-bestanden plus een container restart.
- GNS3 node start/stop is **niet** in de telemetrie-UI geïntegreerd. Operators gebruiken de GNS3 webinterface voor nodebeheer en Grafana enkel voor observability.
- De beslissing beperkte de stack tot bewezen, onderhouden tools (Grafana + Prometheus + Loki), wat de latere migratie van poc-1a naar mgmt01 eenvoudiger maakte: enkel forwarders en datasources moesten mee, geen zelfgeschreven applicatie.

Zie ook: [Component: Telemetry Stack](../components/telemetry-stack.nl.md)
