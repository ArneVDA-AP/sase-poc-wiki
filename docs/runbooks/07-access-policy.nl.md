---
title: "Runbook: Access Policy"
tags: [runbook, entra-id, zero-trust, conditional-access, posture-check]
---

# Runbook: Access Policy

**Bron:** `Verslag40.md` (implementatie, 2 juni 2026); `Doc7_ZTNA_Context_Aware.md` + Addendum E.2 (ontwerpintentie)
**Node(s):** Entra ID (`aplab.be`-tenant) + Intune + NetBird Dashboard
**Vereisten:** Alle voorgaande runbooks afgerond (volledige SASE-stack operationeel)
**Status:** Geïmplementeerd (Verslag40). Gate 1: 5 CA policies (3 Aan, 2 Report-only); Gate 2: Intune device compliance operationeel; Gate 3 volledig operationeel.

> **Gates 1 en 2 zijn geïmplementeerd (Verslag40).** Gate 1 = vijf Conditional Access-policies: MFA Required (Aan), Block Legacy Auth (Aan), Risk-Based Block (Aan), Geo-Block alleen België (Report-only), Require Compliant Device (Report-only → "Report-only: Success"). De twee Report-only-policies blijven zo tot de demo-voorbereiding. Gate 2 = Intune device compliance (`2ITCSC1A-SASE-Windows-Compliance`), attestation-based. Dit is de gedeployde apparaat-gate, **niet** NetBird-posture checks (die een optionele, niet-gedeployde defense-in-depth-laag blijven). Gate 3 is operationeel en wordt gedekt door Runbooks 03-06.

---

## Vereistenchecklist

- [ ] Entra ID-tenant (`aplab.be`) toegankelijk
- [ ] A5-licentie actief (bevestigd 1 april 2026, bevat Entra ID P1 + P2)
- [ ] Rechten als Conditional Access-beheerder of Globale beheerder
- [ ] NetBird-app-registratie `2ITCSC1A-Netbird-Sandbox` (App ID `11803ee8-eb15-462c-a286-5415c17a29c6`) aanwezig in Entra ID (alleen voor NetBird-inloggen, **niet** een CA-doel; zie valkuil in Stap 1)
- [ ] NetBird-inloggen via Microsoft werkt (Runbook 02, Stap 8)
- [ ] Persona-beveiligingsgroepen aanwezig: `2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`
- [ ] Testaccounts (single-persona): een studenten-lid (bijv. `Student_1@aplab.be`, A5-gelicenseerd zodat Intune van toepassing is), een docenten-lid, en `2itcsc1a_admin1` (admins / break-glass)
- [ ] **MFA geregistreerd voor testaccount** (via `https://aka.ms/mfasetup`)
- [ ] mobile01 Entra-joined + Intune-enrolled (verschijnt in Intune als `2ITCSC1A-MOB-1`)
- [ ] NetBird-versie op mobile01 genoteerd: `netbird version`
- [ ] NetBird Dashboard toegankelijk: `https://netbird.sandbox.local`

> **Valkuil: MFA moet geregistreerd zijn VÓÓR het inschakelen van de CA MFA-policy.** Niet-geregistreerde MFA geeft een "MFA vereist maar niet geconfigureerd"-blokkering bij de volgende aanmelding (inclusief de verificatiesessie zelf). Registreer MFA eerst, schakel dan de policy in.

---

## [GEÏMPLEMENTEERD] Stap 1: Entra ID Conditional Access-policies aanmaken (Gate 1)

Navigeer naar: `https://entra.microsoft.com → Protection → Conditional Access → Policies → New policy`

> **Valkuil: target All resources, NIET de NetBird-app.** App-getargete CA policies vuren nooit bij NetBird/Zitadel OIDC-aanmeldingen. CA matcht op de *resource* van het token: een OIDC-login vraagt Microsoft Graph (`00000003-0000-0000-c000-000000000000`) op, niet de NetBird-app-registratie. Dus een policy gescoped op "Apps selecteren → NetBird" toont **Not Applied** bij elke NetBird-login (Verslag40, B40.4). Alle vijf policies targeten daarom **All resources** en beperken de blast radius via **user-scoping** (persona-groepen als include) in plaats van app-scoping (de inverse van Addendum E's strategie).
>
> **Geen resource-exclusions.** Vanaf 15 juni 2026 handhaaft Microsoft All-resources-policies die resource-*exclusions* dragen óók bij OIDC-only-aanmeldingen; vóór die datum worden zulke policies niet gehandhaafd bij OIDC-only-aanmeldingen (een bypass). All-resources *zonder* exclusions wordt aan beide kanten van die datum consistent gehandhaafd, dus elke policy gebruikt All resources zonder resource-exclusion (B40.5). Blast-radius-controle gebeurt uitsluitend via user-scoping: include de persona-groepen (`2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`), exclude `2itcsc1a_admin1` als break-glass op elke policy. Omdat `2itcsc1a_admin1` het enige lid is van de admins-persona en overal is uitgesloten, valt de admins-persona feitelijk onder geen enkele CA policy (B40.6, bewust geaccepteerd voor nu).

### Policy 1: 2ITCSC1A-SASE-PoC-MFA-Required

```
Naam: 2ITCSC1A-SASE-PoC-MFA-Required

Assignments:
  Gebruikers: Include 2ITCSC1A-Studenten + 2ITCSC1A-Docenten + 2ITCSC1A-Admins
  Uitsluiten: 2itcsc1a_admin1 (break-glass)

  Target resources: All resources (geen app-targeting, geen resource-exclusions)

  Conditions:
    Client-apps → Configureren: Ja
      aangevinkt: Browser
      aangevinkt: Mobiele apps en desktopclients

Access controls:
  Verlenen: Meervoudige verificatie vereisen

Enable policy: On
```

### Policy 2: 2ITCSC1A-SASE-PoC-Geo-Block

Maak eerst een Benoemde locatie aan:

```
Protection → Conditional Access → Named locations → + Countries location
Naam: 2ITCSC1A-SASE-PoC-Allowed-Countries
aangevinkt: België
aangevinkt: Nederland (optioneel, voor pendelende studenten)
```

Maak dan de policy aan:

```
Naam: 2ITCSC1A-SASE-PoC-Geo-Block

Assignments:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Locaties → Configureren: Ja
      Opnemen: Elke locatie
      Uitsluiten: 2ITCSC1A-SASE-PoC-Allowed-Countries

Access controls:
  Verlenen: Toegang blokkeren

Enable policy: Report-only (→ On at demo)
```

### Policy 3: 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

```
Naam: 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

Assignments:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Client-apps → Configureren: Ja
      aangevinkt: Exchange ActiveSync-clients
      aangevinkt: Andere clients
      (NIET aanvinken: Browser of Mobiele apps)

Access controls:
  Verlenen: Toegang blokkeren

Enable policy: On
```

### Policy 4: 2ITCSC1A-SASE-PoC-Risk-Block (vereist A5/P2)

```
Naam: 2ITCSC1A-SASE-PoC-Risk-Block

Assignments:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Aanmeldingsrisico → Configureren: Ja
      aangevinkt: Hoog
      aangevinkt: Gemiddeld

Access controls:
  Verlenen: Meervoudige verificatie vereisen
  (Voor hoog risico: overweeg Toegang blokkeren)

Enable policy: On
```

### Policy 5: 2ITCSC1A-SASE-Require-Compliant-Device

Deze policy verbindt Gate 1 met Gate 2: het vereist dat het apparaat door Intune als compliant is gemarkeerd (Stap 2). Let op: deze naam heeft **geen** "PoC"-segment.

```
Naam: 2ITCSC1A-SASE-Require-Compliant-Device

Assignments:
  Gebruikers: Include 2ITCSC1A-Studenten / Uitsluiten 2itcsc1a_admin1
    (studenten-only; docent1/admin1 zijn bewust ongelicenseerd, dus studenten-only
     scopen voorkomt een lockout, B40.20)
  Target resources: All resources

  Conditions:
    Apparaatplatforms → Configureren: Ja → Windows

Access controls:
  Verlenen: Apparaat moet als compliant zijn gemarkeerd

Enable policy: Report-only (→ On at demo), gaf "Report-only: Success" bij een compliant mobile01-login
```

**Controlepunt Gate 1:**

- [ ] Alle 5 policies targeten **All resources** (geen app-targeting, geen resource-exclusions)
- [ ] Elke policy include de persona-groepen en exclude `2itcsc1a_admin1`
- [ ] MFA-registratie voltooid voor het testaccount
- [ ] Test aanmelden op mobile01: MFA-prompt verschijnt
- [ ] Entra ID → Protection → Sign-in logs → Conditional Access-tab toont `Resource: Microsoft Graph, Matched` en policies als "Success" / "Report-only: Success"

---

## [GEÏMPLEMENTEERD] Stap 2: Het Intune-device compliance policy aanmaken (Gate 2)

Gate 2 is **Intune device compliance**, niet NetBird-posture. Omdat de in-scope apparaten beheerd zijn (Entra-joined + Intune-enrolled), attesteert Intune de werkelijke apparaatstatus via de beheeragent (posture dat de eindgebruiker niet kan spoofen). Policy 5 (Stap 1) consumeert deze attestation op authenticatietijdstip.

Navigeer naar: `https://intune.microsoft.com → Devices → Compliance → Policies → Create policy → Windows 10 and later`

```
Naam: 2ITCSC1A-SASE-Windows-Compliance
Platform: Windows 10 and later

Compliance settings:
  Microsoft Defender Antimalware:                                  Require
  Defender Antivirus:                                             Require
  Defender Antispyware:                                           Require
  Defender Antimalware security intelligence up-to-date:          Require
  Real-time protection:                                          Require
  Firewall:                                                      Require
  Minimum OS-versie:                                             10.0.22000.0
  BitLocker / Secure Boot / Code integrity / TPM / encryptie:    Not configured

Acties bij non-compliance:
  Apparaat als non-compliant markeren: Onmiddellijk (0 dagen respijt)

Toewijzing:
  Included groep: 2ITCSC1A-Studenten (gebruikersgroep)
```

Drie effectieve controles blijven over na het schrappen van de encryptie/boot-instellingen: **OS-versie, antivirus, firewall** (B40.14; de rubric vraagt "device posture", niet encryptie).

> **Valkuil: een getargete gebruiker zonder Intune-licentie toont "Not applicable", niet "Non-compliant".** In Verslag40 rapporteerde de policy `Total 0 / Not applicable` totdat de teststudent een A5-licentie kreeg (die Intune Plan 1 omvat). `docent1`/`admin1` werden bewust ongelicenseerd gelaten, wat de reden is dat Policy 5 studenten-only is gescoped (B40.20).
>
> **Valkuil: een verlopen MDM-sessie blokkeert de evaluatie, en een reboot lost dit niet op.** Na het licentiëren bleven `Device status → Total 0` en `Last contacted` bevroren, zelfs na een reboot. De MDM-sessie was verlopen, dus het apparaat kon niet authenticeren op het policy-evaluatiekanaal. Een verse gebruikersaanmelding (geen reboot) herstelde de evaluatie; mobile01 rapporteerde toen **Compliant** (B40.20).
>
> **Valkuil: de Microsoft control plane moet SSL-Bump omzeilen.** Intune-apparaatregistratie en de compliant-device check mislukken als Squid Microsoft-endpoints bumpt. Zorg dat `*.microsoftonline.com`, `enterpriseregistration.windows.net`, `.microsoftazuread-sso.com` en `.live.com` op de Squid splice/no-bump-lijst staan (Runbook 03, B40.9/40.10/44.13).

**Controlepunt Gate 2:**

- [ ] `2ITCSC1A-SASE-Windows-Compliance` aangemaakt en toegewezen aan `2ITCSC1A-Studenten`
- [ ] Teststudent heeft een A5-licentie (Intune)
- [ ] Intune → Devices → mobile01 (`2ITCSC1A-MOB-1`) toont **Compliant**
- [ ] Een NetBird-login door die student toont `2ITCSC1A-SASE-Require-Compliant-Device → Report-only: Success` in de CA-aanmeldingstab

---

## [OPTIONEEL, NIET GEDEPLOYD] Stap 3: NetBird Posture Check aanmaken (defense-in-depth)

> **Niet gedeployd.** Met beheerde apparaten dekt Intune compliance (Stap 2) de device posture al via attestation. De NetBird-posture check hieronder is een *optionele* defense-in-depth-laag: onafhankelijke timing (tunnel-bouw) en mechanisme (client-side controle). Het werd **niet** gedeployd in de sandbox; de stappen blijven als ontwerpreferentie.

NetBird Dashboard → Access Control → Posture Checks → Create Posture Check

**Naam:** `SASE-PoC-Compliance`
**Beschrijving:** Hybride posture check: OS-versie, AV-proces, geo, clientversie

### Controle 1: OS-versie

```
Windows: minimale kernelversie 10.0.19041
```

> **Valkuil: Dit is de kernelversie, niet de marketingversie.** Windows 10 "21H1" vs kernel 10.0.19041; gebruik de kernelversie. 10.0.19041 is Windows 10 2004, de eerste versie met native WireGuard kernelmodule.

### Controle 2: NetBird-clientversie

```
Minimumversie: <vul uitvoer in van 'netbird version' op mobile01>
```

### Controle 3: Procescontrole (Antivirus)

```
Windows-pad: C:\Program Files\Windows Defender\MsMpEng.exe
macOS-pad:   /usr/libexec/syspolicyd  (XProtect)
Linux-pad:   /usr/sbin/clamd          (ClamAV-daemon)
```

> **Valkuil: Als geen pad is opgegeven voor een bepaald OS, blokkeert NetBird verbindingen van dat OS standaard.** iOS en Android hebben geen `process_check` beschikbaar; mobiele platforms worden geblokkeerd als de posture check een procescontrole bevat. Voor de PoC is dit acceptabel (mobile01 is Windows). Let op dat procescontroles te omzeilen zijn: een dummy-bestand op het verwachte pad voldoet aan de controle zonder dat de beveiligingssoftware daadwerkelijk draait.

### Controle 4: Geolocatiecontrole

```
Actie:   Toestaan
Landen:  België
```

Dit is defense-in-depth naast de CA-geoblokkering (verschillende GeoIP-databases compenseren elkaars fouten). Zie [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.nl.md).

---

## [OPTIONEEL, NIET GEDEPLOYD] Stap 4: Posture check koppelen aan ACL policies

```
NetBird Dashboard → Access Control → Policies
→ Selecteer de relevante policy (Personas-to-Core-Services)
→ Tab: Posture Checks
→ Browse Checks → selecteer "SASE-PoC-Compliance"
→ Add Posture Checks
→ Save Changes
```

Posture checks zijn per policy, niet globaal. Koppel aan elke policy die de persona-peers gebruiken.

**Controlepunt (optionele NetBird-posture):**

- [ ] Posture check "SASE-PoC-Compliance" aangemaakt met alle 4 controles
- [ ] Gekoppeld aan de Personas-to-Core-Services policy
- [ ] mobile01 verbindt: `netbird status` → Connected
- [ ] Dashboard → Peers → mobile01 → Posture Checks toont "Passed"

---

## Stap 5: Validatiescenario's

### Scenario 1: Positieve test, compliant device, juiste locatie [GEVALIDEERD]

> **Status:** Gevalideerd (Verslag40). 5 CA policies geïmplementeerd (3 Aan, 2 Report-only). MFA, blokkering van verouderde authenticatie en risicogebaseerde blokkering zijn Aan. Geo-Block en Require-Compliant-Device staan op Report-only tot demo; beide bewezen al "Report-only: Success".

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | `netbird down` op mobile01 | Tunnel gesloten |
| 2 | `netbird up` | Microsoft-inlogpagina opent |
| 3 | Inloggen + MFA | MFA-prompt, inloggen geslaagd |
| 4 | Wacht op tunnel | `netbird status` → Connected |
| 5 | `ping 100.70.154.79` | Antwoord ontvangen |
| 6 | Controleer CA sign-in log | `Resource: Microsoft Graph, Matched`; MFA → Success; Require-Compliant-Device → Report-only: Success; Geo-Block/Legacy-Auth/Risk-Block → Not Applied |

### Scenario 2: Negatieve test Gate 1, geoblokkering

> Geo-Block staat op Report-only tot demo, dus het sign-in log registreert wat er *zou* gebeuren (`Report-only: Failure`); een echte weigering treedt pas op zodra de policy op Aan wordt gezet.

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Verwijder België uit `2ITCSC1A-SASE-PoC-Allowed-Countries` | Bron-IP niet langer in toegestane locaties |
| 2 | `netbird down && netbird up` | Inlogpagina opent |
| 3 | Inloggen | Report-only: inloggen slaagt nog steeds (zou geweigerd worden indien Aan) |
| 4 | Sign-in log | `2ITCSC1A-SASE-PoC-Geo-Block` → "Report-only: Failure" (→ "Failure"/Block indien Aan) |
| 5 | **Herstel:** voeg België terug toe | Geo-Block keert terug naar "Not applied" |

### Scenario 3: Negatieve test Gate 2, OS-versie (Intune)

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Intune → `2ITCSC1A-SASE-Windows-Compliance` → Minimum OS-versie `10.0.99999.0` | Onmogelijk hoge versie |
| 2 | Forceer een sync op mobile01 (Company Portal / `Settings → Accounts → Access work → Sync`) | Apparaat opnieuw geëvalueerd |
| 3 | Intune → Devices → mobile01 | **Non-compliant** |
| 4 | `netbird down && netbird up`; controleer CA sign-in log | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" (→ Block indien Aan) |
| 5 | **Herstel:** stel Minimum OS terug op `10.0.22000.0` + sync | Apparaat keert terug naar Compliant |

### Scenario 4: Negatieve test Gate 2, antivirus / real-time protection (Intune)

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Schakel Defender real-time protection uit: `Set-MpPreference -DisableRealtimeMonitoring $true` (als beheerder) | RTP uit |
| 2 | Forceer een Intune-sync op mobile01 | Apparaat opnieuw geëvalueerd |
| 3 | Intune → Devices → mobile01 | **Non-compliant** (Real-time protection = Require) |
| 4 | `netbird up`; controleer CA sign-in log | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" |
| 5 | **Herstel:** `Set-MpPreference -DisableRealtimeMonitoring $false` + sync | Weer Compliant |

> Opmerking: Tamper Protection moet mogelijk eerst worden uitgeschakeld: Settings → Windows Security → Virus & Threat Protection → Tamper Protection: Off. Schakel direct na de test opnieuw in.

### Scenario 5: End-to-end, alle drie de gates

| Stap | Gate | Actie | Verwacht |
|------|------|-------|---------|
| 1 | n.v.t. | `netbird up` | Inlogpagina opent |
| 2 | Gate 1 | Inloggen + MFA | Token ontvangen; MFA → Success |
| 3 | Gate 2 | Intune compliance geattesteerd bij login (Policy 5) | Apparaat Compliant → "Report-only: Success" |
| 4 | Gate 3 | Surfen naar `https://google.com` | SSL Bump: certificaat van SASE-PoC-CA |
| 5 | Gate 3 | EICAR-testbestand downloaden | ClamAV blokkeert |
| 6 | Gate 3 | Surfen naar RPZ-geblokkeerd domein | DNS NXDOMAIN |
| 7 | n.v.t. | CA sign-in log | `Resource: Microsoft Graph, Matched`; gehandhaafde policies "Success" |
| 8 | n.v.t. | Intune → Devices → mobile01 | Compliant |

---

## Eerlijke beperkingen

| Beperking | Waarom | Status / mitigatie |
|-----------|--------|--------------------|
| Twee policies op Report-only | Geo-Block en Require-Compliant-Device staan bewust op Report-only tot demo | Op Aan zetten bij demo-voorbereiding; beide bewezen al "Report-only: Success" (B40.23) |
| Admins-persona onder geen enkele CA policy | `2itcsc1a_admin1` is het enige admins-lid en is de break-glass-uitsluiting op elke policy (B40.6) | Voeg een tweede admin-account toe om de admins-persona onder CA te brengen |
| docent1/admin1 niet gedekt door Policy 5 | Bewust ongelicenseerd gelaten; Policy 5 is studenten-only gescoped om een lockout te voorkomen (B40.20) | Licenseer docenten/admins voor Intune om Gate 2 naar die persona's uit te breiden |
| Geen Continuous Access Evaluation (CAE) | Gate 2 herevalueert bij authenticatie en op Intune's periodieke cyclus, niet continu per sessie | Schakel CAE in voor near-real-time intrekking |
| GeoIP niet 100% nauwkeurig | IP-geolocatiedatabases bevatten fouten | Alleen Geo-Block (Gate 1, CA); de optionele NetBird-geocontrole (andere DB) is niet gedeployd |
| C2-beaconing via WireGuard-tunnel | Een gecompromitteerd apparaat kan de WireGuard-tunnel (UDP 51820) als C2-beaconingkanaal gebruiken (buiten Squid's zichtbereik) | Endpoint Detection + Suricata (ziet WireGuard als versleuteld UDP) |
| NetBird-posture niet gedeployd | Intune-attestation dekt device posture niet-spoofbaar; de client-side `process_check` is spoofbaar (een dummy-binary op het pad voldoet) | Deploy NetBird-posture (Stappen 3-4) alleen als onbeheerde/BYOD-apparaten opnieuw in scope komen |

> **Dekkingsopmerking:** Met Intune-attestation gedeployd als Gate 2 is het grootste hiaat in het BYOD-tijdperk-plan (endpoint attestation die procesgebaseerd en spoofbaar was) gedicht: Intune rapporteert de werkelijke apparaatstatus niet-spoofbaar. De voornaamste resterende hiaten ten opzichte van commerciële SASE (Zscaler ZPA, Netskope Private Access) zijn Continuous Access Evaluation (CAE) en per-sessie continu posture in plaats van evaluatie bij authenticatie.

Zie [Concept: Zero Trust](../concepts/zero-trust.nl.md) voor de volledige analyse van het drie-gates-model.

---

## Gerelateerd

- [Component: NetBird](../components/netbird.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: CA + Posture hybride (Drie-gates-model)](../decisions/ca-posture-hybrid.nl.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.nl.md)
