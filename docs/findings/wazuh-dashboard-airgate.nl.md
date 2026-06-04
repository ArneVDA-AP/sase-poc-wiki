---
title: "Bevinding: Wazuh Dashboard Apps-sectie werkt niet in air-gapped omgeving"
tags: [finding, wazuh, dashboard, air-gap]
---

# Bevinding: Wazuh Dashboard Apps-sectie werkt niet in air-gapped omgeving

**Component:** [Wazuh](../components/wazuh.nl.md)  
**Ernst:** Valkuil (bekende beperking)

## Wat er gebeurde

De app-modules van het Wazuh Dashboard (Security Events, de Agents-tab) zijn gegate: in plaats van te laden meldt het dashboard de manager als **"Status: Offline"** met **"Updates status: Error checking updates"** en herleidt het hard naar de server-APIs-pagina (API-configuratie). De **Discover**-interface daarentegen werkt volledig.

Dit is misleidend, want de manager-API is in werkelijkheid gezond — `authenticate`, `info` en `stats` geven alle HTTP 200. Het "Offline"-oordeel is een vals-negatief dat door één falende controle wordt geproduceerd, niet door een onbereikbare manager.

## Oorzaak

Twee internet-afhankelijke controles falen in de air-gapped sandbox en cascaderen tot de offline-bepaling:

1. Op de manager retourneert `GET /manager/version/check` **HTTP 500**. Dit endpoint voert een internet-afhankelijke update-/CTI-controle uit; zonder uitgaand internet in de sandbox geeft het een fout in plaats van een versie terug.
2. De `POST /api/check-api` van het dashboard retourneert vervolgens **HTTP 500 — "Could not obtain manager UUID"**. Het dashboard interpreteert deze fout als een offline manager, zet "Status: Offline" / "Updates status: Error checking updates", en gate't de app-modules achter de herleiding naar de API-configuratie.

De gate is dus **geen** DNS-/hostnaamresolutieprobleem en geen eindeloze hang — het is een snelle 500 van een internet-afhankelijke gezondheidscontrole die als "manager offline" wordt geïnterpreteerd.

## Oplossing / workaround

Er is geen definitieve fix toegepast; de gate wordt aanvaard als een bekende air-gap-beperking. Gebruik **Discover** als primaire analyse-interface — het bevraagt de indexer rechtstreeks en wordt niet beïnvloed door de versie-/UUID-controle van de manager:

- Navigeer naar Discover in de Wazuh Dashboard-zijbalk
- Selecteer het `wazuh-alerts-*` index pattern
- Gebruik KQL- of Lucene-querysyntax voor filtering (bijv. `rule.groups:"sase"`)
- Maak opgeslagen zoekopdrachten en visualisaties aan indien nodig

Kandidaat-fixpaden (te herbekijken met Gerben, nog niet gevalideerd): schakel de update-/CTI-controle uit aan de managerzijde (zijn `api.yaml`) of de overeenkomstige dashboard-plugininstelling zodat `check-api` niet langer afhangt van de versiecontrole, en pas toe met `wazuh-control restart` (nooit `docker restart` — bind-mounts worden in-place overschreven). Zie [Component: Wazuh](../components/wazuh.nl.md).

## Lessen

- Een "Offline"-oordeel in de dashboard-app is geen bewijs dat de manager down is — bevestig rechtstreeks met de manager-API (`authenticate`/`info`/`stats` die 200 geven) voordat je connectiviteit gaat onderzoeken.
- Verwacht in air-gapped Wazuh-deployments dat de app-modules worden gegate door de internet-afhankelijke update-/versiecontrole; plan analyseworkflows vanaf het begin rond Discover.
- De beperking treft enkel de dashboard-gemakslaag — dataverzameling, agent-enrollment, regelverwerking en alerting zijn onaangetast.
