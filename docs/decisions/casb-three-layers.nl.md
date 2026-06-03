---
title: "Beslissing: Drielaags CASB-architectuur"
tags: [decision, casb, sase, nats, wazuh, squid]
---

# Beslissing: Drielaags CASB-architectuur

**Status:** Geïmplementeerd  
**Datum:** Mei-juni 2026 (Verslag32-Verslag39)

## Context

De SASE-architectuur vereist Cloud Access Security Broker (CASB)-functionaliteit zoals gedefinieerd in Gartners CASB-subfunctieraamwerk. Een enkel handhavingsmodel kan niet alle vereiste tijdshorizonten dekken: blokkering op aanvraagtijdstip, post-event auditrespons en real-time cross-component correlatie. De architectuur moet meerdere handhavingslagen combineren om binnen PoC-scope betekenisvolle CASB-dekking te bereiken.

## Beslissing

Een drielaags CASB-model is geïmplementeerd, waarbij elke laag een andere tijdshorizon voor handhaving dekt:

**Laag 1 — Inline (Squid SWG-pipeline):**
Handhaving op aanvraagtijdstip. Squid verwerkt elk HTTP/HTTPS-verzoek via de ICAP-pipeline en past URL-filtering en identiteitsgebaseerde blokkering toe. Wanneer het verzoek van een gebruiker een geblokkeerde categorie raakt of overeenkomt met een DLP-patroon, wordt het geweigerd voordat het antwoord wordt teruggestuurd. Deze laag werkt op millisecondensniveau — de gebruiker ziet onmiddellijk een blokkeringspagina.

**Laag 2 — API-mode (Wazuh + Microsoft Graph API):**
Post-event audithandhaving. Wazuh neemt Microsoft 365-auditgebeurtenissen op via de Graph API en past detectieregels toe. Wanneer een beleidsschending wordt gedetecteerd (bijv. massale bestandsdownload vanuit SharePoint, extern delen van gevoelige documenten), triggert Wazuh Active Response geautomatiseerde remediatie — typisch binnen minuten na het optreden van de gebeurtenis. Deze laag dekt SaaS-activiteit die niet via de proxy verloopt.

**Laag 3 — Real-time (NATS + Control Daemon):**
Cross-component threat scoring en quarantaine. De Control Daemon subscribet op events van meerdere bronnen (Wazuh-alerts, Squid deny-logs, Suricata IDS-alerts) via NATS JetStream en onderhoudt een threat score per gebruiker. Wanneer de score een drempel overschrijdt, stuurt de daemon een quarantainecommando naar NetBird via de API, waarmee de peer van het netwerk wordt geïsoleerd. Deze laag werkt op secondensniveau en correleert signalen die geen enkel component alleen kan zien.

## Gevolgen

- Elke laag dekt een andere tijdshorizon: aanvraagtijdstip (milliseconden), auditrespons (minuten), cross-component correlatie (seconden).
- De drie lagen samen bieden ongeveer 15-25% Gartner CASB-subfunctiedekking. Dit is voldoende voor PoC-scope en demonstreert het architectuurpatroon zonder volledige CASB-compliance te claimen.
- Laag 1 vereist dat de volledige SWG-pipeline (Squid + c-icap + ClamAV + Python DLP) operationeel is.
- Laag 2 vereist Wazuh-integratie met Microsoft Graph API en passende M365-auditlogconfiguratie.
- Laag 3 vereist dat NATS JetStream, de Control Daemon en NetBird API-toegang gelijktijdig operationeel zijn.
- Het toevoegen van nieuwe CASB-subfuncties betekent het uitbreiden van een van de drie lagen in plaats van een vierde te bouwen — de architectuur is ontworpen om uitbreidbaar te zijn binnen het bestaande model.

Zie ook: [Component: Squid](../components/squid.md), [Component: Wazuh](../components/wazuh.md), [Component: Control Daemon](../components/control-daemon.md), [Component: NATS](../components/nats-jetstream.md), [Concept: SASE](../concepts/sase.md)
