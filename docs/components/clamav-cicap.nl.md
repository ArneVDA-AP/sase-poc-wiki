---
title: "ClamAV + c-icap — Malware-scanning en DLP-laag 1"
tags: [clamav, c-icap, icap, antivirus, yara, dlp, respmod, opnsense, squid]
---

# ClamAV + c-icap — Malware-scanning en DLP-laag 1

**Rol:** ICAP RESPMOD-service op pop01 — scant alle HTTP-responses (downloads) op malware-handtekeningen, DLP-gevoelige inhoud (creditcards via StructuredDataDetection) en aangepaste patronen (YARA-regels voor IBAN, BSN, CONFIDENTIAL-labels en AWS-sleutels).  
**Versie:** ClamAV 1.x (meegeleverd met OPNsense `os-clamav`), c-icap (meegeleverd met `os-c-icap`)  
**Configuratielocatie:** `/usr/local/etc/clamd.conf` (ClamAV), `/var/log/cicap/cicap_JJJJMMDD.log` (c-icap-logs)

---

## Werking in deze stack

ClamAV draait als daemon (`clamd`) op pop01 en ontvangt bestanden van c-icap voor scanning. c-icap is het ICAP-serverproces waarmee Squid communiceert. Wanneer Squid een response van het internet ontvangt (een gedownload bestand, een webpagina-body), stuurt het een kopie naar c-icap via ICAP RESPMOD. c-icap extraheert de body, geeft deze door aan ClamAV voor scanning en retourneert ofwel `ICAP/1.0 204 No Content` (schoon — Squid laat het door) of een doorverwijzing naar een blokpagina (infectie/DLP-overeenkomst — Squid retourneert 403).

**Waarom RESPMOD en niet REQMOD voor ClamAV:** De `virus_scan`-service van ClamAV verwerkt HTTP-responsebodies correct. POST-verzoekbodies (formuliergegevens, bestandsuploads) gebruiken `multipart/form-data`-codering, die c-icap's `virus_scan` niet correct parseert — geüploade inhoud wordt als onbewerkte bytes aan ClamAV doorgegeven, waardoor DLP-patroonherkenning onbetrouwbaar is. Voor upload-scanning, zie [Python DLP](python-dlp.md). Zie [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md).

**DLP-laag 1 — wat ClamAV detecteert:**
- **Malware-handtekeningen** — 3,6 miljoen+ uit ClamAV's officiële databases (daily, main, bytecode)
- **StructuredDataDetection (SDD)** — Luhn-gevalideerde creditcarddetectie (drempelwaarde: 3+ geldige kaartnummers per bestand)
- **YARA-regels** — vier aangepaste regels geladen uit `/var/db/clamav/dlp_custom.yar`:
  - `DLP_Confidential_Label` — "CONFIDENTIAL", "VERTROUWELIJK", "GEHEIM", "DO NOT DISTRIBUTE" (hoofdletterongevoelig)
  - `DLP_IBAN_Pattern` — NL-, DE- en BE-IBAN-formaatpatronen
  - `DLP_BSN_Candidate` — 9-cijferige reeksen in clusters (drempelwaarde >2)
  - `DLP_AWS_AccessKey` — patroon `AKIA[A-Z0-9]{16}`

YARA-regels worden uitgevoerd *na* de bestandsdecompressie-engine van ClamAV — een regel die overeenkomt met "CONFIDENTIAL" wordt geactiveerd op een `.docx`-bestand omdat ClamAV de ZIP-container uitpakt en de binnenste `word/document.xml` scant. Dit is een significant voordeel ten opzichte van het scannen van onbewerkte bytes.

---

## Configuratie

### ClamAV + c-icap inschakelen

Installeren via OPNsense → Systeem → Firmware → Plugins: `os-clamav` en `os-c-icap`.

Configureren via Services → ClamAV Antivirus:
- ClamAV inschakelen: ✔
- Scannen op bestandstypen: Tekstbestanden, Binaire bestanden, Uitvoerbare bestanden, Archieven, GIF, JPEG, Office
- 204-responses toestaan: ✔ (prestaties — geen kopie van body bij schoon resultaat)

De GUI heeft **geen** "Scanmodus"- of "Poort"-veld (in tegenstelling tot oudere handleidingbeschrijvingen). Modus wordt geconfigureerd aan de Squid-kant; poort 1344 is de vaste standaard.

### Squid ICAP-integratie

Via Services → Squid Web Proxy → Antivirus (ICAP):
- ICAP-host: `127.0.0.1`
- ICAP-poort: `1344`
- ICAP-service: `virus_scan`

**Belangrijk:** Na elke ICAP-configuratiewijziging `configctl proxy restart` uitvoeren — een GUI-"Toepassen" alleen activeert ICAP niet in de actieve Squid-daemon.

### StructuredDataDetection

In `/usr/local/etc/clamd.conf`:

```ini
StructuredDataDetection yes
StructuredMinCreditCardCount 3
StructuredMinSSNCount 3
StructuredSSNFormatNormal yes
StructuredCCOnly no
```

Herladen zonder volledige herstart:
```bash
configctl clamav stop
configctl clamav start
```

### YARA-regels

```bash
cat > /var/db/clamav/dlp_custom.yar << 'EOF'
rule DLP_Confidential_Label {
    strings:
        $s1 = "CONFIDENTIAL" nocase
        $s2 = "VERTROUWELIJK" nocase
        $s3 = "GEHEIM" nocase
        $s4 = "DO NOT DISTRIBUTE" nocase
    condition:
        any of them
}
rule DLP_IBAN_Pattern {
    strings:
        $iban_nl = /NL\d{2}[A-Z]{4}\d{10}/
        $iban_de = /DE\d{2}\d{18}/
        $iban_be = /BE\d{2}\d{12}/
    condition:
        any of them
}
rule DLP_BSN_Candidate {
    strings:
        $bsn = /\b\d{9}\b/
    condition:
        #bsn > 2
}
rule DLP_AWS_AccessKey {
    strings:
        $ak = /AKIA[0-9A-Z]{16}/
    condition:
        $ak
}
EOF
clamdscan --reload
```

YARA-bestanden in `/var/db/clamav/` worden automatisch geladen door ClamAV. Overeenkomsten verschijnen met het `.UNOFFICIAL`-achtervoegsel in de logs — dit is normaal voor aangepaste regels.

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Squid](squid.md) | ICAP RESPMOD ← | Squid stuurt alle HTTP-responses voor scanning |
| [Python DLP](python-dlp.md) | aanvullend | Python DLP verwerkt uploads (REQMOD); ClamAV verwerkt downloads (RESPMOD) |
| [Suricata](suricata.md) | parallel | Beide draaien op pop01; geen interactie — verschillende inspectiedomeinen |

**ICAP is Squid-globaal:** De ICAP-serviceconfiguratie is automatisch van toepassing op alle Squid-listeners — zowel de LAN-listener als de pre-auth NetBird-overlay-listener. Per-listener ICAP-configuratie is niet nodig. Dit verschilt van SSL Bump, dat per `http_port`-directive moet worden opgegeven.

---

## Bekende problemen / valkuilen

**GUI-timeout bij eerste database-download** — de ClamAV-handtekeningdownload (~300 MB) duurt langer dan de GUI-timeout. De download gaat op de achtergrond door. Controleer met `freshclam --verbose`.

**`database.clamav.net` retourneert 403 bij directe HTTP-verzoeken** — het CDN vereist de freshclam-useragent. Een 403 van de server bevestigt dat TCP/TLS correct werkt; het is geen firewallprobleem.

**Opstaartwaarschuwingen van c-icap zijn cosmetisch** — berichten zoals `WARNING: Can not check the used c-icap release to build service virus_scan.so` hebben alleen betrekking op verificatie van de buildversie, niet op de runtimefunctionaliteit.

**c-icap-logpad is niet voor de hand liggend** — logs staan in `/var/log/cicap/cicap_JJJJMMDD.log`, niet in `/var/log/c-icap/server.log`.

**YARA-regels valideren niet algoritmisch** — `DLP_IBAN_Pattern` herkent het formaat maar voert geen mod-97-controlesomvalidatie uit. `DLP_BSN_Candidate` herkent 9-cijferige clusters maar voert geen 11-proef uit. Valse positieven zijn mogelijk. De [Python DLP-server](python-dlp.md) op het uploadpad voert wel algoritmische validatie uit.

**`--ssl-no-revoke` vereist voor curl.exe-tests** — curl.exe op Windows gebruikt de schannel TLS-stack, die CRL/OCSP-verificatie probeert op het SSL Bump-certificaat. Zelfondertekende CA's hebben geen intrekkingseindpunt, waardoor dit mislukt. Voeg `--ssl-no-revoke` toe aan alle CLI-testopdrachten.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Component: Squid](squid.md)
- [Component: Python DLP](python-dlp.md)
- [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md)
