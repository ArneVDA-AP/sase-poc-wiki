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

**CA-policies werken nog steeds:** Entra ID Conditional Access evalueert op het Entra ID `/authorize`-endpoint gericht op de NetBird-appregistratie `2ITCSC1A-Netbird-Sandbox` (`11803ee8-eb15-462c-a286-5415c17a29c6`). Of de omleiding afkomstig is van Zitadel of van het NetBird Dashboard is irrelevant: de gebruiker authenticeert rechtstreeks met Entra ID, en CA valt in voor die appregistratie.

> **App-registratiewissel (V30, mei 2026).** De eerdere build hergebruikte een gedeelde appregistratie (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`), wat een verborgen koppeling tussen de sandbox en de teamprojectstack veroorzaakte. V30 maakte een toegewijde `2ITCSC1A-Netbird-Sandbox`-registratie (`11803ee8-eb15-462c-a286-5415c17a29c6`) en richtte de Entra ID-federatie van Zitadel daarop. De sandbox-specifieke registratie is de huidige waarde; `cebe0d74…` is verouderd.

**TLS:** Het quickstart-script probeert Let's Encrypt voor `netbird.sandbox.local`, maar dit mislukt (niet-publieke hostnaam). In plaats daarvan wordt de `tls internal`-directive van Caddy (ingebouwde lokale CA) gebruikt. OIDC vereist HTTPS, niet een publiek vertrouwd certificaat.

## Gevolgen

- Alle CA-gerichte policies moeten verwijzen naar de NetBird-appregistratie `2ITCSC1A-Netbird-Sandbox` (`11803ee8-eb15-462c-a286-5415c17a29c6`), niet naar een Zitadel-registratie
- Nieuwe gebruikers die authenticeren via Entra ID verschijnen als "in behandeling" in het NetBird Dashboard; handmatige goedkeuring vereist (beveiligingsfunctie van het quickstart-script)
- Zitadel gebruikt geneste `roles` JSON; NetBird verwacht een platte `groups`-array; transformatie van groepsclaim kan nodig zijn in Zitadel
- `docker compose up -d caddy` (niet `restart`) is vereist wanneer volume-mounts wijzigen
- CA-policies moeten app-specifiek zijn (gericht op alleen de NetBird-appregistratie) omdat de `aplab.be`-tenant gedeeld is; tenant-brede policies zouden andere projecten en studenten beïnvloeden

## Implementatiedetail: groups-claim via twee Zitadel Actions

Het groups-claim-mechanisme dat de volledige identiteitsketen aandrijft is geïmplementeerd via twee Zitadel Actions (v1):

- **Action 1 (External Auth):** Ontvangt het Entra ID-token bij login, mapt de Entra weergavenaam-string → clean naam via allowlist (`2ITCSC1A-Studenten` → `Studenten`), schrijft naar user metadata `sase_groups`
- **Action 2 (Complement Token):** Leest `sase_groups` metadata bij token-uitgifte, injecteert `groups`-claim in JWT via `setClaim`

Beide actions hebben `allowed-to-fail: true`. Dit is een bewuste beslissing. Fail-closed handhaving hoort in de policy-laag (NetBird ACL's, Squid persona-ACL's), niet in de authenticatie-action. Een action-fout moet identity-verrijking degraderen, niet alle gebruikers buitensluiten.

Zie: [Beslissing: GroupSync Pad B](../decisions/groupsync-pad-b.md)
