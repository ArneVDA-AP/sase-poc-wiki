---
title: "Beslissing: NATS accounts{}-authenticatie voor JetStream"
tags: [decision, nats, jetstream, authentication]
---

# Beslissing: NATS accounts{}-authenticatie voor JetStream

**Status:** Geïmplementeerd  
**Datum:** Mei 2026 (Verslag32)

## Context

NATS vereist een authenticatiemodel om de toegang tot JetStream-streams en -consumers te beveiligen. Identity Bridge en Control Daemon publiceren en subscriben beide op JetStream-subjects, waarvoor `$JS.API.>`-toegang nodig is voor het aanmaken van streams, het beheren van consumers en het bevestigen van berichten. Er bestaan twee authenticatiemodellen in NATS: het eenvoudigere `authorization{}`-blok (single-account) en het meer capabele `accounts{}`-blok (multi-account met expliciete permissietoekenningen).

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **`authorization{}` (single-account)** | Eenvoudigere configuratie; minder regels in `nats-server.conf` | Weigert JetStream API-operaties op `$JS.API.>` stilzwijgend. Geen foutmeldingen — operaties falen simpelweg zonder uitleg. Empirisch bevestigd als niet-functioneel voor JetStream |
| **`accounts{}` (multi-account)** | Volledige JetStream API-toegang via expliciete `$JS.API.>`-permissietoekenningen; ondersteunt account-niveau isolatie | Uitgebreidere configuratie; gebruikers moeten binnen benoemde accounts worden gedefinieerd |

## Beslissing

Het `accounts{}`-model wordt gebruikt voor NATS-authenticatie. Dit is empirisch vastgesteld: het `authorization{}`-model weigerde stilzwijgend alle JetStream-operaties ondanks ogenschijnlijk correcte permissieconfiguratie. Overschakelen naar `accounts{}` met expliciete `$JS.API.>` publish/subscribe-permissies loste het probleem onmiddellijk op.

Alle NATS-gebruikers (Identity Bridge, Control Daemon) worden gedefinieerd binnen een dedicated account dat expliciete permissies heeft voor JetStream API-subjects, applicatiespecifieke subjects en delivery-subjects.

## Gevolgen

- Alle NATS-gebruikers moeten worden gedefinieerd binnen benoemde accounts in `nats-server.conf`. Het top-level `authorization{}`-blok kan niet naast `accounts{}` worden gebruikt.
- Secrets (wachtwoorden/tokens van gebruikers) worden geïnjecteerd via omgevingsvariabele-interpolatie met hex-gecodeerde waarden, niet base64. NATS ondersteunt `$ENV_VAR`-syntax in het configuratiebestand.
- Het toevoegen van een nieuwe NATS-client vereist het definiëren van de gebruiker binnen het juiste account en het expliciet toekennen van subject-niveau permissies.
- Het `accounts{}`-model biedt account-niveau isolatie als bijkomend voordeel — als toekomstige componenten gescheiden permissiedomeinen nodig hebben, kunnen extra accounts worden toegevoegd zonder herstructurering.

Zie ook: [Component: NATS](../components/nats-jetstream.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: Control Daemon](../components/control-daemon.md)
