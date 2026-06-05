---
title: "Aanvals- & bypass-scenario's (demovalidatie)"
tags: [testing, sase, demo, swg, ztna, casb, fwaas, ids, dlp, rpz]
---

# Aanvals- & bypass-scenario's (demovalidatie)

Deze scenario's valideren elke SASE-pijler door aanvallen of policy bypasses uit te voeren die de stack moet detecteren en blokkeren. Elk scenario verwijst naar een of meer acceptatietests (F1–F15, T-A1–T-A13) beschreven in [Acceptatietests](acceptance-tests.nl.md).

---

## Scenario's per pijler

### ZTNA: Identiteit & tunnel

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| A1 | Niet-ge-enrolld apparaat probeert DC-LAN-toegang | Geen route naar 10.0.0.0/24: destination unreachable | F8 | `ping 10.0.0.100` vanaf niet-ge-enrollde host |
| A2 | NetBird-login via Entra ID | OIDC-flow voltooid, tunnel actief, peer in juiste groep | F1, F2 | `netbird up` gevolgd door browser SSO |
| A3 | Non-compliant device logt in | Conditional Access blokkeert (report-only tot demo) | F3 | device compliance aanpassen in Intune |

### SWG: Webgateway-inspectie

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| B1 | Surfen naar geblokkeerde URL-categorie | Squid 403 ERR_ACCESS_DENIED | F5 | `curl.exe -x http://100.70.154.79:3128 http://gambling.com` |
| B2 | EICAR-malwaredownload | ClamAV blokkeert, Squid 403 (8245 bytes vs 68) | F7 | `curl.exe -x ... --ssl-no-revoke https://secure.eicar.org/eicar.com` |
| B3 | SSL Bump-verificatie | Certificate issuer = SASE-PoC-CA, niet het origineel | F6 | Browser naar google.com, certificaat inspecteren |
| B4 | No-bump-uitzondering (Microsoft-login) | Certificate issuer = Microsoft (origineel) | F6 | Browser naar login.microsoftonline.com, certificaat inspecteren |
| B5 | DLP-upload: creditcardnummer in POST-body | Python DLP ICAP blokkeert, 403 | T-A3 | `curl.exe -x ... -X POST -d "CC: 4532015112830366"` naar httpbin |
| B6 | DLP-download: CONFIDENTIAL-label | ClamAV YARA blokkeert | T-A1 | Bestand downloaden met CONFIDENTIAL-markering |
| B7 | DLP-threshold: 4x CC-nummers blokkeren, 1x doorlaten | SDD-threshold enforcement | T-A2 | POST met 4 vs 1 creditcardnummers |

### CASB: Identiteitsgebaseerde filtering

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| C1 | Student surft naar deepai.org | Geblokkeerd, 403 (Studenten-policy) | CASB L1 | `curl.exe -x ... https://deepai.org` als student |
| C2 | Docent surft naar deepai.org | Toegestaan, 200 (Docenten-policy) | CASB L1 | Zelfde URL als docentidentiteit |
| C3 | SharePoint anonieme deellink | Gedetecteerd (regel 100601); Active Response trekt link in achter de ENFORCE-gate (standaard detect-only, live revoke nog niet actief) | CASB L2 | Anonieme deellink aanmaken in SharePoint |

### FWaaS / IDS: Netwerkdetectie

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| D1 | testmyids.com attack response | Suricata SID 2100498 (GPL ATTACK_RESPONSE) | F9 | `curl.exe -x ... http://testmyids.com/` |
| D2 | Verdachte User-Agent "BlackSun" | Suricata SID 2008983 (ET MALWARE) | T-A8 | `curl.exe -x ... -A "BlackSun" http://example.com` |
| D3 | DNS-anomaliedetectie | Suricata SID 2027863 | T-A9 | Automatisch via .biz TLD-queries |
| D4 | DC-LAN-verkeersdetectie (vtnet1) | Suricata-alert op vtnet1 | T-A7 | `apt update` op dc01 |

### DNS Threat Intelligence

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| E1 | RPZ-geblokkeerd domein queryen vanaf pop01 | NXDOMAIN met aa-vlag | T-A4 | `dig @127.0.0.1 <rpz-domain>` op pop01 |
| E2 | RPZ-geblokkeerd domein queryen vanaf mobile01 | NXDOMAIN via overlay DNS | T-A5 | `nslookup <rpz-domain>` op mobile01 |
| E3 | RPZ-geblokkeerd domein queryen vanaf dc01 | NXDOMAIN via DC-LAN | T-A6 | `dig @10.0.0.1 <rpz-domain>` op dc01 |

### Eventgestuurde handhaving (NATS)

| # | Scenario | Verwacht resultaat | Valideert | Testcommando |
|---|----------|--------------------|-----------|--------------|
| F1 | Malware-event met hoge ernst | Control daemon plaatst peer in quarantaine (binnen enkele seconden) | CASB L3 | c-icap RESPMOD-malwaredetectie triggeren |
| F2 | Meerdere medium-alerts (scoreopbouw) | Dreigingsscore stijgt, overschrijdt threshold, triggert quarantaine | CASB L3 | Opeenvolgende alerts van dezelfde client |
| F3 | Scoreverval na quarantaine | Peer hersteld naar personagroep na vervalperiode | CASB L3 | Wachten op sliding-window-verval |

---

## Belangrijke testopmerkingen

- Alle proxytests vanaf Windows vereisen `--ssl-no-revoke` wegens Schannel CRL-controlefout op SASE-PoC-CA (zie [Finding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.nl.md)).
- ICMP-ping naar overlay-peers faalt by design onder het ALLOW-only policy model. Gebruik altijd poortgebaseerde tests.
- Elke Suricata SID vuurt eenmaal per TCP-flow door connection pooling (zie [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.nl.md)).

---

## Gerelateerd

- [Acceptatietests (F1–F15)](acceptance-tests.nl.md)
- [Architectuur](../overview/architecture.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Finding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.nl.md)
- [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.nl.md)
