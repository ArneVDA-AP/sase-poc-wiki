---
title: "Concept: ICAP — Internet Content Adaptation Protocol"
tags: [icap, squid, clamav, c-icap, dlp, reqmod, respmod, proxy]
---

# Concept: ICAP — Internet Content Adaptation Protocol

**Definitie:** Een lichtgewicht protocol (RFC 3507) waarmee een proxy inhoudinspectie kan uitbesteden aan externe services — Squid stuurt HTTP-transacties naar ICAP-servers, die een verdict teruggeven (doorlaten, aanpassen of blokkeren).

## Hoe het hier van toepassing is

Squid op pop01 kan HTTPS-inhoud niet op applicatielaagniveau inspecteren zonder hulp. ICAP is het integratieprotocol dat Squid verbindt met twee externe inspectiediensten:

```
BYOD-client → Squid (SSL Bump decodeert) → ICAP REQMOD → Python DLP (upload-controle)
                                          → ICAP RESPMOD → ClamAV/c-icap (download-scan)
```

Beide ICAP-services draaien gelijktijdig en onafhankelijk. ICAP REQMOD verwerkt de aanvraag voordat Squid die doorstuurt; ICAP RESPMOD verwerkt het antwoord voordat Squid het aan de client levert.

## Waar het in de stack voorkomt

**[Squid](../components/squid.md)** — de ICAP-client. Squid routeert:
- POST/PUT/PATCH-aanvragen → ICAP REQMOD → `icap://192.168.122.23:1345/dlpscan` (Python DLP)
- Alle antwoorden → ICAP RESPMOD → `icap://127.0.0.1:1344/virus_scan` (ClamAV/c-icap)

**[ClamAV/c-icap](../components/clamav-cicap.md)** — ICAP RESPMOD-server. c-icap ontvangt HTTP-antwoordbodies van Squid, geeft ze door aan ClamAV voor malware + DLP-scanning, en retourneert `204 No Content` (schoon) of een omleiding naar een blokpagina (gedetecteerd).

**[Python DLP](../components/python-dlp.md)** — ICAP REQMOD-server. Ontvangt POST/PUT/PATCH-aanvraagbodies, parseert multipart/form-data correct, past algoritmische DLP-validators toe (Luhn, mod-97, 11-proef, regex) en retourneert een blokantwoord of `200 OK`.

## Belangrijke onderscheidingen

**REQMOD vs RESPMOD:**

| | REQMOD | RESPMOD |
|--|--------|---------|
| Timing | Voordat aanvraag upstream wordt doorgestuurd | Nadat antwoord van upstream aankomt |
| Payload | HTTP-aanvraag (headers + body) | HTTP-antwoord (headers + body) |
| Gebruik | Upload DLP — wat stuurt de gebruiker | Malwarescanning, download DLP |
| In deze stack | Python DLP op mgmt01:1345 | ClamAV/c-icap op pop01:1344 |

**Waarom twee ICAP-servers:** ClamAV's c-icap `virus_scan`-service parseert `multipart/form-data`-aanvraagbodies niet correct — POST-inhoud arriveert als ruwe bytes, waardoor DLP-patroonherkenning op formuliervelden onbetrouwbaar is. De Python DLP-server is specifiek toegevoegd om upload-inspectie te verwerken met correcte multipart-parsing. Zie [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md).

**ICAP is Squid-globaal:** De ICAP-serviceconfiguratie in Squid is van toepassing op alle listeners automatisch — zowel de LAN-listener (`10.0.0.1:3128`) als de NetBird-overlay-listener (`100.70.154.79:3128`). Dit verschilt van SSL Bump, dat per `http_port`-directive gespecificeerd moet worden.

**`bypass=on` voor de Python DLP-service:** Als de DLP-container onbereikbaar is, laat Squid verkeer door zonder inspectie (fail-open). ClamAV-scanning (RESPMOD) blijft actief. Een DLP-storing blokkeert niet al het verkeer — een bewuste beschikbaarheidsafweging.

**`configctl proxy restart` vereist:** Een Squid GUI "Apply" alleen activeert ICAP-wijzigingen niet in de draaiende daemon. Herstart altijd via `configctl proxy restart` na het aanpassen van de ICAP-configuratie.

## Bronnen

- `raw/Doc1_Squid_WPAD_PAC.md` §2 (Squid ICAP-orchestratie)
- `raw/Doc2_ClamAV_DLP_Pipeline.md` (ClamAV RESPMOD, waarom REQMOD niet werkt voor uploads)
- `raw/Verslag21.md` (Python DLP-beslissing, twee-laags architectuur)
