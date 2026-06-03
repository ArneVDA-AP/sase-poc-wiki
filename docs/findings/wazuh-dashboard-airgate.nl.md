---
title: "Bevinding: Wazuh Dashboard Apps-sectie werkt niet in air-gapped omgeving"
tags: [finding, wazuh, dashboard, air-gap]
---

# Bevinding: Wazuh Dashboard Apps-sectie werkt niet in air-gapped omgeving

**Component:** [Wazuh](../components/wazuh.md)  
**Ernst:** Valkuil (bekende beperking)

## Wat er gebeurde

De Wazuh Dashboard Apps-sectie is onbereikbaar. Het klikken op de "API"-sectie triggert een verbindingscontrole naar externe Wazuh-updateservers, die faalt in de air-gapped sandbox-omgeving. De UI toont een eindeloze laadspinner zonder foutmelding of time-outindicatie.

## Oorzaak

Het dashboard voert een versie/UUID-gezondheidscontrole uit tegen Wazuh's externe API (`https://api.wazuh.com/...`) als onderdeel van de Apps-sectie-initialisatie. Deze controle vereist internettoegang. In de air-gapped labomgeving hangt het verzoek voor onbepaalde tijd, waardoor de volledige Apps-sectie niet kan laden.

## Oplossing / workaround

Gebruik de **Discover**-interface (directe OpenSearch-query) als primaire analysetool. Discover biedt volledige mogelijkheden voor het zoeken, filteren en visualiseren van events — voldoende voor alle PoC-behoeften:

- Navigeer naar Discover in de Wazuh Dashboard-zijbalk
- Selecteer het `wazuh-alerts-*` index pattern
- Gebruik KQL- of Lucene-querysyntax voor filtering
- Maak opgeslagen zoekopdrachten en visualisaties aan indien nodig

De Apps-sectiefuncties (Agents-overzicht, SCA, Vulnerability Detection-dashboards) zijn niet beschikbaar, maar de onderliggende data is volledig toegankelijk via Discover.

## Lessen

- Air-gapped deployments van Wazuh moeten verwachten dat de Apps-sectie niet functioneel is — plan analyseworkflows rond Discover vanaf het begin
- Discover is de primaire analyse-interface en wordt niet beïnvloed door de externe API-afhankelijkheid
- Deze beperking heeft geen invloed op dataverzameling, agent-enrollment of alerting — alleen de dashboard-gemakslaag
