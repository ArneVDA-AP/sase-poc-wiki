---
title: "Python DLP ICAP-server — Upload-DLP-laag 2"
tags: [python, dlp, icap, reqmod, docker, squid, multipart]
---

# Python DLP ICAP-server — Upload-DLP-laag 2

**Rol:** ICAP REQMOD-service op mgmt01 — scant HTTP-verzoekbodies (uploads, formulierinzendingen) op gevoelige gegevens: creditcardnummers (Luhn), IBAN (mod-97), BSN (11-proef), AWS-toegangssleutels. Blokkeert POST/PUT/PATCH-verzoeken die overeenkomsten bevatten.  
**Versie:** Aangepaste Python 3.11, pyicap (gepatcht), Dockerfile op mgmt01  
**Configuratielocatie:** `/opt/dlp-icap/` op mgmt01, `/usr/local/etc/squid/pre-auth/dlp-icap.conf` op pop01

---

## Werking in deze stack

De Python DLP-server draait als Docker-container op mgmt01 en luistert op `192.168.122.23:1345`. Squid op pop01 routeert POST-, PUT- en PATCH-verzoeken naar deze server via ICAP REQMOD voordat ze naar het internet worden doorgestuurd.

De server ontvangt de volledige HTTP-verzoekbody, parseert `multipart/form-data`-inhoud correct (een mogelijkheid die ClamAV/c-icap mist) en past algoritmische validators toe:
- **Creditcards** — Luhn-algoritme (valideert controlecijfer, niet alleen formaat)
- **IBAN** — mod-97-controlesomvalidatie
- **BSN** (Nederlands burgerservicenummer) — 11-proefvalidatie
- **AWS-toegangssleutels** — regexpatroon `AKIA[A-Z0-9]{16}`

Als een overeenkomst boven de drempelwaarde wordt gevonden, retourneert de server een ICAP-blokkeerrespons. Squid toont een opgemaakte DLP-blokpagina aan de client (onderscheiden van de generieke Squid 403-pagina).

**Waarom een aparte server voor upload-DLP:** De `virus_scan` c-icap-service van ClamAV parseert `multipart/form-data` of URL-gecodeerde POST-bodies niet correct. De c-icap-ontwikkelaar bevestigde deze beperking. POST-gegevens worden als onbewerkte bytes doorgegeven, waardoor DLP-patroonherkenning voor geüploade inhoud onbetrouwbaar is. Zie [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md).

---

## Configuratie

### Dockerfile

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir pyicap python-docx openpyxl pypdf && \
    sed -i 's/import collections/import collections; import collections.abc/' \
        /usr/local/lib/python3.11/site-packages/pyicap.py && \
    sed -i 's/collections\.Callable/collections.abc.Callable/g' \
        /usr/local/lib/python3.11/site-packages/pyicap.py

COPY dlp_server.py /app/dlp_server.py
WORKDIR /app
CMD ["python", "dlp_server.py"]
```

De `sed`-patches zijn vereist omdat pyicap op PyPI `collections.Callable` gebruikt, dat in Python 3.10+ is verwijderd. Zonder de patch crasht de container bij elk ICAP-verzoek. Zie [Bevinding: pyicap collections-bug](../findings/pyicap-collections-bug.md).

### Squid-integratie (pre-auth include)

Bestand: `/usr/local/etc/squid/pre-auth/dlp-icap.conf`

```squid
icap_service svc_dlp_req reqmod_precache bypass=on icap://192.168.122.23:1345/dlpscan

acl dlp_methods method POST PUT PATCH

adaptation_access svc_dlp_req allow dlp_methods localnet
adaptation_access svc_dlp_req allow dlp_methods subnets
adaptation_access svc_dlp_req deny all
```

Belangrijke ontwerpkeuzes:
- `bypass=on` — als de DLP-container niet beschikbaar is, passeert verkeer zonder inspectie (fail-open). ClamAV-malwarescanning blijft actief. Een DLP-storing blokkeert geen legitiem zakelijk verkeer.
- Alleen POST/PUT/PATCH — GET-verzoeken bevatten geen uploads; ze door DLP routeren voegt alleen latentie toe zonder voordeel.
- Pre-auth include — overleeft GUI-wijzigingen en Squid-updates.

Na het aanmaken van dit bestand: `configctl proxy restart`.

### Verbindingscontrole vanaf pop01

```bash
printf "OPTIONS icap://192.168.122.23:1345/dlpscan ICAP/1.0\r\nHost: 192.168.122.23\r\n\r\n" \
  | nc -w 5 192.168.122.23 1345
# Verwacht: ICAP/1.0 200 OK, Methods: REQMOD
```

Gebruik `printf`, niet `echo -e` — FreeBSD's `/bin/sh` interpreteert `-e` niet als escape-vlag, waardoor `\r\n` letterlijk wordt verzonden.

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Squid](squid.md) | ICAP REQMOD ← | Squid stuurt POST/PUT/PATCH-bodies vóór doorsturen |
| [ClamAV/c-icap](clamav-cicap.md) | aanvullend | ClamAV verwerkt downloads (RESPMOD); Python verwerkt uploads (REQMOD) |
| mgmt01 Docker | runtime-afhankelijkheid | Container moet actief zijn; `restart: unless-stopped` in compose |

**Netwerkpad:** pop01 → mgmt01 via `192.168.122.0/24` (WAN/beheersegment). Dit pad bestaat onafhankelijk van de NetBird-overlay. De DLP-server functioneert ook wanneer de NetBird-tunnel niet actief is.

---

## Bekende problemen / valkuilen

**pyicap Python 3.10+-incompatibiliteit** — zie [Bevinding: pyicap collections-bug](../findings/pyicap-collections-bug.md). De Dockerfile patcht dit automatisch.

**Client-IP verschijnt als "unknown" in DLP-logs** — de server leest `X-Client-IP` uit `self.enc_req_headers` (ingekapselde headers), maar Squid stuurt het client-IP als een ICAP-niveauheader toegankelijk via `self.headers`. Dit is een cosmetisch probleem; blokkering werkt correct.

**Containerlogs in Docker stdout** — voor Wazuh SIEM-integratie, voeg een Docker-logging-driver toe (syslog gericht op Wazuh) of configureer de Wazuh-agent op mgmt01 om containerlogs te bewaken. Dit is een geplande taak voor Fase 4.

**Algoritmische validators hebben drempelwaarden** — één creditcardnummer in een POST-body activeert standaard geen blokkering. Pas drempelwaarden aan in `dlp_server.py` op basis van de gebruikssituatie.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Component: Squid](squid.md)
- [Component: ClamAV/c-icap](clamav-cicap.md)
- [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md)
- [Bevinding: pyicap collections-bug](../findings/pyicap-collections-bug.md)
