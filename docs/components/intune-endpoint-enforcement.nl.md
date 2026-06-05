---
title: "Intune Endpoint Enforcement"
tags: [intune, mdm, firewall, identity]
---

# Intune Endpoint Enforcement

**Rol:** De MDM die de SASE-posture op managed Windows-clients pusht. Dit is de device-zijde van de stack, naast de netwerk-zijde controles (Squid SWG, Unbound RPZ, NetBird overlay).  
**Platform:** Microsoft Intune / Entra ID (`aplab.be`-tenant)  
**Configuratielocatie:** Intune-portaal (`https://intune.microsoft.com`): Configuration profiles, Endpoint Security Firewall en Remediations, allemaal gescopet op de dynamische device-group `2ITcsc1A-SASE-Devices`

---

## Werking in deze stack

De managed Windows-devices zijn Entra-joined en Intune-enrolled (zie [Beslissing: Managed Windows Devices Scope](../decisions/managed-devices-scope.md)). Die enrollment is wat de hele aanpak mogelijk maakt: in plaats van elke client met de hand te configureren, pusht Intune de proxy, het certificaat, de firewall, de QoS en de routes rechtstreeks naar het device.

De taakverdeling tussen Intune en de rest van de stack volgt het Drie-gate model (zie [Beslissing: CA + NetBird Posture](../decisions/ca-posture-hybrid.md)). Entra ID Conditional Access kijkt naar identiteit (Gate 1) en leest de device compliance die Intune doorgeeft (Gate 2). Deze pagina beschrijft de andere kant van diezelfde managed-device-basis: de configuratie die Intune naar het toestel pusht. Elk profiel zet één stuk van de SASE-posture vast:

- **Afgedwongen PAC-proxy** (`2ITCSC1A-SASE-Proxy-PAC`) stuurt browserverkeer door de pop01 Squid SWG. Zo staat de proxy echt vast op het device en is het geen instelling die de gebruiker zelf kan wissen.
- **Trusted-cert-profiel** (`2ITCSC1A-SASE-PoC-CA-Root`) installeert de SASE-PoC-CA-root, zodat de certificaatketen van SSL Bump vertrouwd is en HTTPS niet breekt.
- **Firewall-block-set** (`2ITCSC1A-SASE-FW-BypassBlock`) sluit de twee bypass-vectoren op protocolniveau (QUIC en DoT) die verkeer anders om de SWG en de gefilterde resolver heen zouden leiden.
- **DSCP/QoS-profiel** (`2ITCSC1A-SASE-QoS-Teams`) draagt de Teams-markeringen waarop de VyOS SASE-gateway shapet.
- **Route-remediation** (`2ITCSC1A-Route-Remediation`) installeert de M365-Optimize split-tunnel, zodat latency-gevoelige Microsoft-media de NetBird exit node mijdt.

Eén ding komt bij alle vijf terug: het statusveld per profiel in Intune zegt niets over de werkelijkheid. Wat telt is de device-store zelf (`certutil -store`, `Get-ChildItem Cert:\LocalMachine\Root`, het register, `Find-NetRoute`); dat is wat daadwerkelijk geverifieerd is. Een profiel kan rustig "Succeeded" tonen terwijl er niets geland is (V41).

## Configuratie

### Afgedwongen PAC-proxy: `2ITCSC1A-SASE-Proxy-PAC`

Een Custom OMA-URI-profiel dat de NetworkProxy-CSP aanstuurt. De Settings Catalog toont NetworkProxy niet, dus loopt het via de directe OMA-URI-route:

```
./Vendor/MSFT/NetworkProxy/ProxySettingsPerUser   Integer  0
./Vendor/MSFT/NetworkProxy/AutoDetect             Integer  0
./Vendor/MSFT/NetworkProxy/SetupScriptUrl         String   http://wpad.sandbox.local/wpad.dat
```

`ProxySettingsPerUser=0` maakt de instelling systeembreed (alle apps die WinINET respecteren), `AutoDetect=0` forceert de expliciete PAC in plaats van WPAD-discovery, en `SetupScriptUrl` wijst naar het bestaande WPAD/PAC-doel. Dit is de MDM-variant van de SWG-afdwinging; de handmatige Windows-proxy blijft de fallback ([Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md), stap 8). Er staat met opzet geen ProxyServer-node in: een PAC-URL samen met een ingevulde manual proxy server is een bekende Intune-bug die het profiel sloopt. Na het landen van het profiel is een reboot nodig om de cert- en proxy-cache te wissen.

### Trusted-cert-profiel: `2ITCSC1A-SASE-PoC-CA-Root`

Een Trusted-certificate-template (Windows 10 and later) die de publieke SASE-PoC-CA-`.crt` (zonder private key) in de **Computer certificate store - Root** zet. Hierdoor vertrouwt het device het SSL-Bump-certificaat (issuer `O=SASE PoC`, thumbprint `55EF83FE8BE080B7DF9AE92E9E55CFCEF3AC4537`), zodat gebumpte HTTPS valideert in plaats van certificaatfouten te geven.

### Firewall-block-set: `2ITCSC1A-SASE-FW-BypassBlock`

Twee gerichte Microsoft Defender Firewall outbound-regels via Endpoint Security → Firewall:

| Regel | Richting | Protocol | Remote port | Actie | Effect |
|-------|----------|----------|-------------|-------|--------|
| `Block-QUIC-Outbound-UDP443` | Out | UDP (17) | 443 | Blocked | Duwt browsers van QUIC terug naar TCP/443, zodat Squid de sessie kan bumpen |
| `Block-DoT-Outbound-TCP853` | Out | TCP (6) | 853 | Blocked | Dwingt DNS door de gefilterde resolver (beschermt het RPZ- en threat-intel-pad) |

Dit is bewust een **gerichte block-set en geen default-deny**. Een volledige default-deny outbound legt het MDM-check-in-pad plat: het WinHTTP-SYSTEM-kanaal naar de Microsoft control plane is een grote set FQDN's achter een CDN, en die laat zich met IP-gebaseerde firewall-regels niet netjes allowlisten. Precies de IT1214934-footgun. DoH over TCP/443 valt niet op poort te blokkeren en blijft een aparte zorg. Het schema eist dat het TCP/UDP-protocol op elke regel met poort-ranges gezet is, anders faalt de regel met "the parameter is incorrect".

### DSCP- / QoS-profiel: `2ITCSC1A-SASE-QoS-Teams`

Drie NetworkQoSPolicy-CSP-nodes (officieel ondersteund op Intune-managed en Entra-joined devices) die `ms-teams.exe` matchen op source-port-ranges en DSCP markeren:

| Stream | Source ports | DSCP | Klasse |
|--------|--------------|------|--------|
| Teams audio | 50000–50019 | 46 | EF |
| Teams video | 50020–50039 | 34 | AF41 |
| Teams screenshare | 50040–50059 | 18 | AF21 |

De markeringen worden op het endpoint gecontroleerd door `HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS` uit te lezen (alle drie de policies aanwezig, `NetProfile 7` = alle profielen, `Protocol 3` = TCP+UDP). De controle op de markering op de wire gebeurt op het VyOS-pad, waar de SASE-gateway de DSCP-waarde verderop ziet.

### Split-tunnel route-remediation: `2ITCSC1A-Route-Remediation`

Een Intune Remediation (detect + remediate, **draait als SYSTEM**, 64-bit, elk uur) die de M365-Optimize-routes toevoegt en naar de fysieke gateway laat wijzen in plaats van naar de NetBird `wt0`-interface:

```
13.107.64.0/18
52.112.0.0/14
52.120.0.0/14
```

Windows kiest een route op de langste prefix, dus een `/14` wint van de `/1`-split van NetBird, los van de metric. NetBird kan zelf geen routes uitsluiten, dus een OS-route via de Remediation is de enige manier om Optimize-media van de exit node weg te houden. De SYSTEM-context heeft de rechten om routes te zetten, dus het local-admin-account is hier niet nodig. Het remediate-script en het detect-script delen exact dezelfde prefix-lijst, en het remediate-script zoekt de fysieke gateway dynamisch op (de `0.0.0.0/0`-route op een interface die niet `wt0` is) in plaats van die hard te coderen.

De Remediation koppelt de exit code los van de succesmelding van het script: een `try/catch` vangt elke `New-NetRoute`-fout op en logt die (`-ErrorAction Stop`), en een verify-loop staat exit 0 pas toe als elke prefix echt buiten de tunnel resolvet. Zonder die controle liep een non-terminating `New-NetRoute`-fout ongemerkt door naar de succesregel: Intune logde exit 0 terwijl er geen route bestond. Dezelfde soort vals "succes" als de certificaat-`Succeeded` die loog. Na één tick staat het device op `Without issues` in het portaal.

### Device-scoping: `2ITcsc1A-SASE-Devices`

Al het bovenstaande wijst naar één dynamische device-group met deze membership-regel:

```
device.displayName -startsWith "2ITcsc1A-"
```

Targeten op user-group is hier te grof: een Proxy- of Firewall-CSP op het device van een klasgenoot dat niet op de overlay zit, zou diens internet breken. Daarom gaat de config naar de device-group. De regel leest de Entra-`displayName`, niet de naam die Intune toont. De casing van de prefix in de regel (`2ITcsc1A-`) wijkt af van de profielnamen in hoofdletters (`2ITCSC1A-…`) en van de device-namen (`2ITCSC1A-MOB-1`, `2ITCSC1A-SITE01`). Dat maakt niets uit: string-operaties in dynamische regels zijn hoofdletterongevoelig (alleen property-namen zijn hoofdlettergevoelig).

## Integratiepunten

| Interface | Richting | Wat |
|-----------|----------|-----|
| NetworkProxy-CSP → WinINET | Uitgaand (config-push) | Zet de systeem-PAC-URL die Edge en Chrome lezen en stuurt verkeer naar pop01 Squid `100.70.154.79:3128`; de WinHTTP-SYSTEM-proxy blijft met opzet Direct voor de MDM-check-in |
| Trusted-cert → Computer-Root-store | Uitgaand (config-push) | Installeert SASE-PoC-CA, zodat Squid SSL Bump (issuer `O=SASE PoC`) vertrouwd is |
| Endpoint Security Firewall → Defender Firewall | Uitgaand (config-push) | Twee outbound-block-regels (UDP/443 QUIC, TCP/853 DoT) |
| NetworkQoSPolicy-CSP → register | Uitgaand (config-push) | Teams-DSCP-markeringen waarop de VyOS SASE-gateway shapet |
| Route-Remediation → Windows-routetabel | Uitgaand (SYSTEM, elk uur) | M365-Optimize split-tunnel weg van de NetBird exit node |
| Entra ID Conditional Access | Upstream (consumeert de status) | Gate 2 leest de Intune device compliance; CA Policy 5 vereist een compliant device. Zie [Runbook 07: Access Policy](../runbooks/07-access-policy.md) |

## Bekende problemen / valkuilen

- **De Intune-status liegt bij een gedeelde CSP-setting.** Op de gedeelde `aplab.be`-tenant blokkeert een ruim gescopet Trusted-Root-profiel van een andere groep stilletjes het jouwe als het op dezelfde `Windows81TrustedRootCertificate`-CSP-setting mikt: beide profielen melden "Succeeded", maar er landt maar één certificaat in de store. Alleen de device-store vertelt de waarheid. De oplossing was het conflicterende profiel zo te scopen dat het managed device erbuiten viel; na een sync landde de SASE-PoC-CA alsnog (V41).
- **IT1214934: een firewall-reconciliatie kan het device bricken.** Een firewall-policy hernoemen of toevoegen kan een bestaande outbound-regel omslaan naar "block any outbound". Daarmee valt alle internetcommunicatie weg en is er geen controle meer op afstand; herstellen kan alleen met een lokale login plus het opschonen van het register onder `FirewallPolicy\Mdm\FirewallRules`. Daarom is de block-set gericht en geen default-deny, en daarom is het local-admin-account `.\saseuser` een voorwaarde voor herstel, klaargezet vóór de eerste firewall-push. Kaal `saseuser` werkt niet (UAC leest het als een Entra-account); de prefix `.\saseuser` voor een lokaal account wel.
- **`New-NetRoute` weigert `PersistentStore` op deze build** met System Error 87 (`ERROR_INVALID_PARAMETER`) en schrijft alleen naar ActiveStore. Persistentie komt daarom van de Remediation die elk uur draait (zelfhelend tegen een reboot en tegen een NetBird-reconnect die de tabel herschrijft), niet van een persistente route.
- **`tracert` ziet niets op de Optimize-relays.** De Microsoft-Optimize-relays beantwoorden geen ICMP- of UDP-probes, dus drie sterren in `tracert` betekenen niet dat de split-tunnel faalde. Controleer met `Find-NetRoute -RemoteIPAddress 52.112.0.1`: dat vraagt Windows zelf welke interface het zou kiezen (het antwoordt `Ethernet0`, de fysieke gateway).
- **Enrollment zet de handmatige proxy uit.** De Entra-enrollment schakelt de handmatig ingestelde PAC-setup uit, dus HTTPS gaat Direct tot het afgedwongen-PAC-profiel landt. Dat is precies waar dat profiel voor is, geen bug.

## Gerelateerd

- [Beslissing: CA + NetBird Posture (Drie-gate model)](../decisions/ca-posture-hybrid.md)
- [Beslissing: Managed Windows Devices Scope](../decisions/managed-devices-scope.md)
- [Component: NetBird](netbird.md)
- [Component: Squid](squid.md)
- [Runbook 07: Access Policy](../runbooks/07-access-policy.md)
- [Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md)
