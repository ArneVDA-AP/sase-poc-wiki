---
title: "Beslissing: NetBird Service-User PAT voor Identity Bridge"
tags: [decision, netbird, identity-bridge, api]
---

# Beslissing: NetBird Service-User PAT voor Identity Bridge

**Status:** Geïmplementeerd  
**Datum:** Mei 2026 (Verslag31)

## Context

Identity Bridge moet de NetBird Management API pollen om groepslidmaatschap van Zitadel te synchroniseren naar NetBird-peergroepen. De voor de hand liggende aanpak is een Personal Access Token (PAT) van een bestaande admingebruiker te gebruiken. NetBird-issue #3127 onthult echter een destructief neveneffect: wanneer de API wordt gepolld met een reguliere gebruikers-PAT, verwijdert het systeem bij elke API-aanroep alle JWT-gepropageerde auto-groups van alle peers die bij die gebruiker horen. Dit verbreekt stilzwijgend groepsgebaseerde beleidstoewijzingen voor de peers van de gebruiker die pollt.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Reguliere gebruikers-PAT** | Eenvoudig op te zetten; hergebruik van bestaand admin-account | Triggert issue #3127: elke API-poll verwijdert JWT-gepropageerde auto-groups van alle peers van de PAT-eigenaar. Verbreekt groepsgebaseerde toegangspolicies stilzwijgend |
| **Service-user PAT met admin-rol** | Service-user heeft geen JWT-gepropageerde auto-groups en geen peers, het verwijdereffect heeft niets om op te werken | Vereist het aanmaken van een dedicated service-user met admin-rol in NetBird; PAT moet veilig opgeslagen worden |

## Beslissing

Een dedicated service-user genaamd `identity-bridge` met admin-rol wordt aangemaakt in NetBird. De PAT van deze service-user wordt gebruikt voor alle API-polling door Identity Bridge.

Het kernpunt: issue #3127 verwijdert JWT-gepropageerde auto-groups van alle peers van de PAT-eigenaar. Een service-user heeft geen JWT-gepropageerde auto-groups (deze authenticeert nooit via OIDC) en heeft geen peers (het is geen apparaat). Daarom heeft het verwijdereffect niets om op te werken: het wordt uitgevoerd op een lege set.

## Gevolgen

- Er moet een service-user met admin-rol bestaan in NetBird management. Dit is een geprivilegieerd account dat uitsluitend voor API-toegang wordt gebruikt.
- De PAT moet veilig opgeslagen worden in het `.env`-bestand van de Identity Bridge-deployment. Deze mag niet worden gecommit naar versiebeheer.
- Als de service-user per ongeluk peers of groepen krijgt toegewezen, kan issue #3127 opnieuw optreden. De service-user moet peer-vrij en groep-vrij blijven.
- De workaround is specifiek voor de huidige NetBird-versie. Als #3127 upstream wordt opgelost, zou een reguliere gebruikers-PAT weer veilig zijn, maar de service-user-aanpak blijft hoe dan ook schoner.

Zie ook: [Bevinding: NetBird issue #3127](../findings/netbird-issue-3127.md), [Component: NetBird](../components/netbird.md), [Component: Identity Bridge](../components/identity-bridge.md)
