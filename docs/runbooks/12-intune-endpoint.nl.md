---
title: "Runbook: Intune Endpoint"
tags: [runbook, intune, mdm, firewall, identity]
---

# Runbook: Intune Endpoint

**Bron:** `Verslag41.md` + Route-Remediation-addendum (3 juni 2026); `Verslag44.md` (SITE01-enrollment)
**Node(s):** Entra ID / Intune (`aplab.be`-tenant) → managed Windows-endpoints; pop01 Squid (verificatiedoel)
**Vereisten:** Volledige SASE-stack operationeel (Runbooks 01–07); een managed Windows-device dat Entra-joined + Intune-enrolled is
**Status:** Geïmplementeerd (Verslag41). Vijf profielen gebouwd en geverifieerd op `2ITCSC1A-MOB-1`; tweede device `2ITCSC1A-SITE01` enrolled (Verslag44).

> Deze runbook is het **deployment-spoor** voor de device-zijde van de SASE-posture: het pusht het certificaat, de afgedwongen proxy, de firewall, de QoS en de split-tunnel-routes naar managed Windows-endpoints via Intune. De configuratiedetails en het waarom achter elk profiel staan in [Component: Intune Endpoint Enforcement](../components/intune-endpoint-enforcement.nl.md). Deze pagina geeft de stappen en de verificatie dat elk stuk daadwerkelijk geland is. Eén regel loopt door het hele spoor: **de device-store is de grondwaarheid, niet het Intune-statusveld per profiel** (een profiel kan "Succeeded" tonen terwijl er niets geland is).

---

## Vereistenchecklist

- [ ] Entra ID-tenant (`aplab.be`) toegankelijk met Intune Administrator-rechten
- [ ] Doel-device Entra-joined + Intune-enrolled (geverifieerd via `dsregcmd /status` → `AzureAdJoined: YES`, `Managed by MDM`)
- [ ] Het gelicenseerde enrollment-account is de primary user van het device (`student_1`, A5 → Intune Plan 1); het persona-account op de overlay mag anders zijn, want device-config-profielen targeten de device group op `displayName`, niet de user
- [ ] Lokale admin `.\saseuser` klaargezet als recovery-pad **vóór de eerste firewall-push** (zie Stap 1)
- [ ] Squid-overlay-listener actief op pop01 (`100.70.154.79:3128`) zodat de afgedwongen PAC en de bump te verifiëren zijn
- [ ] Microsoft-control-plane-endpoints op de Squid-splice/no-bump-lijst (Runbook 03) zodat de MDM-check-in en de registratie niet gebumpt worden
- [ ] "Remediations" geactiveerd in Intune (lost de eenmalige licensing-data-prerequisite op die **Create** grijs maakt)

> **Valkuil: zet `.\saseuser` klaar vóór de firewall-push.** Een firewall-reconciliatiebug (IT1214934) kan een bestaande outbound-regel in "block any outbound" veranderen en alle internet afsnijden zonder controle op afstand. Herstel is een lokale login plus het opschonen van de `FirewallPolicy\Mdm\FirewallRules`-registry. Kaal `saseuser` faalt bij de UAC-prompt (het wordt als Entra-account gelezen); de lokale-account-prefix `.\saseuser` werkt. Bevestig de recovery-login *vóór* Stap 4.

---

## Stap 1: De dynamische device group aanmaken

Config gaat naar een **device** group, niet naar een user group: een Proxy- of Firewall-CSP op een klasgenoot-device dat niet op de overlay zit zou hun internet breken.

Navigeer naar: `https://intune.microsoft.com → Groups → New group → Security → Dynamic Device`

```
Name: 2ITcsc1A-SASE-Devices
Membership rule:  device.displayName -startsWith "2ITcsc1A-"
```

De regel leest de **Entra**-`displayName`, niet de in Intune getoonde naam. De casing in de regel (`2ITcsc1A-`) wijkt af van de device-namen in hoofdletters (`2ITCSC1A-MOB-1`, `2ITCSC1A-SITE01`). Dat is onschadelijk, want string-operaties in dynamische regels zijn hoofdletterongevoelig (enkel property-namen zijn hoofdlettergevoelig).

**Verifieer:** Group → tab **Validate rules** slaagt voor de device-naam; Group → Members toont het doel-device.

---

## Stap 2: Het trusted-cert-profiel pushen (`2ITCSC1A-SASE-PoC-CA-Root`)

Installeert de SASE-PoC-CA-root zodat de SSL-Bump-certificaatketen van Squid vertrouwd wordt en gebumpte HTTPS valideert in plaats van certificaatfouten te geven.

Op pop01 (OPNsense): `System → Trust → Authorities`, exporteer het **publieke** `.crt` naast `SASE-PoC-CA` (geen `.p12` / private key).

Navigeer naar: `Intune → Devices → Configuration → Windows 10 and later → Templates → Trusted certificate`

```
Name: 2ITCSC1A-SASE-PoC-CA-Root
Certificate file: publieke SASE-PoC-CA .crt
Destination store: Computer certificate store - Root
Assignment: 2ITcsc1A-SASE-Devices
```

**Verifieer in de device-store, niet in de Intune-status:**

```powershell
certutil -store Root | findstr /i SASE
# Issuer/Subject: O=SASE PoC ; Thumbprint 55EF83FE8BE080B7DF9AE92E9E55CFCEF3AC4537
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -match 'SASE PoC'
```

> **Valkuil: CSP-conflict in de gedeelde tenant.** In de gedeelde `aplab.be`-tenant blokkeert een los-gescopet Trusted-Root-profiel van een andere groep, gericht op dezelfde `Windows81TrustedRootCertificate`-CSP-setting, stilletjes het jouwe. Beide profielen tonen "Succeeded" terwijl maar één certificaat landt. Als `findstr SASE` niets teruggeeft maar de CA van een andere groep wel aanwezig is, scope het conflicterende profiel zo dat het device erbuiten valt; na de sync landt de SASE-PoC-CA. De device-store is de enige grondwaarheid.

---

## Stap 3: Het afgedwongen-PAC-proxy-profiel pushen (`2ITCSC1A-SASE-Proxy-PAC`)

Dwingt browserverkeer door de pop01 Squid-SWG, zodat de proxy op het device afgedwongen wordt en geen instelling is die de gebruiker kan wissen. De Settings Catalog surfacet NetworkProxy niet, dus een Custom OMA-URI-profiel stuurt de NetworkProxy-CSP rechtstreeks aan.

Navigeer naar: `Intune → Devices → Configuration → Windows 10 and later → Templates → Custom → Add OMA-URI Settings`

```
./Vendor/MSFT/NetworkProxy/ProxySettingsPerUser   Integer  0
./Vendor/MSFT/NetworkProxy/AutoDetect             Integer  0
./Vendor/MSFT/NetworkProxy/SetupScriptUrl         String   http://wpad.sandbox.local/wpad.dat
```

`ProxySettingsPerUser=0` maakt het systeembreed (alle WinINET-apps), `AutoDetect=0` forceert de expliciete PAC. Zet **geen** `ProxyServer`-node: een PAC-URL gecombineerd met een niet-lege manual proxy server is een bekende Intune-bug die het profiel breekt. Naam `2ITCSC1A-SASE-Proxy-PAC`, assign aan `2ITcsc1A-SASE-Devices`. **Reboot** om de cert-/proxy-cache te wissen nadat het profiel geland is.

**Verifieer dat het verkeer door Squid gaat:** surf naar `https://google.com`. De certificaat-issuer is `O=SASE PoC` (gebumpt, het device vertrouwt de gepushte CA). Volg op pop01 het Squid-access-log: het overlay-bron-IP van het device verschijnt op de `TCP_MISS`/`TCP_TUNNEL`-regels (geen collaps naar een gateway-IP), wat bevestigt dat het verkeer via de proxy loopt.

> **Opmerking:** Entra-enrollment zet de handmatig geconfigureerde PAC uit, dus HTTPS gaat Direct (een "witte pagina") totdat dit profiel landt. Dat is precies waar het profiel voor dient, geen bug. De WinHTTP-SYSTEM-proxy blijft Direct by design voor de MDM-check-in.

---

## Stap 4: De firewall-block-set pushen (`2ITCSC1A-SASE-FW-BypassBlock`)

Sluit de twee bypass-vectoren op protocolniveau (QUIC en DoT) die anders verkeer rond de SWG en de gefilterde resolver zouden sturen. Bevestig de `.\saseuser`-recovery-login (vereiste) vóór deze push.

Navigeer naar: `Intune → Endpoint security → Firewall → Create policy → Windows → Microsoft Defender Firewall Rules`

```
Name: 2ITCSC1A-SASE-FW-BypassBlock

Rule: Block-QUIC-Outbound-UDP443
  Direction: Out | Protocol: UDP (17) | Remote ports: 443 | Action: Blocked
Rule: Block-DoT-Outbound-TCP853
  Direction: Out | Protocol: TCP (6)  | Remote ports: 853 | Action: Blocked

Assignment: 2ITcsc1A-SASE-Devices
```

Dit is bewust een **gerichte block-set, GEEN default-deny**. Een volledige default-deny outbound brickt het MDM-check-in-pad (het WinHTTP-SYSTEM-kanaal is een grote CDN-gebackte FQDN-set die IP-gebaseerde regels niet schoon kunnen allowlisten): de IT1214934-footgun. Het schema vereist dat het TCP/UDP-protocol gezet is op elke regel met poort-ranges, anders faalt de regel met "the parameter is incorrect".

**Verifieer dat er niets legitiems breekt:**

- `netbird status` → Connected (de UDP/443-block raakt de WireGuard-tunnel op 51820 niet)
- `https://google.com` laadt, issuer `O=SASE PoC` (browsers vallen van QUIC terug op TCP/443, Squid bumpt het)
- Intune → Devices → device → Sync → **successful** (het WinHTTP-SYSTEM-pad blijft open)

---

## Stap 5: Het DSCP/QoS-profiel pushen (`2ITCSC1A-SASE-QoS-Teams`)

Draagt de Teams-verkeersmarkeringen die de VyOS-SASE-gateway shapet. De NetworkQoSPolicy-CSP is officieel ondersteund op Intune-managed + Entra-joined devices (precies de staat van het device).

Navigeer naar: `Intune → Devices → Configuration → Windows 10 and later → Templates → Custom → Add OMA-URI Settings` (drie nodes per stream, prefix `./Device/Vendor/MSFT/NetworkQoSPolicy/`):

| Stream | `SourcePortMatchCondition` | `DSCPAction` | `AppPathNameMatchCondition` |
|--------|----------------------------|--------------|-----------------------------|
| TeamsAudio | `50000-50019` | `46` | `ms-teams.exe` |
| TeamsVideo | `50020-50039` | `34` | `ms-teams.exe` |
| TeamsScreenshare | `50040-50059` | `18` | `ms-teams.exe` |

Naam `2ITCSC1A-SASE-QoS-Teams`, assign aan `2ITcsc1A-SASE-Devices`.

**Verifieer de registry-markeringen** (leesbaar als non-admin user, want een HKLM-read vereist geen elevatie):

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS'
# TeamsAudio / TeamsVideo / TeamsScreenshare elk aanwezig:
#   DSCP 46/34/18, NetProfile 7 (alle profielen), Protocol 3 (TCP+UDP)
```

> **Opmerking:** De marking-op-de-wire-check hoort op het VyOS-pad (de SASE-gateway leest de DSCP-waarde downstream); een lokale outbound-capture op het endpoint toont de mark onbetrouwbaar omdat die onder de NDIS-hook zit.

---

## Stap 6: De split-tunnel-route-Remediation deployen (`2ITCSC1A-Route-Remediation`)

Voegt de M365-Optimize-routes toe die naar de fysieke gateway wijzen in plaats van naar de NetBird-`wt0`-interface, zodat latency-gevoelige Microsoft-media de exit node omzeilt. NetBird heeft geen native route-exclusion, dus een OS-route via een Remediation is de enige manier om Optimize-media van de tunnel te houden. Windows kiest de route op longest-prefix-match, dus een `/14` wint van NetBirds `/1`-split, ongeacht de metric.

Navigeer naar: `Intune → Devices → Remediations → Create script package`

```
Name: 2ITCSC1A-Route-Remediation
Run this script using the logged-on credentials: No   (draait als SYSTEM)
Enforce script signature check: No
Run script in 64-bit PowerShell: Yes
Schedule: Hourly
Assignment: 2ITcsc1A-SASE-Devices
```

Prefix-lijst gedeeld door beide scripts (detect + remediate moeten **exact dezelfde** lijst gebruiken):

```
13.107.64.0/18
52.112.0.0/14
52.120.0.0/14
```

Het detect-script meldt de routes missing (exit 1) wanneer een prefix off-tunnel ontbreekt; het remediate-script ontdekt de fysieke gateway dynamisch (de `0.0.0.0/0`-route op een niet-`wt0`-interface) en voegt de routes opnieuw toe. SYSTEM heeft de route-privilege, dus dit heeft het lokale-admin-account niet nodig.

**Pas de addendum-fix toe: koppel de exit-code los van de staat.** Het remediate-script moet:

1. `-ErrorAction Stop` zetten op `New-NetRoute` zodat een mislukte add een terminating error wordt;
2. dit in een `try/catch` zetten die de fout **bewaart en logt** in plaats van die te slikken;
3. een **verify-loop** draaien die exit 0 enkel toestaat wanneer elke prefix daadwerkelijk off-tunnel resolvet (`InterfaceAlias -ne 'wt0'`).

Zonder dit loopt een non-terminating `New-NetRoute`-fout stil door naar de succesregel en logt Intune exit 0 terwijl er geen route bestaat: dezelfde klasse vals "succes" als de certificaat-`Succeeded` die loog.

**Verifieer:**

```powershell
Find-NetRoute -RemoteIPAddress 52.112.0.1
# InterfaceAlias: Ethernet0 (de fysieke gateway), niet wt0
```

Na een tick meldt het device **Without issues** in de portal.

> **Valkuil: `New-NetRoute` weigert `PersistentStore` op deze build** (System Error 87, `ERROR_INVALID_PARAMETER`); het schrijft enkel ActiveStore. Persistentie komt daarom van de hourly Remediation (zelfhelend tegen reboot en tegen een NetBird-reconnect die de tabel herschrijft), niet van een persistente route.
>
> **Valkuil: `tracert` is blind voor de Optimize-relays.** Microsoft-Optimize-relays beantwoorden geen ICMP/UDP-probes, dus `tracert` die drie sterren toont betekent niet dat de split-tunnel faalde. Verifieer met `Find-NetRoute`, dat Windows rechtstreeks vraagt welke interface het zou kiezen.

---

## Stap 7: Uitgewerkt enrollment-voorbeeld, SITE01 (Tiny11) bereikt Entra-joined + Intune-enrolled

De remote-site-machine `2ITCSC1A-SITE01` (een Tiny11-image) is met `student_1` tot Entra-joined + Intune-enrolled gebracht. Dit is de werkende procedure om een tweede managed device te enrollen zodat de profielen hierboven ook daar landen.

Procedure (alle join-handelingen **aan de console**, niet over RDP, want de WAM-flow gedraagt zich anders in een remote sessie):

1. **Account:** enroll met het gelicenseerde account `student_1` (het enige Intune-gelicenseerde account; het persona-account `docent1` is bewust ongelicenseerd). De persona-laag (NetBird) en de device-owner-laag (Entra) zijn onafhankelijk.
2. **Activeer de Workplace-Join-takenset** (een Tiny11-debloat zet ze disabled, wat `0x80041326 SCHED_E_TASK_DISABLED` geeft bij de join):
   ```powershell
   Get-ScheduledTask -TaskPath "\Microsoft\Windows\Workplace Join\" | Enable-ScheduledTask
   ```
   Activeer alle drie (Device-Sync draagt de periodieke CSP-pull; enkel de join-taak activeren laat het device later joined maar zonder policies).
3. **Start de Sign-in Assistant** (Tiny11 laat die gestopt):
   ```powershell
   Set-Service wlidsvc -StartupType Manual; Start-Service wlidsvc
   ```
4. **Bevestig dat de Microsoft-auth-keten splicet op de proxy.** Interactieve auth over de PAC vereist de volledige no-bump-set; zorg dat `.microsoftazuread-sso.com` en `.live.com` op de Squid-splice-lijst staan naast `.microsoftonline.com` en `enterpriseregistration.windows.net` (Runbook 03).
5. **Gebruik de Entra-*join*, niet de registratie.** De e-mailveld-flow doet een device-*registratie*; een geregistreerd device valt niet in de dynamische device group, dus de profielen landen niet. Als `dsregcmd` `WorkplaceJoined: YES` / `AzureAdJoined: NO` toont, disconnect dat account, reboot, en gebruik de link "Join this device to Microsoft Entra ID" (een bestaande WorkplaceJoined-staat verbergt die).

**Resultaat: enrolled.**

```powershell
dsregcmd /status
#   AzureAdJoined : YES
#   DeviceId      : c72bcdc0-12ce-4217-817d-ea105604b54a
#   Managed by MDM (auto-enroll, MDM-scope All)
```

De `displayName 2ITCSC1A-SITE01` matcht de dynamische regel, dus het device komt in `2ITcsc1A-SASE-Devices` en de cert-/proxy-/QoS-profielen landen erop. Het device is onafhankelijk bevestigd tegen de device-store (`dsregcmd`), met bump/splice/RPZ alle drie bewezen vanaf SITE01.

**Resterende taak op deze image:** Defender staat uit op de Tiny11-build, dus het device faalt de antivirus-compliance-check ([Runbook 07: Toegangsbeleid](07-access-policy.nl.md), Gate 2). Herstel Defender voordat de compliance-evaluatie relevant wordt, of documenteer de Defender-uit-staat als een bewuste Tiny11-gap naast de software-KSP-TPM-gap.

---

## Validatiechecklist

- [ ] `2ITcsc1A-SASE-Devices` dynamische group resolvet het doel-device (tab Members)
- [ ] `2ITCSC1A-SASE-PoC-CA-Root` → `certutil -store Root` toont `O=SASE PoC` (thumbprint `55EF…`); geen CA van een andere groep aanwezig
- [ ] `2ITCSC1A-SASE-Proxy-PAC` → `https://google.com` issuer `O=SASE PoC`; overlay-bron-IP in het pop01-access-log
- [ ] `2ITCSC1A-SASE-FW-BypassBlock` → `netbird status` Connected, web via proxy werkt, Intune-Sync successful (gerichte block-set, geen default-deny)
- [ ] `2ITCSC1A-SASE-QoS-Teams` → drie QoS-policies in `HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS` (DSCP 46/34/18, NetProfile 7, Protocol 3)
- [ ] `2ITCSC1A-Route-Remediation` → `Find-NetRoute -RemoteIPAddress 52.112.0.1` geeft `Ethernet0`; portal toont **Without issues**
- [ ] `.\saseuser`-recovery-login bevestigd vóór de firewall-push
- [ ] Tweede device (SITE01) `AzureAdJoined: YES`, in de device group; Defender-uit-compliance-item opgevolgd

---

## Gerelateerd

- [Component: Intune Endpoint Enforcement](../components/intune-endpoint-enforcement.nl.md)
- [Component: NetBird](../components/netbird.nl.md)
- [Component: Squid](../components/squid.nl.md)
- [Beslissing: Beheerde apparaten-scope](../decisions/managed-devices-scope.nl.md)
- [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.nl.md)
- [Runbook 07: Toegangsbeleid](07-access-policy.nl.md)
- [Runbook 03: Proxy & WPAD](03-proxy-wpad.nl.md)
