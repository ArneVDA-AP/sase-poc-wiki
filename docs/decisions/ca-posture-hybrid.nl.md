---
title: "Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)"
tags: [decision, zero-trust, sase, network]
---

# Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)

**Status:** Architectuur gevalideerd en gedocumenteerd (Addendum E, april 2026). Gates 1 en 2 nog niet geactiveerd in sandbox — gepland na evaluatie op 20 april.  
**Datum:** April 2026 (Verslag27, Doc7)

## Context

De aanvankelijke architectuur (Verslag18, Bevinding 17.7) stelde dat "Entra ID Conditional Access NetBird posture checks vervangt." Dit was onjuist. Verslag27 identificeerde en corrigeerde de fout na een gedetailleerde analyse van wat elke technologie daadwerkelijk doet op onbeheerde BYOD.

De vraag: hoe verificeer je zowel *identiteit* als *apparaatstatus* voor BYOD-studenten die hun persoonlijke laptops niet inschrijven bij Intune MDM?

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Alleen CA** | Één controlepunt; door Microsoft beheerd | CA-apparaatconformiteitscontroles vereisen Intune-inschrijving — niet beschikbaar voor onbeheerde BYOD. CA kan MFA en aanmeldingsrisico afdwingen, maar niet OS-versie of AV-status |
| **Alleen posture checks** | Apparaatstatus zonder MDM | Posture checks evalueren bij tunnel-bouwtijd — na enige CA-controle zou hebben gevuurd. Kan gestolen-inloggegevenrisico niet evalueren, MFA niet afdwingen, of afwijkende aanmelding niet detecteren |
| **Hybride (CA + posture)** | Elk dekt wat de andere niet kan | Twee systemen te onderhouden; beide gemarkeerd als GEPLAND in sandbox |

## Beslissing

Hybride model: Entra ID Conditional Access als Gate 1 (authenticatietijdstip) + NetBird Posture Checks als Gate 2 (tunnel-bouwtijdstip). Beide zijn aanvullend, niet inwisselbaar.

**Wat CA (Gate 1) dekt wat posture niet kan:**
- MFA-handhaving
- Aanmeldingsrisicoevaluatie (gelekte inloggegevens, onmogelijke reis, afwijkende aanmeldpatronen via Entra ID Protection — beschikbaar omdat `aplab.be` een A5-licentie heeft die P2 omvat)
- Benoemde locaties (geo-blokkering)
- Blokkering van legacy-authenticatie

**Wat posture (Gate 2) dekt wat CA niet kan:**
- OS-versie-verificatie (geen Intune-inschrijving vereist)
- AV-procesverificatie (`C:\Program Files\Windows Defender\MsMpEng.exe`)
- NetBird-clientversiehandhaving
- Geo-controle als verdediging in de diepte (andere GeoIP-database dan CA)

De les: verifieer altijd wat een technologie daadwerkelijk kan doen op het doeltype apparaat (onbeheerde BYOD), niet op het ideale apparaat (Intune-ingeschreven bedrijfsapparaat). CA-apparaatconformiteitscontroles zijn structureel niet beschikbaar voor onbeheerde BYOD.

## Geplande Gate 1-policies (Addendum E)

Alle policies gericht op appregistratie `cebe0d74-be9f-49ac-9f35-65f11586c1bb` (niet tenant-breed — `aplab.be` is gedeeld):

| Policy | Mechanisme |
|--------|-----------|
| `SASE-PoC-MFA-Required` | MFA vereisen voor alle gebruikers |
| `SASE-PoC-Geo-Block` | Toegang buiten benoemde locaties blokkeren (België, Nederland) |
| `SASE-PoC-Block-Legacy-Auth` | Legacy-authenticatieprotocollen blokkeren |
| `SASE-PoC-Sign-in-Risk` | Hoog aanmeldingsrisico blokkeren (Entra ID Protection) |

## Geplande Gate 2-controles (Addendum E)

| Controle | Drempel |
|----------|---------|
| `os_version_check` (Windows) | Kernel `10.0.19041` (Windows 10 2004 — eerste native WireGuard-kernelmodule) |
| `nb_version_check` | Minimale NetBird-clientversie op moment van activering |
| `process_check` | `C:\Program Files\Windows Defender\MsMpEng.exe` moet actief zijn |
| `geo_location_check` | België (verdediging in de diepte — andere database dan Gate 1) |

## Gevolgen

- Gate 3 (SWG-pipeline) is de enige momenteel operationele gate in de sandbox — biedt inhoudsniveaubescherming ongeacht identiteit/apparaat
- Activering van Gate 1 vereist MFA-pre-registratie voor testaccounts vóór het inschakelen van policies (niet-geregistreerde MFA veroorzaakt een lus die zelfs de verificatiesessie blokkeert)
- OS-versiecontrole gebruikt kernelversie, niet marketingversie — `10.0.19041` correspondeert met Windows 10 2004 (mei 2020 update), de eerste versie met native WireGuard-kernelmoduleondersteuning
- `process_check` voor AV is een basisconformiteitscontrole — het verifieert niet of AV-definities actueel zijn of dat realtime scanning actief is; volledige endpointbescherming is Gate 3 (ClamAV)

Zie ook: [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)
