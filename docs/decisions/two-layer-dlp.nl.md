---
title: "Beslissing: Twee-laags DLP-architectuur"
tags: [decision, dlp, icap, clamav, python, squid]
---

# Beslissing: Twee-laags DLP-architectuur

**Status:** Geïmplementeerd  
**Datum:** Maart/april 2026 (Verslag21)

## Context

De ICAP-pipeline moet zowel uploads (POST-aanvraagbodies) als downloads (HTTP-antwoordbodies) inspecteren op gevoelige gegevens. De aanvankelijke aanpak was ClamAV/c-icap te gebruiken voor beide richtingen. ClamAV biedt StructuredDataDetection (Luhn-gevalideerde creditcards) en YARA-regels — beide nuttig voor DLP.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Alleen ClamAV (beide richtingen)** | Één component, geen extra service | c-icap `virus_scan` parseert `multipart/form-data` niet correct — POST-bodies arriveren als ruwe bytes, waardoor DLP-matching onbetrouwbaar is |
| **ClamAV (RESPMOD) + Python DLP (REQMOD)** | Python kan multipart correct parsen; algoritmische validatie (Luhn, mod-97, 11-proef) | Extra Docker-container op mgmt01; meer bewegende onderdelen |
| **Alleen Python DLP** | Één server verwerkt beide richtingen | Zou ook malware-scanlogica moeten repliceren |

## Beslissing

Twee-laags architectuur: ClamAV/c-icap als ICAP RESPMOD voor downloads, Python DLP-server als ICAP REQMOD voor uploads.

De c-icap-ontwikkelaar bevestigde dat `virus_scan` `multipart/form-data` niet correct parseert — geüploade inhoud wordt als ruwe bytes doorgegeven aan ClamAV. Dit maakt DLP-patroonherkenning op webformulieruploads onbetrouwbaar. De Python DLP-server is specifiek geschreven om dit geval te verwerken: hij implementeert correcte multipart-parsing en past algoritmische validators toe (Luhn voor creditcards, mod-97 voor IBAN, 11-proef voor BSN).

## Gevolgen

- Python DLP-container (`/opt/dlp-icap/` op mgmt01) moet actief zijn voor upload-DLP om te functioneren
- `bypass=on` in Squid's ICAP-configuratie voor de Python DLP-service: als de container uitvalt, gaan uploads door (fail-open). ClamAV blijft actief voor downloadscanning
- Alleen POST/PUT/PATCH-aanvragen worden naar Python DLP gerouteerd — GET-aanvragen bevatten geen door gebruikers geüploade inhoud en ze toevoegen verhoogt alleen de latentie
- ClamAV YARA-regels bieden download-DLP (matching na bestandsdecompositie, dus `.docx`-uitpakken werkt); Python DLP biedt upload-DLP met volledige algoritmische validatie
- pyicap-bibliotheek vereist een Python 3.10+-compatibiliteitspatch in de Dockerfile — zie [Bevinding: pyicap collections bug](../findings/pyicap-collections-bug.md)

## Gerelateerd

- [Component: ClamAV/c-icap](../components/clamav-cicap.md)
- [Component: Python DLP](../components/python-dlp.md)
- [Component: Squid](../components/squid.md)
- [Concept: DLP](../concepts/dlp.md)
- [Concept: ICAP](../concepts/icap.md)
