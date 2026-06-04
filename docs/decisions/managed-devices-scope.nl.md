---
title: "Beslissing: Scope beheerde Windows-apparaten (Lectormandaat R11)"
tags: [decision, scope, intune, entra-id, architecture]
---

# Beslissing: Scope beheerde Windows-apparaten (Lectormandaat R11)

**Status:** Geïmplementeerd (lectormandaat R11, 21 april 2026)  
**Datum:** 21 april 2026

## Context

De oorspronkelijke projectscope richtte zich op BYOD (Bring Your Own Device) met naar schatting 4000+ persoonlijke apparaten. Onder BYOD is device posture-handhaving beperkt omdat studenten hun persoonlijke laptops niet inschrijven bij Intune MDM. Lectormandaat R11 veranderde de scope naar beheerde Windows-apparaten — Intune-beheerd en Entra ID-joined — wat het beschikbare handhavingsmodel voor de architectuur fundamenteel wijzigt.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **BYOD (oorspronkelijke scope)** | Bredere apparaatdekking; weerspiegelt real-world heterogene omgeving | Device posture-controles beperkt tot wat NetBird kan verifiëren zonder MDM (OS-versie, procesaanwezigheid). Geen compliance-attestatie. Kan proxyconfiguratie, firewallregels of certificaten niet op OS-niveau afdwingen |
| **Beheerde Windows-apparaten (lectormandaat)** | Intune-compliance-attestatie beschikbaar (Gate 2). MDM kan proxyconfiguratie, firewallregels en certificaten pushen. Smallere scope maar diepere handhaving | Beperkt tot Windows-apparaten die Entra-joined en Intune-enrolled zijn. Dekt geen persoonlijke apparaten of niet-Windows-platforms |

## Beslissing

Beheerde Windows-apparaten (Intune-beheerd, Entra ID-joined) zoals voorgeschreven door lectormandaat R11. Dit stelt Gate 2 in staat Intune-compliancepolicies te gebruiken voor device posture-verificatie in plaats van uitsluitend te vertrouwen op NetBird posture checks. Endpoint-handhaving wordt vereenvoudigd doordat MDM het proxy PAC-bestand, Windows Firewall-regels en root-CA-certificaten rechtstreeks naar ingeschreven apparaten kan pushen.

## Gevolgen

- **Gate 2-herontwerp:** Device posture-verificatie verschuift van NetBird posture checks (die OS-versie en procesaanwezigheid verifiëren bij tunnel-bouwtijd) naar Intune-compliancepolicies (die OS-versie, Defender AV, firewallstatus en real-time protection verifiëren via MDM-attestatie).
- **Addendum F (Linux PoP transparante proxy)** is grotendeels achterhaald. Het transparante proxy-ontwerp ging uit van BYOD-apparaten die niet centraal geconfigureerd konden worden. Met beheerde apparaten wordt proxyconfiguratie via Intune gepusht — transparante interceptie is niet langer het primaire deploymentmodel.
- **Identity Bridge-mechanisme is ongewijzigd.** De JWT-gebaseerde groepssynchronisatiepipeline (Entra ID -> Zitadel -> NetBird -> Identity Bridge -> NATS -> Squid) werkt op gebruikersidentiteit, niet op apparaattype. De scopewijziging beïnvloedt apparaathandhaving, niet identiteitsfederatie.
- **sitepc01 en mobile01** moeten Entra-joined en Intune-enrolled zijn om geldige test-endpoints te zijn. Beide zijn ingeschreven (mobile01 in Verslag40, sitepc01/SITE01 in Verslag44), maar alleen **mobile01 (`2ITCSC1A-MOB-1`) is geverifieerd als Compliant** (Verslag40). **sitepc01 is nog niet geverifieerd als conform:** Microsoft Defender staat uit op zijn Tiny11-image, wat de AV-controle van `2ITCSC1A-SASE-Windows-Compliance` doet falen — een bekende compliance-gap (Verslag44, B44.12) die gedicht moet worden door Defender te herstellen voordat compliance-evaluatie op dat endpoint wordt ingeschakeld.
- **Certificaatdistributie:** Het interne CA-rootcertificaat kan worden gepusht via een Intune-configuratieprofiel in plaats van handmatige installatie op elk apparaat.

Zie ook: [Beslissing: Entra ID CA + NetBird Posture Checks](ca-posture-hybrid.md), [Concept: Zero Trust](../concepts/zero-trust.md), [Component: NetBird](../components/netbird.md)
