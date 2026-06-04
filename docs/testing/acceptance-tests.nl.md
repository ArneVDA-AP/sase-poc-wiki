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
| **F3** | Posturecontrole | ZTNA | ⏳ Deels bewezen — CA-beleid actief, Intune-enrolled (Report-only) |
| **F4** | Datacentertoegang via ZTNA | ZTNA | ✅ Gevalideerd (bij opbouw, mei 2026) — DC-LAN-over-overlay-pad sindsdien verwijderd in V34, uitgesteld |
| **F5** | URL-filtering | SWG | ✅ Gevalideerd |
| **F6** | SSL-Bump-inspectie | SWG | ✅ Gevalideerd |
| **F7** | Malwaredetectie (ClamAV) | SWG | ✅ Gevalideerd |
| **F8** | Firewall-segmentatie | FWaaS | ✅ Gevalideerd — DC-LAN-isolatie blijft gelden (niet-ingeschreven = geen route); positief ACL-pad verwijderd in V34 |
| **F9** | Suricata-alertgeneratie | FWaaS | ✅ Gevalideerd (uitgebreid voorbij oorspronkelijke definitie) |
| **F10** | Centrale logaggregatie (SIEM) | SIEM | ✅ Operationeel — Wazuh + NATS-forwarder + pop01-agent |
| **F11** | CASB-alert en herstel | CASB | ✅ Functioneel — Wazuh + M365 Management Activity API + Active Response |
| **F12** | IPsec-tunnelverbinding | SD-WAN | ✖ N.v.t. — architectuurbeslissing (zie [SD-WAN Geschrapt](../decisions/sdwan-descoped.md)) |
| **F13** | QoS-verkeersclassificatie | SD-WAN | ✖ N.v.t. als klassieke-IPsec-test — QoS is in plaats daarvan geïmplementeerd + gevalideerd onder het ZT-Branch-model (V43 Test #5; zie [ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)) |
| **F14** | Datacentertoegang via SD-WAN | SD-WAN | ✖ N.v.t. — architectuurbeslissing |
| **F15** | Volledige SASE-validatie | Integratie | ✅ Gedeeltelijk — stappen 1–6 + 8 gevalideerd, stap 7 N.v.t. (klassiek IPsec SD-WAN), stap 9 operationeel (SIEM via Wazuh) |

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
| **T-A10** | Identity Bridge — overlay-IP naar personagroepresolutie | ✅ Gevalideerd |
| **T-A11** | NATS-eventbus — cross-componentgebeurtenisaflevering | ✅ Gevalideerd |
| **T-A12** | Control Daemon — dreigingsscoreaccumulatie + quarantaine | ✅ Gevalideerd |
| **T-A13** | Wazuh SIEM — NATS-gebeurtenisingestie + dashboardquery | ✅ Gevalideerd |

---

## Dekking per SASE-pijler

| Pijler | Gevalideerd | Gepland | N.v.t. |
|--------|-------------|---------|--------|
| **ZTNA** | F1, F2, F4 | F3 (architectuur gereed) | — |
| **SWG** | F5, F6, F7, T-A1, T-A2, T-A3, T-A4–A6 | — | — |
| **FWaaS** | F8, F9, F10, T-A7, T-A8, T-A9 | — | — |
| **CASB** | F11, T-A10, T-A11, T-A12, T-A13 | — | — |
| **SD-WAN** | QoS + failover-detectie (ZT-Branch, V43 Test #5/#6) | — | F12, F13, F14 (klassiek IPsec) |

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
4. Tunnel wordt actief; peer verschijnt in dashboard, toegewezen aan zijn persona-groep (`Studenten`/`Docenten`/`Admins`) via de Entra ID-groepsclaim in het OIDC-token — de groep materialiseert in NetBird bij de eerste verbinding (Pad B stript de `2ITCSC1A-`-prefix naar de interne groepsnaam)

Zie [NetBird](../components/netbird.md), [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md).

---

### F3 — Posturecontrole (Deels bewezen)

**Poort 1 — Entra ID Conditional Access:** Vijf CA-beleidsregels geïmplementeerd:

| Beleid | Modus | Effect |
|--------|-------|--------|
| SASE-PoC-MFA-Required | Actief | MFA afgedwongen op NetBird-app |
| SASE-PoC-Geo-Block | Report-only | België-only geobeperking (report-only tot demo) |
| SASE-PoC-Block-Legacy-Auth | Actief | Exchange ActiveSync + verouderde clients geblokkeerd |
| SASE-PoC-Risk-Block | Actief | Gemiddeld/Hoog aanmeldingsrisico triggert MFA |
| SASE-PoC-Compliant-Device | Report-only | Intune-conform apparaat vereist (report-only tot demo) |

**Poort 2 — Intune-apparaatnaleving:** Nalevingsbeleid dwingt OS-versie, Defender AV + firewall ingeschakeld, en realtime-beveiliging actief af.

Ingeschreven apparaten:

- **mobile01** — ingeschreven als `2ITCSC1A-MOB-1`, conform
- **sitepc01** — ingeschreven in NetBird als `docent1` (Docenten); afzonderlijk Entra joined + Intune-ingeschreven via `student1` (het enige Intune-gelicentieerde sandbox-account)

Twee beleidsregels blijven in report-only-modus (geo-blokkering, conform apparaat) tot de demo om lockout tijdens testen te vermijden.

**Kritieke voorwaarde:** Controleer MFA-registratie op het testaccount voordat CA-beleidsregels worden geactiveerd. Zonder eerdere MFA-instelling triggert de eerste aanmelding een onherstelbare "MFA vereist maar niet geconfigureerd"-blokkering, waardoor het account ontoegankelijk wordt.

Zie [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.md).

---

### F4 — Datacentertoegang via ZTNA

**✅ Gevalideerd bij opbouw (mei 2026); het pad is nadien verwijderd in V34.** Op het moment van validatie was DC-LAN (10.0.0.0/24) bereikbaar via het NetBird Networks-mechanisme: een host die niet bij NetBird was ingeschreven had geen route naar dit subnet, ongeacht IP-connectiviteit, terwijl een ingeschreven peer met het ACL-beleid `Datacenter Access` het wél kon bereiken.

```powershell
# mobile01 (PowerShell) — zoals getest in mei 2026
Test-NetConnection 10.0.0.100 -Port 80
```

Op het moment van validatie: `InterfaceAlias: wt0`, `TcpTestSucceeded: True`, wat het ACL-beleid `Datacenter Access` en de Networks-routeringsconfiguratie uitoefende.

**Huidige toestand:** Het `Internal-DC` Network en het `Datacenter Access`-beleid zijn verwijderd in de V34-personamigratie — site-to-site-achtige resourcetoegang is in strijd met het Zero-Trust per-resource-model. DC-LAN-over-overlay is uitgesteld en kan hervat worden in de Cosmos-sessie. Zie [runbook 02](../runbooks/02-ztna-overlay.md) en [NetBird](../components/netbird.md).

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

**✅ Gevalideerd.** De kern-segmentatie-eigenschap — een niet-ingeschreven apparaat heeft geen route naar DC-LAN (10.0.0.0/24) — geldt onafhankelijk van enig overlay-toegangspad en blijft vandaag waar.

Gevalideerd door contrast (mei 2026, toen het positieve pad nog bestond):

- mobile01 **zonder** NetBird: `ping 10.0.0.100` → Bestemming onbereikbaar (geen route) — *nog steeds waar*
- mobile01 **met** NetBird + het `Datacenter Access`-beleid: `Test-NetConnection 10.0.0.100 -Port 80` → `TcpTestSucceeded: True` via `wt0` — *positief pad verwijderd in V34*

**Architectuurnoot:** DC-LAN gebruikte NetBird Networks (niet Network Routes). Het positieve pad vereiste zowel overlay-inschrijving als lidmaatschap van de groep `SASE-InternalResources`; zowel het `Internal-DC` Network als die groep zijn verwijderd in de V34-personamigratie (zie [runbook 02](../runbooks/02-ztna-overlay.md)). De negatieve isolatie-eigenschap — niet-ingeschreven = geen route — blijft ongewijzigd.

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

> Het Handboek definieerde F9 oorspronkelijk als "controleer of alert zichtbaar is in Wazuh-dashboard." Wazuh is nu operationeel (F10) en Suricata-alerts stromen via NATS naar Wazuh, waarmee deze leemte is gedicht. Suricata-detectie en Wazuh-integratie zijn beide volledig gevalideerd.

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

### T-A10 — Identity Bridge: overlay-IP naar personagroepresolutie

Test dat de Identity Bridge overlay-IP's correct vertaalt naar Entra ID-personagroepen. Valideert door het `/lookup`-eindpunt te bevragen met een bekend overlay-IP en te controleren dat de teruggegeven groep overeenkomt met de verwachte persona.

---

### T-A11 — NATS-eventbus: cross-componentgebeurtenisaflevering

Test end-to-end gebeurtenisaflevering van een producent (bijv. Suricata) via NATS naar zowel de Control Daemon als Wazuh. Valideert door een Suricata-alert te triggeren en te controleren dat de gebeurtenis verschijnt in zowel NATS-monitoring als Wazuh.

---

### T-A12 — Control Daemon: dreigingsscoreaccumulatie + quarantaine

Test dreigingsscoreaccumulatie en quarantaine. Valideert door meerdere gemiddeld-ernstige gebeurtenissen te sturen voor dezelfde client, de Redis-score toename te observeren, en te verifiëren dat de peer wordt verwijderd uit personagroepen wanneer de drempelwaarde wordt overschreden.

---

### T-A13 — Wazuh SIEM: NATS-gebeurtenisingestie + dashboardquery

Test NATS-naar-Wazuh gebeurtenisingestie. Valideert door een detectiegebeurtenis te triggeren en vervolgens het Wazuh Discover-dashboard te bevragen naar de gebeurtenis met correcte velden.

---

## Operationele tests — voorheen gepland

### F10 — Centrale logaggregatie (SIEM)

Wazuh is geïmplementeerd en operationeel. De NATS-forwarder bruggt Suricata-alerts van de eventbus naar Wazuh, en de pop01-agent voedt lokale logbronnen (Squid-toegangslog, Suricata `eve.json`, OPNsense-firewalllog, Python DLP Docker-log). Gebeurtenissen zijn opvraagbaar in het Wazuh Discover-dashboard.

### F11 — CASB-alert en herstel

Wazuh is geïmplementeerd met M365 Management Activity API-integratie en Active Response:

- **Inline-laag:** Squid + ICAP (operationeel)
- **API-laag:** Wazuh + M365 Management Activity API
- **Aangepaste regels:** basisregel `100600` (`producer=o365`), met `100601` (`AnonymousLinkCreated`), `100602` (`SharingLinkCreated` + scope `Anyone`), `100603` (`SharingSet` + guest)
- **Actieve respons:** `sharepoint_remediate.sh` (regels 100601/100602), `guest_remediate.sh` (regel 100603) — achter de ENFORCE-poort, standaard detect-only (live revoke nog niet actief)

Graph API-tenantmachtigingen bevestigd op 1 april 2026 (beheerdertoestemming verleend op `aplab.be`).

---

## F15 — Volledige validatie stap voor stap

| Stap | Beschrijving | Status |
|------|-------------|--------|
| 1 | Mobiele gebruiker logt in via Entra ID | ✅ Gevalideerd (F2) |
| 2 | Bezoekt HTTPS-site → SSL Bump actief | ✅ Gevalideerd (F6) |
| 3 | Bezoekt geblokkeerde site → Geblokkeerd | ✅ Gevalideerd (F5) |
| 4 | Downloadt EICAR → Geblokkeerd | ✅ Gevalideerd (F7) |
| 5 | Bezoekt testmyids.com → Alert in IDS | ✅ Gevalideerd (F9-1) |
| 6 | Opent datacentersite → Geslaagd | ✅ Gevalideerd bij opbouw (F4); DC-LAN-over-overlay-pad verwijderd in V34, uitgesteld |
| 7 | Sitegebruiker pingt datacenter via tunnel | ✖ N.v.t. — klassieke IPsec site-to-site geschrapt; de ZT-Branch gebruikt in plaats daarvan per-apparaat NetBird-inschrijving |
| 8 | QoS-markering zichtbaar op VyOS | ✅ Gevalideerd — DSCP-markering zichtbaar via `tc` op VyOS eth0 (V43 Test #5: EF geclassificeerd, 0 drops onder last) |
| 9 | Managementdashboard toont alle services UP | ✅ Operationeel (Wazuh SIEM geïmplementeerd) |

Stappen 1–5, 8 en 9 weerspiegelen de huidige stack. Stap 6 is gevalideerd bij opbouw (F4), maar het DC-LAN-over-overlay-pad is verwijderd in V34 (uitgesteld). Stap 7 is architectureel N.v.t. — klassiek IPsec SD-WAN, vervangen door het ZT-Branch-model.

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
