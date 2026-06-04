---
title: "Bevinding: Wazuh Dashboard Offline-redirect veroorzaakt door lege manager-UUID"
tags: [finding, wazuh, dashboard, air-gap, docker]
---

# Bevinding: Wazuh Dashboard Offline-redirect veroorzaakt door lege manager-UUID

**Component:** [Wazuh](../components/wazuh.nl.md)  
**Ernst:** Valkuil (opgelost 2 juni 2026)

## Wat er gebeurde

Na een `docker compose down -v` (waarmee named Docker volumes werden gewist) gevolgd door een manager-downgrade (4.14.5 → 4.14.3) toonden de app-modules van het Wazuh Dashboard (Security Events, Agents-tab) **"Status: Offline"** / **"Updates status: Error checking updates"** en werd bij elke keer laden hard doorverwezen naar de API-configuratiepagina. Tegelijkertijd raakte de pop01-agent (ID 001) verbroken. De Discover-interface bleef gedurende de hele periode volledig werken.

## Oorzaak

Twee afzonderlijke problemen combineerden zich:

**1. Lege manager-UUID (oorzaak van de Offline-redirect)**  
`docker compose down -v` wist named Docker volumes, inclusief `global.db`. Op een nieuwe `global.db` is de manager-UUID leeg (`"uuid": ""`). De `POST /api/check-api` van het dashboard retourneert HTTP 500 — "Could not obtain manager UUID"; het dashboard interpreteert dit als een offline manager, zet "Status: Offline" en gate't alle app-modules achter de API-configuratie-redirect.

**2. Versie-skew (oorzaak van agent-ontkoppeling)**  
De OPNsense-plugin `os-wazuh-agent` levert een vaste agent-versie (4.14.5). De compatibiliteitsregel van Wazuh vereist **manager ≥ agent**. Door de manager te downgraden naar 4.14.3 belandde hij onder de agent, wat enrollment-weigering veroorzaakte en de `(4101)` versie-waarschuwing in de agent-logs.

**Belangrijk:** `GET /manager/version/check` → HTTP 500 ("Error checking updates") wordt veroorzaakt door de air-gap die het CTI-update-endpoint blokkeert, maar het is een *apart* probleem van de Offline-redirect. In Wazuh 4.14.5 (fix #8130) is deze controle non-blocking — het veroorzaakt **niet** de Offline-redirect. De redirect wordt uitsluitend veroorzaakt door de lege UUID.

## Twee structurele regels

**Regel 1 — De agent-versie bepaalt de manager-ondergrens.**  
De OPNsense-plugin pint de agent-versie (momenteel 4.14.5). Een FreeBSD-package downgraden is brick-risk. Houd manager altijd ≥ agent. Controleer de agent-versie vóór elke manager-versiewijziging.

**Regel 2 — Versie-werk = in-place tag-aanpassing, nooit `down -v`.**  
`docker compose down -v` vernietigt named volumes: UUID (`global.db`), agent-registratie (`client.keys`) en geïndexeerde alerts. Gebruik voor versie-werk: tag aanpassen in `docker-compose.yml`, `docker compose pull`, daarna `docker compose up -d` (zonder `-v`). De UUID en agent-registratie blijven bewaard.

## Oplossing

Opgelost op 2 juni 2026 via in-place bump naar Wazuh 4.14.5 GA (zonder `-v`), waardoor de `global.db`-UUID van de 4.14.3-run bewaard bleef. De pop01-agent nam opnieuw deel en toont Active. De dashboard-app laadt zonder redirect.

Alle acceptatiecriteria voldaan:
- `wazuh-control info` → `v4.14.5`
- `GET /manager/info` → `uuid` niet-leeg; dashboard-app laadt zonder redirect
- `agent_control -l` → `001 OPNsense.internal Active`
- Discover `rule.id:100540` → `data.producer:unbound`-document geïndexeerd

## Lessen

- De "Status: Offline"-redirect wordt veroorzaakt door een **lege manager-UUID**, niet door DNS of de air-gap CTI-controle. Verifieer met `GET /manager/info` (Dev Tools in het dashboard) vóór je connectiviteitsproblemen gaat onderzoeken.
- `docker compose down -v` is destructief voor Wazuh: UUID en agent-registratie zitten in named volumes, niet in bind-mounts. Gebruik altijd `docker compose up -d` (zonder `-v`) voor versie-werk.
- De CTI-500 ("Error checking updates") is cosmetisch in Wazuh 4.14.5+ en kan genegeerd worden in air-gapped deployments.
- De OPNsense agent-versie bepaalt de manager-ondergrens — zet de manager nooit onder de agent-versie.
