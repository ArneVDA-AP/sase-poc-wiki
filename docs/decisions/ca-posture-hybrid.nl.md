---
title: "Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)"
tags: [decision, zero-trust, sase, network]
---

# Beslissing: Entra ID CA + NetBird Posture Checks (Hybride drie-gate model)

**Status:** Geïmplementeerd (Gate 1: Entra ID CA, 5 policies; Gate 2: Intune device compliance; beide live sinds Verslag40, 2 juni 2026; Gate 3: SWG operationeel). NetBird posture checks blijven een optionele, niet-geïmplementeerde defense-in-depth-uitbreiding.  
**Datum:** Architectuur april 2026 (Addendum E.2); geïmplementeerd 2 juni 2026 (Verslag40)

## Context

De vraag die deze beslissing beantwoordt: hoe verifieer je zowel *identiteit* als *apparaatstatus* voordat je netwerktoegang verleent?

Een vroege stellingname (Verslag18, Bevinding 17.7, "Entra ID Conditional Access vervangt NetBird posture checks") werd voor het eerst herzien in Verslag27 onder de aanname van *onbeheerde BYOD*. Op onbeheerde apparaten kan CA apparaatconformiteit niet controleren (dat vereist Intune-enrollment), dus werden NetBird posture checks voorgesteld als een aanvullende apparaatstatus-gate (Doc7, Addendum E.1).

Die premisse veranderde vervolgens. Een scopecorrectie stelde vast dat de in-scope apparaten **beheerde, Intune-ge-enrollde Windows-apparaten** zijn, niet onbeheerde BYOD (Addendum E.2; zie [Beslissing: Scope beheerde Windows-apparaten](managed-devices-scope.md)). Met beheerde apparaten *kan* CA apparaatstatus evalueren, via Intune-conformiteitsattestatie. Dus werd **Intune-conformiteit het primaire apparaatpostuurmechanisme (Gate 2)**, en NetBird posture checks werden gedegradeerd tot een **optionele defense-in-depth-uitbreiding** die nooit in de sandbox is gedeployd. Verslag40 (2 juni 2026) implementeerde dit beheerde-apparaatmodel.

## Overwogen opties

> De eerste drie rijen vormen de **BYOD-tijdperk-analyse** (Addendum E.1 / Doc7). Ze worden bewaard als beslissingsgeschiedenis; hun gedeelde premisse (onbeheerde BYOD) werd vervangen door de beheerde-apparaat-scopecorrectie (laatste rij).

| Optie | Voor | Tegen |
|-------|------|-------|
| **Alleen CA** | Eén controlepunt; door Microsoft beheerd | Op *onbeheerde* BYOD is CA-apparaatconformiteit niet beschikbaar (vereist Intune). CA kan MFA en aanmeldingsrisico afdwingen, maar geen OS-versie of AV-status |
| **Alleen NetBird posture** | Apparaatstatus zonder MDM | Evalueert alleen bij tunnel-bouwtijd. Kan gestolen-inloggegevenrisico niet evalueren, MFA niet afdwingen, of afwijkende aanmelding niet detecteren. Procescontrole is spoofbaar |
| **CA + NetBird posture (BYOD-tijdperk-plan)** | Elk dekt wat de andere niet kan | **Premisse vervangen.** Ging uit van onbeheerde BYOD; zodra apparaten beheerd waren, verving Intune-conformiteit NetBird posture als de apparaat-gate |
| **CA + Intune-conformiteit (beheerd apparaat, geïmplementeerd)** | Attestatie-gebaseerd apparaatpostuur op authenticatietijdstip; niet spoofbaar door de eindgebruiker; zelfde control plane als identiteit | Beperkt tot Intune-ge-enrollde Windows-apparaten |

## Beslissing

Beheerde-apparaat **drie-gate model** (Verslag40):

- **Gate 1 (Identiteit):** Entra ID Conditional Access (5 policies, authenticatietijdstip)
- **Gate 2 (Apparaat):** Intune-apparaatconformiteit (attestatie-gebaseerd postuur, geëvalueerd bij authenticatie via CA Policy 5 en op Intune's periodieke cyclus)
- **Gate 3 (Inhoud):** SWG-pipeline (Squid SSL-Bump + ClamAV + DLP + Unbound RPZ, elke aanvraag)

NetBird posture checks blijven *beschikbaar* als een optionele, onafhankelijke defense-in-depth-laag (andere timing, namelijk tunnel-bouw, en ander mechanisme, namelijk client-side controle), maar werden **niet gedeployd**: met beheerde apparaten dekt Intune-conformiteit de apparaatpostuureis van de rubric al via attestatie in plaats van een spoofbare procescontrole.

**Wat CA (Gate 1) dekt:**
- MFA-handhaving
- Aanmeldingsrisicoevaluatie (gelekte inloggegevens, onmogelijke reis, afwijkende aanmeldpatronen via Entra ID Protection; beschikbaar omdat `aplab.be` een A5-licentie heeft die P2 omvat)
- Benoemde locaties (geo-blokkering)
- Blokkering van legacy-authenticatie

**Wat Intune-conformiteit (Gate 2) dekt:**
- OS-versie (attestatie-gebaseerd, minimum `10.0.22000.0`)
- Defender antivirus actief + security intelligence actueel + real-time protection
- Windows Defender Firewall actief

Omdat Intune de werkelijke apparaatstatus rapporteert via de beheeragent (attestatie), is Gate 2 **niet spoofbaar door de eindgebruiker**. Dit is het belangrijkste voordeel ten opzichte van de NetBird `process_check` die het verving, die alleen verifieert dat een binary op een pad bestaat.

## Geïmplementeerde Gate 1-policies

| Policy | Status |
|--------|--------|
| CA Policy 1 (MFA vereist) | ✅ Actief |
| CA Policy 2 (Geo-blokkering, alleen België) | Report-only: Success bewezen, On bij demo |
| CA Policy 3 (Legacy-auth blokkering) | ✅ Actief |
| CA Policy 4 (Risico-gebaseerde blokkering) | ✅ Actief |
| CA Policy 5 (Conform apparaat vereist) | Report-only: Success bewezen, On bij demo |

Alle policies richten zich op ALLE resources (niet een specifieke app): CA-policies gericht op een specifieke app vuren nooit bij NetBird/Zitadel OIDC-aanmeldingen omdat CA matcht op tokenresource (Microsoft Graph), niet op client-app. Dit weerlegt de kernscopingstrategie van Addendum E sectie E.2.2. User-scoping via persona-groepen met admin1 als break-glass uitsluiting.

## Geïmplementeerde Gate 2-controles

| Controle | Status |
|----------|--------|
| Intune-conformiteitsbeleid (OS-versie, Defender AV + firewall, real-time protection) | ✅ Actief |
| mobile01 (`2ITCSC1A-MOB-1`) Entra joined + Intune enrolled + conform | ✅ Geverifieerd |

BitLocker/TPM geschrapt: rubric vereist "device posture", niet encryptie. Drie posturecontroles: OS-versie, AV, firewall. Policy op Report-only tot demo-voorbereiding.

## Gevolgen

- Gate 1 (Entra ID CA) en Gate 2 (Intune-conformiteit) zijn beide live sinds Verslag40: vier CA-policies die handhaven (MFA, legacy-auth-blokkering, risico-blokkering) plus Policy 5 (conform apparaat) en Geo-Block op Report-only tot demo. Gate 3 (SWG-pipeline) werkt onafhankelijk van identiteit/apparaat bij elke aanvraag, dus alle drie de gates zijn operationeel, niet alleen Gate 3
- Activering van Gate 1 vereist MFA-pre-registratie voor testaccounts vóór het inschakelen van policies (niet-geregistreerde MFA veroorzaakt een lus die zelfs de verificatiesessie blokkeert)
- De admins-persona valt onder geen enkele CA-policy: `2itcsc1a_admin1` is het enige lid en is de break-glass-uitsluiting op elke policy, dus deze is bewust ongereguleerd om uitsluiting te voorkomen
- Intune Gate 2 is afhankelijk van het un-bumped bereikbaar zijn van de Microsoft control plane: `*.microsoftonline.com` en `enterpriseregistration.windows.net` staan op de Squid splice/no-bump-lijst, anders breken apparaatregistratie en de conform-apparaatcontrole (Policy 5)
- **Optionele NetBird posture-laag (niet gedeployd):** waren NetBird posture checks als apparaat-gate gebruikt, dan zou de OS-controle de kernelversie lezen (`10.0.19041` = Windows 10 2004, de eerste met de native WireGuard-kernelmodule) en zou de AV-controle een `process_check` zijn die alleen verifieert dat een binary op een pad bestaat (spoofbaar, en blind voor of definities actueel zijn of real-time scanning actief is). Intune-attestatie (Gate 2, minimum OS `10.0.22000.0`) vervangt dit; diepe inhouds-/endpointbescherming is Gate 3 (ClamAV)

Zie ook: [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)
