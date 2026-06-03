---
title: "Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)"
tags: [decision, zero-trust, sase, network]
---

# Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)

**Status:** Geïmplementeerd (Gates 1+2 operationeel)  
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

## Geïmplementeerde Gate 1-policies

| Policy | Status |
|--------|--------|
| CA Policy 1 — MFA vereist | ✅ Actief |
| CA Policy 2 — Geo-blokkering (alleen België) | Report-only: Success bewezen — → On bij Sessie 11 |
| CA Policy 3 — Legacy-auth blokkering | ✅ Actief |
| CA Policy 4 — Risico-gebaseerde blokkering | ✅ Actief |
| CA Policy 5 — Conform apparaat vereist | Report-only: Success bewezen — → On bij Sessie 11 |

Alle policies richten zich op ALLE resources (niet een specifieke app) — CA-policies gericht op een specifieke app vuren nooit bij NetBird/Zitadel OIDC-aanmeldingen omdat CA matcht op tokenresource (Microsoft Graph), niet op client-app. User-scoping via persona-groepen met admin1 als break-glass uitsluiting.

## Geïmplementeerde Gate 2-controles

| Controle | Status |
|----------|--------|
| Intune-conformiteitsbeleid (OS-versie, Defender AV + firewall, real-time protection) | ✅ Actief |
| mobile01 (`2ITCSC1A-MOB-1`) Entra joined + Intune enrolled + conform | ✅ Geverifieerd |

BitLocker/TPM geschrapt — rubric vereist "device posture", niet encryptie. Drie posturecontroles: OS-versie, AV, firewall. Policy op Report-only tot demo-voorbereiding (Sessie 11).

## Gevolgen

- Gate 3 (SWG-pipeline) is de enige momenteel operationele gate in de sandbox — biedt inhoudsniveaubescherming ongeacht identiteit/apparaat
- Activering van Gate 1 vereist MFA-pre-registratie voor testaccounts vóór het inschakelen van policies (niet-geregistreerde MFA veroorzaakt een lus die zelfs de verificatiesessie blokkeert)
- OS-versiecontrole gebruikt kernelversie, niet marketingversie — `10.0.19041` correspondeert met Windows 10 2004 (mei 2020 update), de eerste versie met native WireGuard-kernelmoduleondersteuning
- `process_check` voor AV is een basisconformiteitscontrole — het verifieert niet of AV-definities actueel zijn of dat realtime scanning actief is; volledige endpointbescherming is Gate 3 (ClamAV)

Zie ook: [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)
