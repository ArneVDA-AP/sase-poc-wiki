---
title: "Concept: DLP — Data Loss Prevention"
tags: [dlp, icap, clamav, python, squid, sase]
---

# Concept: DLP — Data Loss Prevention

**Definitie:** Controles die de exfiltratie van gevoelige gegevens detecteren en blokkeren — creditcardnummers, IBAN's, nationale ID's, vertrouwelijkheidsmarkeringen — terwijl ze een inspectiepunt passeren.

## Hoe het hier van toepassing is

DLP werkt op twee lagen in deze stack, elk met een andere verkeersrichting en inhoudstype:

**Laag 1 — Downloads (ClamAV RESPMOD):**  
ClamAV onderschept HTTP-antwoorden (downloads, webpagina-bodies) via ICAP RESPMOD. Het gebruikt:
- `StructuredDataDetection` — Luhn-gevalideerde creditcardnummers (drempel: 3+ per bestand)
- YARA-regels — IBAN-patronen, BSN 9-cijferige clusters, AWS-toegangssleutelformaat, vertrouwelijkheidsmarkeringen (CONFIDENTIAL, VERTROUWELIJK, GEHEIM, DO NOT DISTRIBUTE)

De bestandsdecompositiemotor van ClamAV pakt containers uit vóór scanning — een YARA-regel die overeenkomt met "VERTROUWELIJK" zal activeren op een `.docx`-bestand omdat ClamAV de ZIP uitpakt en `word/document.xml` direct scant.

**Laag 2 — Uploads (Python DLP REQMOD):**  
De Python DLP-server onderschept POST/PUT/PATCH-aanvraagbodies via ICAP REQMOD. Het parseert `multipart/form-data` correct (wat ClamAV niet kan) en past algoritmische validators toe:
- Creditcards — Luhn-algoritme (valideert het controlecijfer, niet alleen het patroonformaat)
- IBAN — mod-97-controlesomvalidatie
- BSN (burgerservicenummer) — 11-proefvalidatie
- AWS-toegangssleutels — `AKIA[A-Z0-9]{16}` regex

Wanneer een overeenkomst de drempel overschrijdt, retourneert de server een blokantwoord. Squid toont een gestijlde DLP-blokpagina die verschilt van zijn generieke 403-pagina.

## Waar het in de stack voorkomt

**[ClamAV/c-icap](../components/clamav-cicap.md)** — DLP Laag 1, downloadpad. YARA-regels in `/var/db/clamav/dlp_custom.yar`. StructuredDataDetection in `/usr/local/etc/clamd.conf`. ICAP RESPMOD op `127.0.0.1:1344`.

**[Python DLP](../components/python-dlp.md)** — DLP Laag 2, uploadpad. Docker-container op mgmt01, ICAP REQMOD op `192.168.122.23:1345`. Squid routeert alleen POST/PUT/PATCH naar deze service (GET-aanvragen bevatten geen door gebruikers geüploade inhoud).

**[Squid](../components/squid.md)** — de ICAP-client die beide lagen orkestreert. Routeert aanvragen naar Python DLP (REQMOD) en antwoorden naar ClamAV (RESPMOD).

## Belangrijke onderscheidingen

**Algoritmische validatie vs patroonherkenning:** YARA-regels in ClamAV matchen formaatpatronen — ze detecteren *kandidaat*-IBAN's of BSN's maar valideren geen controlesommen. De Python DLP-server past volledige algoritmische validatie toe (mod-97, 11-proef, Luhn). Valse-positiefpercentages verschillen significant: ClamAV Laag 1 kan sommige accidentele overeenkomsten markeren; Python DLP Laag 2 markeert alleen wiskundig geldige waarden.

**Upload vs download DLP:** De twee lagen bestrijken verschillende bedreigingsvectoren. Het downloaden van een bestand met gevoelige gegevens (bijv. een bedrijfsdocument per ongeluk geüpload naar een publieke share) wordt gedekt door ClamAV. Het actief exfiltreren van gegevens via een webformulierupload of API-aanroep wordt gedekt door de Python DLP-server.

**`bypass=on` voor Python DLP:** Squid is geconfigureerd met `bypass=on` voor de Python DLP-service. Als de mgmt01-container onbereikbaar is, gaan uploads door. ClamAV-scanning (downloads) wordt niet omzeild. Dit is een bewuste beschikbaarheidsafweging: DLP-storingen mogen legitiem bedrijfsverkeer niet blokkeren.

**ClamAV kan multipart/form-data niet parsen:** De c-icap `virus_scan`-service ontvangt POST-bodies als ruwe bytes. Zonder correcte multipart-parsing is DLP-patroonherkenning op veldwaarden in formulierverzendingen onbetrouwbaar. Dit is de kernreden waarom een aparte Python ICAP-server bestaat. Zie [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md).

## Bronnen

- `raw/Doc2_ClamAV_DLP_Pipeline.md` (volledige DLP-pipeline-architectuur)
- `raw/Verslag21.md` (beslissing om Python DLP-server toe te voegen)
