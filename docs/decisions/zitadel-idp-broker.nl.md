---
title: "Beslissing: Zitadel als IdP-broker"
tags: [decision, zero-trust, sase, network]
---

# Beslissing: Zitadel als IdP-broker

**Status:** Geïmplementeerd  
**Datum:** Maart 2026 (Verslag19, Doc6)

## Context

NetBird vereist een OIDC-provider voor gebruikersauthenticatie. Het handboek (v4 §19) beschreef een directe Entra ID-verbinding: Entra ID geconfigureerd als OIDC-provider rechtstreeks in het NetBird Dashboard. De `aplab.be`-tenant is een Microsoft 365 A5 Educational-tenant die wordt gedeeld door meerdere Syntrapxl-projecten.

Het NetBird quickstart-script (`getting-started-with-zitadel.sh`) installeert Zitadel automatisch als primaire OIDC-uitgever.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Directe Entra ID in NetBird Dashboard** | Eenvoudigere keten; één component minder | Quickstart-script installeert Zitadel sowieso; omzeilen vereist handmatige NetBird-setup |
| **Zitadel als broker met Entra ID externe IdP** | Volgt quickstart; voegt gebruikersbeheerlaag toe (rollen, groepen) onafhankelijk van Entra ID; centraal beheerpunt | Twee-staps OIDC-keten: NetBird → Zitadel → Entra ID |

## Beslissing

Zitadel als primaire OIDC-uitgever (geïnstalleerd door quickstart-script), met Entra ID verbonden als externe IdP aan Zitadel.

Dit was een architecturale realiteit van het quickstart-script, geen bewuste keuze tussen opties. Het quickstart produceert een werkende stack met Zitadel als uitgever. Entra ID wordt daarna toegevoegd als externe IdP in de console van Zitadel. De resulterende keten is architecturaal solide: Zitadel biedt een centrale gebruikersbeheerlaag, en Entra ID biedt schoolaccountauthenticatie.

**CA-policies werken nog steeds:** Entra ID Conditional Access evalueert op het Entra ID `/authorize`-eindpunt gericht op de NetBird-appregistratie (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`). Of de omleiding afkomstig is van Zitadel of van het NetBird Dashboard is irrelevant — de gebruiker authenticeert rechtstreeks met Entra ID, en CA valt in voor die appregistratie.

**TLS:** Het quickstart-script probeert Let's Encrypt voor `netbird.sandbox.local` — dit mislukt (niet-publieke hostnaam). In plaats daarvan wordt de `tls internal`-directive van Caddy (ingebouwde lokale CA) gebruikt. OIDC vereist HTTPS, niet een publiek vertrouwd certificaat.

## Gevolgen

- Alle CA-gerichte policies moeten verwijzen naar de NetBird-appregistratie (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`), niet naar een Zitadel-registratie
- Nieuwe gebruikers die authenticeren via Entra ID verschijnen als "in behandeling" in het NetBird Dashboard — handmatige goedkeuring vereist (beveiligingsfunctie van het quickstart-script)
- Zitadel gebruikt geneste `roles` JSON; NetBird verwacht een platte `groups`-array — transformatie van groepsclaim kan nodig zijn in Zitadel
- `docker compose up -d caddy` (niet `restart`) is vereist wanneer volume-mounts wijzigen
- CA-policies moeten app-specifiek zijn (gericht op alleen de NetBird-appregistratie) omdat de `aplab.be`-tenant gedeeld is — tenant-brede policies zouden andere projecten en studenten beïnvloeden
