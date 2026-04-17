---
title: "Acceptatietestresultaten (F1–F15)"
tags: [sase, ztna, swg, fwaas, casb, sd-wan, testing]
---

# Acceptatietestresultaten (F1–F15)

**Versie:** 1.0 — april 2026  
**Scope:** Volledige sandboxtestdekking — alle F1–F15-acceptatietests (Handboek v4 §46) plus aanvullende validatietests buiten het oorspronkelijke kader.  
**Bron:** `raw/SASE_PoC_Testrapport.md`

---

## Algemene status

| Test | Naam | Pijler | Status |
|------|------|--------|--------|
| **F1** | ZTNA-tunnelverbinding | ZTNA | ✅ Gevalideerd |
| **F2** | Entra ID SSO | ZTNA | ✅ Gevalideerd |
| **F3** | Posturecontrole | ZTNA | ⏳ Gepland — architectuur gereed (Addendum E) |
| **F4** | Datacentertoegang via ZTNA | ZTNA | ✅ Gevalideerd |
| **F5** | URL-filtering | SWG | ✅ Gevalideerd |
| **F6** | SSL-Bump-inspectie | SWG | ✅ Gevalideerd |
| **F7** | Malwaredetectie (ClamAV) | SWG | ✅ Gevalideerd |
| **F8** | Firewall-segmentatie | FWaaS | ✅ Gevalideerd (indirect — ZTNA ACL-handhaving) |
| **F9** | Suricata-alertgeneratie | FWaaS | ✅ Gevalideerd (uitgebreid voorbij oorspronkelijke definitie) |
| **F10** | Centrale logaggregatie (SIEM) | SIEM | ⏳ Gepland — vereist Wazuh-implementatie |
| **F11** | CASB-alert en herstel | CASB | ⏳ Gepland — vereist Wazuh + Graph API |
| **F12** | IPsec-tunnelverbinding | SD-WAN | ✖ N.v.t. — architectuurbeslissing (zie [SD-WAN Geschrapt](../decisions/sdwan-descoped.md)) |
| **F13** | QoS-verkeersclassificatie | SD-WAN | ✖ N.v.t. — architectuurbeslissing |
| **F14** | Datacentertoegang via SD-WAN | SD-WAN | ✖ N.v.t. — architectuurbeslissing |
| **F15** | Volledige SASE-validatie | Integratie | ✅ Gedeeltelijk — stappen 1–6 gevalideerd, stappen 7–8 N.v.t. (SD-WAN), stap 9 gepland (SIEM) |

### Aanvullende gevalideerde tests (buiten F1–F15)

| Test | Naam | Status |
|------|------|--------|
| **T-A1** | DLP YARA — detectie van CONFIDENTIAL-label bij download | ✅ Gevalideerd |
| **T-A2** | DLP SDD — StructuredDataDetection-drempelwaarde (4× CC → blokkeren, 1× CC → doorlaten) | ✅ Gevalideerd |
| **T-A3** | DLP ICAP REQMOD — Python DLP uploadblokkering (Luhn CC in POST) | ✅ Gevalideerd |
| **T-A4** | DNS RPZ — NXDOMAIN + aa-vlag van pop01 lokaal | ✅ Gevalideerd |
| **T-A5** | DNS RPZ — NXDOMAIN van mobile01 via NetBird-overlay | ✅ Gevalideerd |
| **T-A6** | DNS RPZ — NXDOMAIN van dc01 via DC-LAN | ✅ Gevalideerd |
| **T-A7** | Suricata vtnet1 (LAN) — DC-LAN-verkeersdetectie | ✅ Gevalideerd |
| **T-A8** | Suricata verdachte User-Agent | ✅ Gevalideerd |
| **T-A9** | Suricata DNS-anomaliedetectie | ✅ Gevalideerd |

---

## Dekking per SASE-pijler

| Pijler | Gevalideerd | Gepland | N.v.t. |
|--------|-------------|---------|--------|
| **ZTNA** | F1, F2, F4 | F3 (architectuur gereed) | — |
| **SWG** | F5, F6, F7, T-A1, T-A2, T-A3, T-A4–A6 | — | — |
| **FWaaS** | F8, F9, T-A7, T-A8, T-A9 | F10 (Wazuh-afhankelijkheid) | — |
| **CASB** | — | F11 (Wazuh + Graph API) | — |
| **SD-WAN** | — | — | F12, F13, F14 |

---

## Gevalideerde tests — onderbouwing en sleuteluitvoer

### F1 — ZTNA-tunnelverbinding

`InterfaceAlias: wt0` in de uitvoer van Test-NetConnection bewijst dat al het verkeer via de WireGuard-tunnel vertrekt, niet via de VMware-host-NIC. Als wt0 niet routeerde, zou het antwoord binnenkomen via de standaardadapter.

```powershell
# mobile01 (PowerShell)
netbird status
Test-NetConnection 8.8.8.8 -Port 443
```

Verwachte uitvoer:

```
FQDN: mobile01.netbird.selfhosted
NetBird IP: 100.70.95.98/16
Peers count: 1/1 Connected

InterfaceAlias   : wt0          ← WireGuard-tunnel, niet host-NIC
SourceAddress    : 100.70.95.98
TcpTestSucceeded : True
```

Zie [NetBird](../components/netbird.md).

---

### F2 — Entra ID SSO

Valideert de volledige OIDC-keten: mobile01 → NetBird → Zitadel → Entra ID (login.microsoftonline.com) → terug naar NetBird. De peer die in het NetBird-dashboard verschijnt met de Entra ID-gebruikersnaam bewijst end-to-end identiteitskoppeling — geen lokaal account.

Testprocedure:

1. `netbird down` op mobile01
2. `netbird up` → browser stuurt door naar `https://netbird.sandbox.local` → Zitadel → `login.microsoftonline.com`
3. Authenticeer als `2ITCSC1A-mobile_user1@aplab.be`
4. Tunnel wordt actief; peer verschijnt in dashboard, automatisch toegewezen aan groep `SASE-MobileUsers` via setup key auto-group

Zie [NetBird](../components/netbird.md), [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md).

---

### F3 — Posturecontrole (Gepland)

Architectuur volledig gespecificeerd in Addendum E (april 2026):

- **Poort 1:** Vier Entra ID CA-beleidsregels — MFA-handhaving, geo-blokkering (alleen België), blokkering van verouderde authenticatie, aanmeldingsrisico
- **Poort 2:** Vier NetBird-posturecontroles — OS-kernel ≥ 10.0.19041, clientversie, `MsMpEng.exe` AV-proces, geo België
- Vijf geplande validatiescenario's, inclusief negatieve tests (geo-blokkering, OS-versie spoofing, AV gestopt, verouderde auth, volledig positieve end-to-end)

**Kritieke voorwaarde:** Controleer MFA-registratie op het testaccount voordat CA-beleidsregels worden geactiveerd. Zonder eerdere MFA-instelling triggert de eerste aanmelding een onherstelbare "MFA vereist maar niet geconfigureerd"-blokkering, waardoor het account ontoegankelijk wordt.

Gepland na de tussentijdse evaluatie (20 april 2026). Zie [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.md).

---

### F4 — Datacentertoegang via ZTNA

DC-LAN (10.0.0.0/24) is alleen bereikbaar via het NetBird Networks-mechanisme. Een host die niet is ingeschreven bij NetBird heeft geen route naar dit subnet, ongeacht IP-connectiviteit.

```powershell
# mobile01 (PowerShell)
Test-NetConnection 10.0.0.100 -Port 80
```

Verwacht: `InterfaceAlias: wt0`, `TcpTestSucceeded: True`. Valideert het ACL-beleid `Datacenter Access` en de Networks-routeringsconfiguratie.

Zie [NetBird](../components/netbird.md).

---

### F5 — URL-filtering

`curl.exe` met een expliciet proxy-argument omzeilt de browsercache, die blokkering anders zou kunnen maskeren. `X-Squid-Error: ERR_ACCESS_DENIED` identificeert de Squid-blokkeringspagina — geen server-side 403.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 http://gambling.com -v
```

Verwacht:

```
< HTTP/1.1 403 Forbidden
< X-Squid-Error: ERR_ACCESS_DENIED 0

# Squid-toegangslog (pop01)
TCP_DENIED/403 ... GET http://gambling.com/ - HIER_NONE/-
```

Geblokkeerde categorieën: adult, malware, phishing, gokken (UT1 Toulouse Remote ACL) plus handmatige blacklist-vermeldingen (`gambling.com`, `.bet365.com`, `.pokerstars.com`). Zie [Squid](../components/squid.md).

---

### F6 — SSL-Bump-inspectie

Twee complementaire tests bewijzen selectieve, intentionele HTTPS-interceptie:

- **Normale HTTPS:** Navigeer naar `https://google.com` → certificaatuitgever = `O = SASE PoC` (niet Google)
- **Geen-bump-uitzondering:** Navigeer naar `https://login.microsoftonline.com` → uitgever = Microsoft Corporation (origineel certificaat, niet vervangen)

De tweede test is cruciaal: hij bewijst dat de implementatie bewust is, geen massale decryptie. Microsoft-aanmelding staat op de geen-bump-lijst specifiek omdat SSL Bump de Entra ID OIDC-stroom zou verbreken.

Zie [SSL Bump](../concepts/ssl-bump.md), [Squid](../components/squid.md).

---

### F7 — Malwaredetectie (ClamAV/EICAR)

De vergelijking van 68 bytes versus 8245 bytes is het bewijs van actieve blokkering: 68 bytes is het EICAR-bestand; 8245 bytes is de Squid-blokkeringspagina gegenereerd door de ICAP-pijplijn.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o eicar_test.txt https://secure.eicar.org/eicar.com
```

> `--ssl-no-revoke` is vereist op Windows: Schannel probeert CRL/OCSP-validatie van het SASE-PoC-CA-certificaat, dat geen gepubliceerd CRL-eindpunt heeft. Zie [Bevinding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.md).

Verwacht:

```
# c-icap-log (pop01)
VIRUS DETECTED: Eicar-Test-Signature,
  http client ip: 100.70.95.98,
  http url: https://secure.eicar.org/eicar.com

# Squid-toegangslog
TCP_MISS/403 8245 GET https://secure.eicar.org/eicar.com
```

Zie [ClamAV/c-icap](../components/clamav-cicap.md).

---

### F8 — Firewall-segmentatie

DC-LAN-isolatie wordt afgedwongen op de overlay-laag — alleen NetBird-ingeschreven peers met het ACL-beleid `Datacenter Access` kunnen dc01 bereiken.

Gevalideerd door contrast:

- mobile01 **zonder** NetBird: `ping 10.0.0.100` → Bestemming onbereikbaar (geen route)
- mobile01 **met** NetBird + beleid: `Test-NetConnection 10.0.0.100 -Port 80` → `TcpTestSucceeded: True` via `wt0`

**Architectuurnoot:** DC-LAN gebruikt NetBird Networks (niet Network Routes). Toegang vereist zowel overlay-inschrijving als expliciet groepslidmaatschap in `SASE-InternalResources`.

Zie [NetBird](../components/netbird.md).

---

### F9 — Suricata-alertgeneratie

Vier testcategorieën valideren vier onafhankelijke inspectiedomeinen over twee interfaces:

| Test | Methode | SID | Interface |
|------|---------|-----|-----------|
| F9-1: HTTP-inhoud | `curl.exe -x ... http://testmyids.com/` | 2100498 (GPL ATTACK_RESPONSE id check returned root) | vtnet0 |
| F9-2: DNS-anomalie | Automatisch — Unbound `.biz` TLD-query's tijdens normale werking | 2027863 (ET DNS Non-Compliant DNS Reply UDP) | vtnet0 |
| F9-3: User-Agent | `curl.exe -x ... -A "BlackSun" http://example.com` | 2008983 (ET MALWARE User-Agent BlackSun) | vtnet0 |
| F9-4: LAN-verkeer | `apt update` op dc01 | 2013504 (ET INFO GNU/Linux APT User-Agent Outbound) | vtnet1 |

vtnet1-verificatie:

```bash
grep '"in_iface":"vtnet1".*"event_type":"alert"' /var/log/suricata/eve.json | wc -l
# Verwacht: > 0
```

**Gedifferentieerd alertbeleid:**

| Categorie | Beleid | Onderbouwing |
|-----------|--------|--------------|
| emerging-malware, botcc, C2, Abuse.ch | Blokkeren | Altijd kwaadaardig — geen legitiem verkeer mogelijk |
| tor, info, policy, dns, web | Alerteren | Risico op valse positieven — bewaken, niet blokkeren |

> Het Handboek definieerde F9 oorspronkelijk als "controleer of alert zichtbaar is in Wazuh-dashboard." Wazuh is nog niet geïmplementeerd (Fase 4 — F10). Suricata-detectie is volledig gevalideerd; de Wazuh-integratie is de enige ontbrekende schakel.

> Meerdere curl-verzoeken naar dezelfde bestemming genereren één Suricata-alert per SID per flow — dit is correct gedrag, geen onderdrukking. Oorzaak: Squid-verbindingspooling hergebruikt upstream TCP-verbindingen; Suricata ziet één flow. Zie [Bevinding: Suricata verbindingspooling](../findings/suricata-connection-pooling.md).

Zie [Suricata](../components/suricata.md).

---

### T-A1 — DLP YARA: CONFIDENTIAL-label bij download

YARA werkt na de bestandsdecompositie-engine van ClamAV — het matcht op geëxtraheerde documenttekst, niet op ruwe bytes. Volledige pijplijn: Squid SSL Bump → c-icap RESPMOD → ClamAV-verwerking → YARA-match → HTTP 403.

```bash
# pop01 — lokale verificatie
clamdscan /tmp/dlp_test.txt   # bestand bevat "Dit document is CONFIDENTIAL..."
# Uitvoer: YARA.DLP_Confidential_Label.UNOFFICIAL FOUND
```

```powershell
# mobile01 — end-to-end via ICAP
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o dlp_result.txt http://wpad.sandbox.local/dlp_test.txt -w "%{http_code}"
# Uitvoer: 403
```

Gevalideerde YARA-regels in sandbox:

| Regel | Patroon | Scope |
|-------|---------|-------|
| `DLP_Confidential_Label` | CONFIDENTIAL, VERTROUWELIJK, GEHEIM, DO NOT DISTRIBUTE (nocase) | ✅ End-to-end |
| `DLP_IBAN_Pattern` | NL/DE/BE IBAN-formaat regex | ✅ Lokale clamdscan |
| `DLP_BSN_Candidate` | Cluster van 9-cijferige reeksen (drempelwaarde >2) | ✅ Lokale clamdscan |
| `DLP_AWS_AccessKey` | `AKIA[0-9A-Z]{16}` | ✅ Lokale clamdscan |

**Beperking:** YARA-regels voeren geen algoritmische validatie uit — geen mod-97 voor IBAN, geen 11-proef voor BSN. Valse positieven zijn mogelijk bij downloads. De Python DLP-laag (T-A3) biedt algoritmische validatie voor uploads.

Zie [ClamAV/c-icap](../components/clamav-cicap.md), [DLP](../concepts/dlp.md).

---

### T-A2 — DLP SDD: StructuredDataDetection-drempelwaarde

ClamAV's StructuredDataDetection (SDD) gebruikt Luhn-validatie — willekeurige 16-cijferige reeksen triggeren niet. De drempelwaarde is `StructuredMinCreditCardCount: 3` (geconfigureerd), dus detectie vuur bij 4+ geldige CC-nummers.

```bash
# 4 geldige Luhn CC-nummers → gedetecteerd
echo "CC1: 4532015112830366 CC2: 4916338506082832 CC3: 5425233430109903 CC4: 2223000048410010" > /tmp/sdd_test.txt
clamdscan /tmp/sdd_test.txt
# Uitvoer: Heuristics.Structured.CreditCardNumber FOUND

# 1 geldig CC-nummer → doorgelaten (onder drempelwaarde)
echo "CC: 4532015112830366" > /tmp/sdd_test_single.txt
clamdscan /tmp/sdd_test_single.txt
# Uitvoer: OK
```

Zie [ClamAV/c-icap](../components/clamav-cicap.md).

---

### T-A3 — Python DLP ICAP REQMOD: CC-blokkering bij upload

ClamAV c-icap kan `multipart/form-data` POST-bodies niet verwerken — dit is een bevestigde upstream-beperking (SourceForge: niet geïmplementeerd in de `virus_scan`-service). Python DLP vult deze leemte op via ICAP REQMOD.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -X POST `
  -d "payment_info=4532015112830366&name=Test+User" `
  https://httpbin.org/post
# Verwacht: HTML DLP-blokkeringspagina, HTTP 403
```

De DLP-blokkeringspagina is te onderscheiden van de generieke 403 van Squid door het DLP-specifieke foutbericht.

ICAP OPTIONS-verificatie (bevestigt dat REQMOD actief is):

```bash
# pop01 (FreeBSD) — gebruik printf, niet echo -e (FreeBSD /bin/sh interpreteert -e niet als escapevlag)
printf "OPTIONS icap://192.168.122.23:1345/dlpscan ICAP/1.0\r\nHost: 192.168.122.23\r\n\r\n" | nc -w 5 192.168.122.23 1345
# Uitvoer: ICAP/1.0 200 OK
#          Methods: REQMOD
```

Validatiematrix:

| Invoer | Algoritme | Status |
|--------|-----------|--------|
| `4532015112830366` (CC) | Luhn | ✅ End-to-end gevalideerd |
| `NL91ABNA0417164300` (IBAN) | mod-97 | ✅ Code aanwezig, niet end-to-end getest |
| `111222333` (BSN) | 11-proef | ✅ Code aanwezig, niet end-to-end getest |

Zie [Python DLP](../components/python-dlp.md), [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md).

---

### T-A4, T-A5, T-A6 — DNS RPZ: drie netwerksegmenten

Testdomein: `testentry.rpz.urlhaus.abuse.ch` — een permanente abuse.ch-validatie-invoer, altijd aanwezig in de URLhaus RPZ-feed.

De `aa`-vlag (autoritative answer) is de sleuteldiscriminator: een echte internet-NXDOMAIN mist de `aa`-vlag; een RPZ-blokkering wordt autoratitatief beantwoord door de lokale resolver zonder een upstream-query.

Controletest bevestigt geen overblokkering: `drill @127.0.0.1 google.com` geeft `NOERROR` terug zonder `aa`-vlag.

| Segment | Opdracht | Resultaat |
|---------|----------|-----------|
| **T-A4** pop01 (lokaal) | `drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch` | `rcode: NXDOMAIN, flags: aa` |
| **T-A5** mobile01 (overlay) | `nslookup testentry.rpz.urlhaus.abuse.ch` | Niet-bestaand domein |
| **T-A6** dc01 (DC-LAN) | `drill @10.0.0.1 testentry.rpz.urlhaus.abuse.ch` | `rcode: NXDOMAIN, flags: aa` |

Unbound RPZ-log bevestigt de zonenaam en bron-IP voor elke query:

```
rpz: applied [ioc2rpz-threat-intel] testentry.rpz.urlhaus.abuse.ch.
  rpz-nxdomain 100.70.95.98@58107 testentry.rpz.urlhaus.abuse.ch. A IN
```

DNS-pad voor T-A5: mobile01 → NetBird DNS-relay (100.70.255.254) → pop01 Unbound → RPZ-controle → NXDOMAIN.

Zie [ioc2rpz](../components/ioc2rpz.md), [RPZ](../concepts/rpz.md).

---

## Geplande tests en blokkers

### F10 — Centrale logaggregatie (SIEM)

Vereist Wazuh-implementatie (Fase 4). Logbronnen zijn al beschikbaar: Squid-toegangslog, Suricata `eve.json`, OPNsense-firewalllog, Python DLP Docker-log. Wazuh-Suricata-integratie via de Wazuh OPNsense-module is standaard en vereist geen maatwerkontwikkeling.

### F11 — CASB-alert en herstel

Vereist Wazuh-implementatie en Microsoft Graph API-configuratie. Architectuur is ontworpen (Afbakening v1.2 §2.2):

- **Inline-laag:** Squid + ICAP (al operationeel)
- **API-laag:** Wazuh `ms-graph`-module + Graph API (`AuditLog.Read.All`, `Directory.Read.All`)
- **Aangepaste regels:** SID 100200/100201 voor SharePoint `SharingSet` + `Anyone`/`External`-detectie
- **Actieve respons:** `sharepoint_remediate.sh`

Graph API-tenantmachtigingen bevestigd op 1 april 2026 (beheerdertoestemming verleend op `aplab.be`). Enige resterende blokker: Wazuh-implementatie.

---

## F15 — Volledige validatie stap voor stap

| Stap | Beschrijving | Status |
|------|-------------|--------|
| 1 | Mobiele gebruiker logt in via Entra ID | ✅ Gevalideerd (F2) |
| 2 | Bezoekt HTTPS-site → SSL Bump actief | ✅ Gevalideerd (F6) |
| 3 | Bezoekt geblokkeerde site → Geblokkeerd | ✅ Gevalideerd (F5) |
| 4 | Downloadt EICAR → Geblokkeerd | ✅ Gevalideerd (F7) |
| 5 | Bezoekt testmyids.com → Alert in IDS | ✅ Gevalideerd (F9-1) |
| 6 | Opent datacentersite → Geslaagd | ✅ Gevalideerd (F4) |
| 7 | Sitegebruiker pingt datacenter via tunnel | ✖ N.v.t. — sitepc01 heeft geen OS, IPsec geschrapt |
| 8 | QoS-markering zichtbaar op VyOS | ✖ N.v.t. — QoS-architectuurbeslissing |
| 9 | Managementdashboard toont alle services UP | ⏳ Gepland (Wazuh) |

6 van 9 F15-stappen gevalideerd. De 3 resterende zijn ofwel architectureel N.v.t. (SD-WAN) of gepland (SIEM).

---

## Testomgeving

| Aspect | Waarde |
|--------|--------|
| Snapshot bij validatie | `Fase2-ZTNA-Complete` (basis), daarna Fase 3 incrementeel |
| pop01 RAM | 8 GB (verhoogd van 4 GB voor gelijktijdige Suricata + ClamAV-belasting) |
| ClamAV-handtekeningen | daily v27953, main v63, bytecode v339 — 3.642.437 handtekeningen |
| Suricata-regels | ET Open + Abuse.ch: **79.620+** regels (Hyperscan-engine) |
| RPZ-records | URLhaus + ThreatFox: **71.767** records |
| mobile01 | Windows 11, VMware, extern — simuleert BYOD buiten schoolnetwerk |
| NetBird-overlaybereik | 100.64.0.0/10 |

---

## Gerelateerd

- [Architectuur](../overview/architecture.md)
- [Beslissing: SD-WAN Geschrapt](../decisions/sdwan-descoped.md)
- [Beslissing: CA + Posture hybride (Driepoortsmodel)](../decisions/ca-posture-hybrid.md)
- [Bevinding: curl --ssl-no-revoke op Windows](../findings/curl-ssl-no-revoke.md)
- [Bevinding: Suricata verbindingspooling](../findings/suricata-connection-pooling.md)
- [NetBird](../components/netbird.md)
- [Squid](../components/squid.md)
- [ClamAV/c-icap](../components/clamav-cicap.md)
- [Python DLP](../components/python-dlp.md)
- [Suricata](../components/suricata.md)
- [ioc2rpz](../components/ioc2rpz.md)
