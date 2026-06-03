---
title: "Bevinding: NetBird JWT allow groups-veld veroorzaakt volledige lockout bij misconfiguratie"
tags: [finding, netbird, jwt, authentication]
---

# Bevinding: NetBird JWT allow groups-veld veroorzaakt volledige lockout bij misconfiguratie

**Component:** [NetBird](../components/netbird.md)  
**Ernst:** Blokkerend

## Wat er gebeurde

Het "JWT allow groups"-veld in het NetBird Dashboard (Settings -> Groups) werd ingevuld met een groepsnaam die niet overeenkwam met enige JWT-claimwaarde. Het resultaat was onmiddellijk en totaal: ALLE gebruikers kregen 401 Unauthorized. De volledige NetBird-deployment werd ontoegankelijk — geen enkele gebruiker kon authenticeren, geen enkele peer kon verbinden, en het dashboard zelf werd onbruikbaar om de fout te corrigeren.

## Oorzaak

NetBird controleert de JWT `groups`-claim tegen de allow list bij elke authenticatiepoging. Het veld werkt als een **whitelist**, niet als een filter: als de allow list niet leeg is en geen enkele groep in de JWT van de gebruiker overeenkomt met een entry in de lijst, wordt authenticatie geweigerd. Er is geen "soft fail" of respijtperiode — de lockout is onmiddellijk en geldt voor alle gebruikers, inclusief administrators.

## Oplossing / workaround

**Preventie:** Laat het veld **leeg** (= geen filtering, alle JWT groups worden gesynchroniseerd). Vul het alleen in als je zeker bent dat de waarden exact overeenkomen met wat in de JWT `groups`-claim staat.

**Herstel van lockout:** Directe SQLite-toegang op de management-container is vereist:

```bash
# Toegang tot de NetBird management-container
docker exec -it netbird-management sh

# Zoek en wis de JWT allow groups-instelling
sqlite3 /var/lib/netbird/store.db
# Identificeer de relevante tabel/kolom en wis de allow groups-waarde
```

## Lessen

- Test JWT group sync eerst met het allow-veld leeg — vul het pas in na bevestiging van de exacte claimwaarden
- De veldnaam "JWT allow groups" is misleidend — het functioneert als een strikte whitelist, niet als een permissief filter
- Herstel vereist toegang op databaseniveau, wat niet altijd direct beschikbaar is in alle deploymentscenario's — heb altijd een back-upplan voordat je deze instelling wijzigt
- Referentie: NetBird issue #5399, v0.62-documentatie
