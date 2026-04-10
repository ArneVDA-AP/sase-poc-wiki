---
title: "Runbook: Toegangsbeleid"
tags: [runbook, entra-id, zero-trust, conditional-access, posture-check]
---

# Runbook: Toegangsbeleid

**Bron:** `raw/Doc7_ZTNA_Context_Aware.md`
**Node(s):** Entra ID (`aplab.be`-tenant) + NetBird Dashboard
**Vereisten:** Alle voorgaande runbooks afgerond (volledige SASE-stack operationeel)
**Status:** Gepland — architectuur gevalideerd (Addendum E), Gates 1 & 2 nog niet geïmplementeerd. Gate 3 volledig operationeel.

> **Deze runbook beschrijft geplande stappen.** Gates 1 en 2 zijn architectuurontwerp en implementatieplan, gepland na de tussentijdse evaluatie (20 april 2026). Gate 3 is operationeel en wordt gedekt door Runbooks 03-06. Statuslabels `[GEPLAND]` en `[OPERATIONEEL]` zijn absoluut.

---

## Vereistenchecklist

- [ ] Entra ID-tenant (`aplab.be`) toegankelijk
- [ ] A5-licentie actief (bevestigd 1 april 2026) — bevat Entra ID P1 + P2
- [ ] Rechten als Conditional Access-beheerder of Globale beheerder
- [ ] NetBird-app-registratie (`cebe0d74-...`) aanwezig in Entra ID
- [ ] NetBird-inloggen via Microsoft werkt (Runbook 02, Stap 8)
- [ ] Testaccount beschikbaar: `arne.vda.2itcsc1a@aplab.be`
- [ ] **MFA geregistreerd voor testaccount** (via `https://aka.ms/mfasetup`)
- [ ] NetBird-versie op mobile01 genoteerd: `netbird version`
- [ ] NetBird Dashboard toegankelijk: `https://netbird.sandbox.local`

> **Valkuil: MFA moet geregistreerd zijn VÓÓR het inschakelen van het CA MFA-beleid.** Niet-geregistreerde MFA geeft een "MFA vereist maar niet geconfigureerd"-blokkering bij de volgende aanmelding — inclusief de verificatiesessie zelf. Registreer MFA eerst, schakel dan het beleid in.

---

## [GEPLAND] Stap 1: Entra ID Conditional Access-beleidsregels aanmaken

Navigeer naar: `https://entra.microsoft.com → Protection → Conditional Access → Policies → New policy`

### Beleid 1 — SASE-PoC-MFA-Required

```
Naam: SASE-PoC-MFA-Required

Toewijzingen:
  Gebruikers: Alle gebruikers
  Uitsluiten: (optioneel) break-glass beheerdersaccount

  Doelbronnen:
    Cloud-apps → Apps selecteren → "NetBird SASE PoC"
    (Client-ID: cebe0d74-be9f-49ac-9f35-65f11586c1bb)

  Voorwaarden:
    Client-apps → Configureren: Ja
      aangevinkt: Browser
      aangevinkt: Mobiele apps en desktopclients

Toegangscontroles:
  Verlenen: Meervoudige verificatie vereisen

Beleid inschakelen: Alleen rapporteren (eerste test) → dan Aan
```

### Beleid 2 — SASE-PoC-Geo-Block

Maak eerst een Benoemde locatie aan:

```
Protection → Conditional Access → Named locations → + Countries location
Naam: SASE-PoC-Allowed-Countries
aangevinkt: België
aangevinkt: Nederland (optioneel — voor pendeldende studenten)
```

Maak dan het beleid aan:

```
Naam: SASE-PoC-Geo-Block

Toewijzingen:
  Gebruikers: Alle gebruikers
  Doelbronnen: Apps selecteren → "NetBird SASE PoC"

  Voorwaarden:
    Locaties → Configureren: Ja
      Opnemen: Elke locatie
      Uitsluiten: SASE-PoC-Allowed-Countries

Toegangscontroles:
  Verlenen: Toegang blokkeren

Beleid inschakelen: Aan
```

### Beleid 3 — SASE-PoC-Block-Legacy-Auth

```
Naam: SASE-PoC-Block-Legacy-Auth

Toewijzingen:
  Gebruikers: Alle gebruikers
  Doelbronnen: Apps selecteren → "NetBird SASE PoC"

  Voorwaarden:
    Client-apps → Configureren: Ja
      aangevinkt: Exchange ActiveSync-clients
      aangevinkt: Andere clients
      (NIET aanvinken: Browser of Mobiele apps)

Toegangscontroles:
  Verlenen: Toegang blokkeren

Beleid inschakelen: Aan
```

### Beleid 4 — SASE-PoC-Risk-Block (vereist A5/P2)

```
Naam: SASE-PoC-Risk-Block

Toewijzingen:
  Gebruikers: Alle gebruikers
  Doelbronnen: Apps selecteren → "NetBird SASE PoC"

  Voorwaarden:
    Aanmeldingsrisico → Configureren: Ja
      aangevinkt: Hoog
      aangevinkt: Gemiddeld

Toegangscontroles:
  Verlenen: Meervoudige verificatie vereisen
  (Voor hoog risico: overweeg Toegang blokkeren)

Beleid inschakelen: Aan
```

**Controlepunt Gate 1:**

- [ ] Alle 4 beleidsregels richten zich op de NetBird-app-registratie
- [ ] MFA-registratie voltooid voor testaccount
- [ ] Test aanmelden op mobile01: MFA-prompt verschijnt
- [ ] Entra ID → Protection → Aanmeldingslogboeken → Conditional Access-tab: beleidsregels tonen "Geslaagd"

---

## [GEPLAND] Stap 2: NetBird Posture Check aanmaken

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

## [GEPLAND] Stap 3: Posturecontrole koppelen aan ACL-beleidsregels

```
NetBird Dashboard → Access Control → Policies
→ Selecteer elk relevant beleid (Mobile-to-Services, Datacenter Access)
→ Tab: Posture Checks
→ Browse Checks → selecteer "SASE-PoC-Compliance"
→ Add Posture Checks
→ Save Changes
```

Posturecontroles zijn per beleid, niet globaal. Koppel aan elk beleid dat BYOD-clients gebruiken.

**Controlepunt Gate 2:**

- [ ] Posturecontrole "SASE-PoC-Compliance" aangemaakt met alle 4 controles
- [ ] Gekoppeld aan Mobile-to-Services en Datacenter Access-beleidsregels
- [ ] mobile01 verbindt: `netbird status` → Connected
- [ ] Dashboard → Peers → mobile01 → Posture Checks toont "Passed"

---

## [GEPLAND] Stap 4: Validatiescenario's

### Scenario 1 — Positieve test: conform apparaat, juiste locatie

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | `netbird down` op mobile01 | Tunnel gesloten |
| 2 | `netbird up` | Microsoft-inlogpagina opent |
| 3 | Inloggen + MFA | MFA-prompt, inloggen geslaagd |
| 4 | Wacht op tunnel | `netbird status` → Connected |
| 5 | `ping 100.70.154.79` | Antwoord ontvangen |
| 6 | Controleer aanmeldingslog | CA-beleidsregels: "Geslaagd" |

### Scenario 2 — Negatieve test Gate 1: geoblokkering

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Verwijder België uit SASE-PoC-Allowed-Countries | Beleid blokkeert nu school-IP |
| 2 | `netbird down && netbird up` | Inlogpagina opent |
| 3 | Inloggen | "Toegang geweigerd" — CA blokkeert |
| 4 | `netbird status` | Niet verbonden |
| 5 | Aanmeldingslog | SASE-PoC-Geo-Block → "Mislukt" |
| 6 | **Herstel:** voeg België terug toe | Volgende inlogpoging geslaagd |

### Scenario 3 — Negatieve test Gate 2: OS-versie

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Wijzig posturecontrole: Windows min kernel `10.0.99999` | Onmogelijk hoge versie |
| 2 | `netbird down && netbird up` | Gate 1 slaagt (inloggen OK) |
| 3 | `netbird status` | Geverifieerd maar routes niet beschikbaar |
| 4 | **Herstel:** stel kernel terug in op `10.0.19041` | Verbinding hersteld |

### Scenario 4 — Negatieve test Gate 2: procescontrole

| Stap | Actie | Verwacht |
|------|-------|---------|
| 1 | Stop Defender: `Set-MpPreference -DisableRealtimeMonitoring $true` (als beheerder) | MsMpEng.exe stopt |
| 2 | `netbird down && netbird up` | Gate 1 slaagt |
| 3 | `netbird status` | Routes niet beschikbaar |
| 4 | **Herstel:** `Set-MpPreference -DisableRealtimeMonitoring $false` | AV herstart, posture slaagt |

> Opmerking: Manipulatiebeveiliging moet mogelijk eerst worden uitgeschakeld: Instellingen → Windows-beveiliging → Virus- en bedreigingsbeveiliging → Manipulatiebeveiliging: Uit. Schakel direct na de test opnieuw in.

### Scenario 5 — End-to-end: alle drie de gates

| Stap | Gate | Actie | Verwacht |
|------|------|-------|---------|
| 1 | — | `netbird up` | Inlogpagina opent |
| 2 | Gate 1 | Inloggen + MFA | Token ontvangen |
| 3 | Gate 2 | NetBird evalueert posture | OS OK, AV draait, geo OK → tunnel actief |
| 4 | Gate 3 | Surfen naar `https://google.com` | SSL Bump: certificaat van SASE-PoC-CA |
| 5 | Gate 3 | EICAR-testbestand downloaden | ClamAV blokkeert |
| 6 | Gate 3 | Surfen naar RPZ-geblokkeerd domein | DNS NXDOMAIN |
| 7 | — | Aanmeldingslog | CA-beleidsregels: "Geslaagd" |
| 8 | — | Dashboard → peer-info | Posturecontrole: "Passed" |

---

## Eerlijke beperkingen

| Beperking | Waarom | Productieoplossing |
|-----------|--------|--------------------|
| Procescontrole is te omzeilen | Binair op juist pad ≠ werkende AV | Defender for Endpoint-attestatie (TPM) |
| Geen schijfversleutelingscontrole | NetBird process_check kan dit niet controleren | Intune-nalevingsbeleid (vereist MDM) |
| Geen AV-definitieversiecontrole | Procescontrole verifieert alleen dat het proces draait | Intune/Defender health attestation |
| CA-apparaatnaleving niet beschikbaar | Onbeheerde BYOD, geen Intune-inschrijving | Intune-inschrijving (onrealistisch voor 4000 BYOD) |
| Posture evalueert bij tunnelinstelling, niet continu | Geen realtime sessieevaluatie | NetBird periodieke herevaluatie; CAE voor productie |
| GeoIP niet 100% nauwkeurig | IP-geolocatiedatabases bevatten fouten | Gemitigeerd door combinatie van CA-geoblokkering (Gate 1) met NetBird geocontrole (Gate 2) — verschillende GeoIP-databases |
| C2-beaconing via WireGuard-tunnel | Een gecompromitteerd apparaat kan de WireGuard-tunnel (poort 51820) als C2-beaconingkanaal gebruiken — buiten Squid's zichtbereik | Gate 2 (posture) + eindpuntdetectie; Suricata ziet WireGuard als versleuteld UDP |

> **Dekkingsschatting:** De PoC implementeert ~60-70% van de contextbewuste toegangscontrole die commerciële SASE-oplossingen (Zscaler ZPA, Netskope Private Access) bieden. De voornaamste hiaten zijn eindpuntattestatie (TPM-gebaseerd, niet procesgebaseerd) en Continuous Access Evaluation (CAE).

Zie [Concept: Zero Trust](../concepts/zero-trust.nl.md) voor de volledige analyse van het drie-gates-model.

---

## Gerelateerd

- [Component: NetBird](../components/netbird.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: CA + Posture hybride (Drie-gates-model)](../decisions/ca-posture-hybrid.nl.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.nl.md)
