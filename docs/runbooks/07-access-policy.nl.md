---
title: "Runbook: Toegangsbeleid"
tags: [runbook, entra-id, zero-trust, conditional-access, posture-check]
---

# Runbook: Toegangsbeleid

**Bron:** `raw/Verslag40.md` (implementatie, 2 juni 2026); `raw/Doc7_ZTNA_Context_Aware.md` + Addendum E.2 (ontwerpintentie)
**Node(s):** Entra ID (`aplab.be`-tenant) + Intune + NetBird Dashboard
**Vereisten:** Alle voorgaande runbooks afgerond (volledige SASE-stack operationeel)
**Status:** Geïmplementeerd (Verslag40) — Gate 1: 5 CA-beleidsregels (3 Aan, 2 Report-only); Gate 2: Intune-apparaatnaleving operationeel; Gate 3 volledig operationeel.

> **Gates 1 en 2 zijn geïmplementeerd (Verslag40).** Gate 1 = vijf Conditional Access-beleidsregels: MFA Required (Aan), Block Legacy Auth (Aan), Risk-Based Block (Aan), Geo-Block alleen België (Report-only), Require Compliant Device (Report-only → "Report-only: Success"). De twee Report-only-beleidsregels blijven zo tot de demo-voorbereiding (Sessie 11). Gate 2 = Intune-apparaatnaleving (`2ITCSC1A-SASE-Windows-Compliance`), attestatie-gebaseerd — dit is de uitgerolde apparaat-gate, **niet** NetBird-posturecontroles (die een optionele, niet-uitgerolde defense-in-depth-laag blijven). Gate 3 is operationeel en wordt gedekt door Runbooks 03-06.

---

## Vereistenchecklist

- [ ] Entra ID-tenant (`aplab.be`) toegankelijk
- [ ] A5-licentie actief (bevestigd 1 april 2026) — bevat Entra ID P1 + P2
- [ ] Rechten als Conditional Access-beheerder of Globale beheerder
- [ ] NetBird-app-registratie `2ITCSC1A-Netbird-Sandbox` (App ID `11803ee8-eb15-462c-a286-5415c17a29c6`) aanwezig in Entra ID — alleen voor NetBird-inloggen, **niet** een CA-doel (zie valkuil in Stap 1)
- [ ] NetBird-inloggen via Microsoft werkt (Runbook 02, Stap 8)
- [ ] Persona-beveiligingsgroepen aanwezig: `2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`
- [ ] Testaccounts (single-persona): een studenten-lid (bijv. `Student_1@aplab.be`, A5-gelicenseerd zodat Intune van toepassing is), een docenten-lid, en `2itcsc1a_admin1` (admins / break-glass)
- [ ] **MFA geregistreerd voor testaccount** (via `https://aka.ms/mfasetup`)
- [ ] mobile01 Entra-joined + Intune-enrolled (verschijnt in Intune als `2ITCSC1A-MOB-1`)
- [ ] NetBird-versie op mobile01 genoteerd: `netbird version`
- [ ] NetBird Dashboard toegankelijk: `https://netbird.sandbox.local`

> **Valkuil: MFA moet geregistreerd zijn VÓÓR het inschakelen van het CA MFA-beleid.** Niet-geregistreerde MFA geeft een "MFA vereist maar niet geconfigureerd"-blokkering bij de volgende aanmelding — inclusief de verificatiesessie zelf. Registreer MFA eerst, schakel dan het beleid in.

---

## [GEÏMPLEMENTEERD] Stap 1: Entra ID Conditional Access-beleidsregels aanmaken (Gate 1)

Navigeer naar: `https://entra.microsoft.com → Protection → Conditional Access → Policies → New policy`

> **Valkuil: target All resources, NIET de NetBird-app.** App-getargete CA-beleidsregels vuren nooit bij NetBird/Zitadel OIDC-aanmeldingen. CA matcht op de *resource* van het token — een OIDC-login vraagt Microsoft Graph (`00000003-0000-0000-c000-000000000000`) op, niet de NetBird-app-registratie — dus een beleid gescoped op "Apps selecteren → NetBird" toont **Not Applied** bij elke NetBird-login (Verslag40, B40.4). Alle vijf beleidsregels targeten daarom **All resources** en beperken de blast radius via **user-scoping** (persona-groepen als include) in plaats van app-scoping — de inverse van Addendum E's strategie.
>
> **Geen resource-exclusions.** Vanaf 15 juni 2026 handhaaft Microsoft All-resources-beleidsregels die resource-*exclusions* dragen óók bij OIDC-only-aanmeldingen; vóór die datum worden zulke beleidsregels niet gehandhaafd bij OIDC-only-aanmeldingen (een bypass). All-resources *zonder* exclusions wordt aan beide kanten van die datum consistent gehandhaafd, dus elk beleid gebruikt All resources zonder resource-exclusion (B40.5). Blast-radius-controle gebeurt uitsluitend via user-scoping: include de persona-groepen (`2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`), exclude `2itcsc1a_admin1` als break-glass op elk beleid. Omdat `2itcsc1a_admin1` het enige lid is van de admins-persona en overal is uitgesloten, valt de admins-persona feitelijk onder geen enkel CA-beleid (B40.6 — bewust geaccepteerd voor nu).

### Beleid 1 — 2ITCSC1A-SASE-PoC-MFA-Required

```
Naam: 2ITCSC1A-SASE-PoC-MFA-Required

Toewijzingen:
  Gebruikers: Include 2ITCSC1A-Studenten + 2ITCSC1A-Docenten + 2ITCSC1A-Admins
  Uitsluiten: 2itcsc1a_admin1 (break-glass)

  Doelbronnen: All resources (geen app-targeting, geen resource-exclusions)

  Voorwaarden:
    Client-apps → Configureren: Ja
      aangevinkt: Browser
      aangevinkt: Mobiele apps en desktopclients

Toegangscontroles:
  Verlenen: Meervoudige verificatie vereisen

Beleid inschakelen: Aan
```

### Beleid 2 — 2ITCSC1A-SASE-PoC-Geo-Block

Maak eerst een Benoemde locatie aan:

```
Protection → Conditional Access → Named locations → + Countries location
Naam: 2ITCSC1A-SASE-PoC-Allowed-Countries
aangevinkt: België
aangevinkt: Nederland (optioneel — voor pendelende studenten)
```

Maak dan het beleid aan:

```
Naam: 2ITCSC1A-SASE-PoC-Geo-Block

Toewijzingen:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Doelbronnen: All resources

  Voorwaarden:
    Locaties → Configureren: Ja
      Opnemen: Elke locatie
      Uitsluiten: 2ITCSC1A-SASE-PoC-Allowed-Countries

Toegangscontroles:
  Verlenen: Toegang blokkeren

Beleid inschakelen: Report-only (→ Aan bij Sessie 11)
```

### Beleid 3 — 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

```
Naam: 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

Toewijzingen:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Doelbronnen: All resources

  Voorwaarden:
    Client-apps → Configureren: Ja
      aangevinkt: Exchange ActiveSync-clients
      aangevinkt: Andere clients
      (NIET aanvinken: Browser of Mobiele apps)

Toegangscontroles:
  Verlenen: Toegang blokkeren

Beleid inschakelen: Aan
```

### Beleid 4 — 2ITCSC1A-SASE-PoC-Risk-Block (vereist A5/P2)

```
Naam: 2ITCSC1A-SASE-PoC-Risk-Block

Toewijzingen:
  Gebruikers: Include persona-groepen / Uitsluiten 2itcsc1a_admin1
  Doelbronnen: All resources

  Voorwaarden:
    Aanmeldingsrisico → Configureren: Ja
      aangevinkt: Hoog
      aangevinkt: Gemiddeld

Toegangscontroles:
  Verlenen: Meervoudige verificatie vereisen
  (Voor hoog risico: overweeg Toegang blokkeren)

Beleid inschakelen: Aan
```

### Beleid 5 — 2ITCSC1A-SASE-Require-Compliant-Device

Dit beleid verbindt Gate 1 met Gate 2: het vereist dat het apparaat door Intune als conform is gemarkeerd (Stap 2). Let op: deze naam heeft **geen** "PoC"-segment.

```
Naam: 2ITCSC1A-SASE-Require-Compliant-Device

Toewijzingen:
  Gebruikers: Include 2ITCSC1A-Studenten / Uitsluiten 2itcsc1a_admin1
    (studenten-only — docent1/admin1 zijn bewust ongelicenseerd, dus studenten-only
     scopen voorkomt een lockout — B40.20)
  Doelbronnen: All resources

  Voorwaarden:
    Apparaatplatforms → Configureren: Ja → Windows

Toegangscontroles:
  Verlenen: Apparaat moet als conform zijn gemarkeerd

Beleid inschakelen: Report-only (→ Aan bij Sessie 11) — gaf "Report-only: Success" bij een conforme mobile01-login
```

**Controlepunt Gate 1:**

- [ ] Alle 5 beleidsregels targeten **All resources** (geen app-targeting, geen resource-exclusions)
- [ ] Elk beleid include de persona-groepen en exclude `2itcsc1a_admin1`
- [ ] MFA-registratie voltooid voor het testaccount
- [ ] Test aanmelden op mobile01: MFA-prompt verschijnt
- [ ] Entra ID → Protection → Aanmeldingslogboeken → Conditional Access-tab toont `Resource: Microsoft Graph — Matched` en beleidsregels als "Success" / "Report-only: Success"

---

## [GEÏMPLEMENTEERD] Stap 2: Het Intune-apparaatnalevingsbeleid aanmaken (Gate 2)

Gate 2 is **Intune-apparaatnaleving**, niet NetBird-posture. Omdat de in-scope apparaten beheerd zijn (Entra-joined + Intune-enrolled), attesteert Intune de werkelijke apparaatstatus via de beheeragent — postuur dat de eindgebruiker niet kan spoofen. Beleid 5 (Stap 1) consumeert deze attestatie op authenticatietijdstip.

Navigeer naar: `https://intune.microsoft.com → Devices → Compliance → Policies → Create policy → Windows 10 and later`

```
Naam: 2ITCSC1A-SASE-Windows-Compliance
Platform: Windows 10 and later

Nalevingsinstellingen:
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

Drie effectieve controles blijven over na het schrappen van de encryptie/boot-instellingen: **OS-versie, antivirus, firewall** (B40.14 — de rubric vraagt "device posture", niet encryptie).

> **Valkuil: een getargete gebruiker zonder Intune-licentie toont "Not applicable", niet "Non-compliant".** In Verslag40 rapporteerde het beleid `Total 0 / Not applicable` totdat de teststudent een A5-licentie kreeg (die Intune Plan 1 omvat). `docent1`/`admin1` werden bewust ongelicenseerd gelaten, wat de reden is dat Beleid 5 studenten-only is gescoped (B40.20).
>
> **Valkuil: een verlopen MDM-sessie blokkeert de evaluatie, en een reboot lost dit niet op.** Na het licentiëren bleven `Device status → Total 0` en `Last contacted` bevroren, zelfs na een reboot — de MDM-sessie was verlopen, dus het apparaat kon niet authenticeren op het policy-evaluatiekanaal. Een verse gebruikersaanmelding (geen reboot) herstelde de evaluatie; mobile01 rapporteerde toen **Compliant** (B40.20).
>
> **Valkuil: de Microsoft control plane moet SSL-Bump omzeilen.** Intune-apparaatregistratie en de conform-apparaatcontrole mislukken als Squid Microsoft-endpoints bumpt. Zorg dat `*.microsoftonline.com` en `enterpriseregistration.windows.net` op de Squid splice/no-bump-lijst staan (Runbook 03, B40.9/40.10).

**Controlepunt Gate 2:**

- [ ] `2ITCSC1A-SASE-Windows-Compliance` aangemaakt en toegewezen aan `2ITCSC1A-Studenten`
- [ ] Teststudent heeft een A5-licentie (Intune)
- [ ] Intune → Devices → mobile01 (`2ITCSC1A-MOB-1`) toont **Compliant**
- [ ] Een NetBird-login door die student toont `2ITCSC1A-SASE-Require-Compliant-Device → Report-only: Success` in de CA-aanmeldingstab

---

## [OPTIONEEL — NIET UITGEROLD] Stap 3: NetBird Posture Check aanmaken (defense-in-depth)

> **Niet uitgerold.** Met beheerde apparaten dekt Intune-naleving (Stap 2) het apparaatpostuur al via attestatie. De NetBird-posturecontrole hieronder is een *optionele* defense-in-depth-laag — onafhankelijke timing (tunnel-bouw) en mechanisme (client-side controle). Het werd **niet** uitgerold in de sandbox; de stappen blijven als ontwerpreferentie.

NetBird Dashboard → Access Control → Posture Checks → Create Posture Check

**Naam:** `SASE-PoC-Compliance`
**Beschrijving:** Hybride posturecontrole — OS-versie, AV-proces, geo, clientversie

### Controle 1 — OS-versie

```
Windows: minimale kernelversie 10.0.19041
```

> **Valkuil: Dit is de kernelversie, niet de marketingversie.** Windows 10 "21H1" vs kernel 10.0.19041 — gebruik de kernelversie. 10.0.19041 is Windows 10 2004, de eerste versie met native WireGuard kernelmodule.

### Controle 2 — NetBird-clientversie

```
Minimumversie: <vul uitvoer in van 'netbird version' op mobile01>
```

### Controle 3 — Procescontrole (Antivirus)

```
Windows-pad: C:\Program Files\Windows Defender\MsMpEng.exe
macOS-pad:   /usr/libexec/syspolicyd  (XProtect)
Linux-pad:   /usr/sbin/clamd          (ClamAV-daemon)
```

> **Valkuil: Als geen pad is opgegeven voor een bepaald OS, blokkeert NetBird verbindingen van dat OS standaard.** iOS en Android hebben geen `process_check` beschikbaar — mobiele platforms worden geblokkeerd als de posturecontrole een procescontrole bevat. Voor de PoC is dit acceptabel (mobile01 is Windows). Let op dat procescontroles te omzeilen zijn — een dummy-bestand op het verwachte pad voldoet aan de controle zonder dat de beveiligingssoftware daadwerkelijk draait.

### Controle 4 — Geolocatiecontrole

```
Actie:   Toestaan
Landen:  België
```

Dit is defense-in-depth naast de CA-geoblokkering (verschillende GeoIP-databases compenseren elkaars fouten). Zie [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.nl.md).

---

## [OPTIONEEL — NIET UITGEROLD] Stap 4: Posturecontrole koppelen aan ACL-beleidsregels

```
NetBird Dashboard → Access Control → Policies
→ Selecteer het relevante beleid (Personas-to-Core-Services)
→ Tab: Posture Checks
→ Browse Checks → selecteer "SASE-PoC-Compliance"
→ Add Posture Checks
→ Save Changes
```

Posturecontroles zijn per beleid, niet globaal. Koppel aan elk beleid dat de persona-peers gebruiken.

**Controlepunt (optionele NetBird-posture):**

- [ ] Posturecontrole "SASE-PoC-Compliance" aangemaakt met alle 4 controles
- [ ] Gekoppeld aan het Personas-to-Core-Services-beleid
- [ ] mobile01 verbindt: `netbird status` → Connected
- [ ] Dashboard → Peers → mobile01 → Posture Checks toont "Passed"

---

## Stap 5: Validatiescenario's

### Scenario 1 — Positieve test: conform apparaat, juiste locatie [GEVALIDEERD]

> **Status:** Gevalideerd (Verslag40). 5 CA-beleidsregels geïmplementeerd (3 Aan, 2 Report-only). MFA, blokkering van verouderde authenticatie en risicogebaseerde blokkering zijn Aan. Geo-Block en Require-Compliant-Device staan op Report-only tot Sessie 11 — beide bewezen al "Report-only: Success".

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | `netbird down` op mobile01 | Tunnel gesloten |
| 2 | `netbird up` | Microsoft-inlogpagina opent |
| 3 | Inloggen + MFA | MFA-prompt, inloggen geslaagd |
| 4 | Wacht op tunnel | `netbird status` → Connected |
| 5 | `ping 100.70.154.79` | Antwoord ontvangen |
| 6 | Controleer CA-aanmeldingslog | `Resource: Microsoft Graph — Matched`; MFA → Success; Require-Compliant-Device → Report-only: Success; Geo-Block/Legacy-Auth/Risk-Block → Not Applied |

### Scenario 2 — Negatieve test Gate 1: geoblokkering

> Geo-Block staat op Report-only tot Sessie 11, dus het aanmeldingslog registreert wat er *zou* gebeuren (`Report-only: Failure`); een echte weigering treedt pas op zodra het beleid op Aan wordt gezet.

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Verwijder België uit `2ITCSC1A-SASE-PoC-Allowed-Countries` | Bron-IP niet langer in toegestane locaties |
| 2 | `netbird down && netbird up` | Inlogpagina opent |
| 3 | Inloggen | Report-only: inloggen slaagt nog steeds (zou geweigerd worden indien Aan) |
| 4 | Aanmeldingslog | `2ITCSC1A-SASE-PoC-Geo-Block` → "Report-only: Failure" (→ "Failure"/Block indien Aan) |
| 5 | **Herstel:** voeg België terug toe | Geo-Block keert terug naar "Not applied" |

### Scenario 3 — Negatieve test Gate 2: OS-versie (Intune)

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Intune → `2ITCSC1A-SASE-Windows-Compliance` → Minimum OS-versie `10.0.99999.0` | Onmogelijk hoge versie |
| 2 | Forceer een sync op mobile01 (Company Portal / `Instellingen → Accounts → Toegang tot werk → Sync`) | Apparaat opnieuw geëvalueerd |
| 3 | Intune → Devices → mobile01 | **Non-compliant** |
| 4 | `netbird down && netbird up`; controleer CA-aanmeldingslog | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" (→ Block indien Aan) |
| 5 | **Herstel:** stel Minimum OS terug op `10.0.22000.0` + sync | Apparaat keert terug naar Compliant |

### Scenario 4 — Negatieve test Gate 2: antivirus / real-time protection (Intune)

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Schakel Defender real-time protection uit: `Set-MpPreference -DisableRealtimeMonitoring $true` (als beheerder) | RTP uit |
| 2 | Forceer een Intune-sync op mobile01 | Apparaat opnieuw geëvalueerd |
| 3 | Intune → Devices → mobile01 | **Non-compliant** (Real-time protection = Require) |
| 4 | `netbird up`; controleer CA-aanmeldingslog | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" |
| 5 | **Herstel:** `Set-MpPreference -DisableRealtimeMonitoring $false` + sync | Weer Compliant |

> Opmerking: Manipulatiebeveiliging moet mogelijk eerst worden uitgeschakeld: Instellingen → Windows-beveiliging → Virus- en bedreigingsbeveiliging → Manipulatiebeveiliging: Uit. Schakel direct na de test opnieuw in.

### Scenario 5 — End-to-end: alle drie de gates

| Stap | Gate | Actie | Verwacht |
|------|------|-------|---------|
| 1 | — | `netbird up` | Inlogpagina opent |
| 2 | Gate 1 | Inloggen + MFA | Token ontvangen; MFA → Success |
| 3 | Gate 2 | Intune-naleving geattesteerd bij login (Beleid 5) | Apparaat Compliant → "Report-only: Success" |
| 4 | Gate 3 | Surfen naar `https://google.com` | SSL Bump: certificaat van SASE-PoC-CA |
| 5 | Gate 3 | EICAR-testbestand downloaden | ClamAV blokkeert |
| 6 | Gate 3 | Surfen naar RPZ-geblokkeerd domein | DNS NXDOMAIN |
| 7 | — | CA-aanmeldingslog | `Resource: Microsoft Graph — Matched`; gehandhaafde beleidsregels "Success" |
| 8 | — | Intune → Devices → mobile01 | Compliant |

---

## Eerlijke beperkingen

| Beperking | Waarom | Status / mitigatie |
|-----------|--------|--------------------|
| Twee beleidsregels op Report-only | Geo-Block en Require-Compliant-Device staan bewust op Report-only tot Sessie 11 | Op Aan zetten bij demo-voorbereiding; beide bewezen al "Report-only: Success" (B40.23) |
| Admins-persona onder geen enkel CA-beleid | `2itcsc1a_admin1` is het enige admins-lid en is de break-glass-uitsluiting op elk beleid (B40.6) | Voeg een tweede admin-account toe om de admins-persona onder CA te brengen |
| docent1/admin1 niet gedekt door Beleid 5 | Bewust ongelicenseerd gelaten; Beleid 5 is studenten-only gescoped om een lockout te voorkomen (B40.20) | Licenseer docenten/admins voor Intune om Gate 2 naar die persona's uit te breiden |
| Geen Continuous Access Evaluation (CAE) | Gate 2 herevalueert bij authenticatie en op Intune's periodieke cyclus, niet continu per sessie | Schakel CAE in voor near-real-time intrekking |
| GeoIP niet 100% nauwkeurig | IP-geolocatiedatabases bevatten fouten | Alleen Geo-Block (Gate 1, CA); de optionele NetBird-geocontrole (andere DB) is niet uitgerold |
| C2-beaconing via WireGuard-tunnel | Een gecompromitteerd apparaat kan de WireGuard-tunnel (UDP 51820) als C2-beaconingkanaal gebruiken — buiten Squid's zichtbereik | Eindpuntdetectie + Suricata (ziet WireGuard als versleuteld UDP) |
| NetBird-posture niet uitgerold | Intune-attestatie dekt apparaatpostuur niet-spoofbaar; de client-side `process_check` is spoofbaar (een dummy-binary op het pad voldoet) | Rol NetBird-posture (Stappen 3-4) alleen uit als onbeheerde/BYOD-apparaten opnieuw in scope komen |

> **Dekkingsopmerking:** Met Intune-attestatie uitgerold als Gate 2 is het grootste hiaat in het BYOD-tijdperk-plan — eindpuntattestatie die procesgebaseerd en spoofbaar was — gedicht: Intune rapporteert de werkelijke apparaatstatus niet-spoofbaar. De voornaamste resterende hiaten ten opzichte van commerciële SASE (Zscaler ZPA, Netskope Private Access) zijn Continuous Access Evaluation (CAE) en per-sessie continu postuur in plaats van evaluatie bij authenticatie.

Zie [Concept: Zero Trust](../concepts/zero-trust.nl.md) voor de volledige analyse van het drie-gates-model.

---

## Gerelateerd

- [Component: NetBird](../components/netbird.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: CA + Posture hybride (Drie-gates-model)](../decisions/ca-posture-hybrid.nl.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.nl.md)
