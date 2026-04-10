---
title: "Concept: DLP — Data Loss Prevention"
tags: [dlp, icap, clamav, python, squid, sase]
---

# Concept: DLP — Data Loss Prevention

**One-line definition:** Controls that detect and block the exfiltration of sensitive data — credit card numbers, IBANs, national IDs, confidential document labels — as it passes through an inspection point.

## How it applies here

DLP operates at two layers in this stack, each covering a different traffic direction and content type:

**Layer 1 — Downloads (ClamAV RESPMOD):**  
ClamAV intercepts HTTP responses (downloads, web page bodies) via ICAP RESPMOD. It uses:
- `StructuredDataDetection` — Luhn-validated credit card numbers (threshold: 3+ per file)
- YARA rules — IBAN patterns, BSN 9-digit clusters, AWS access key format, confidentiality labels (CONFIDENTIAL, VERTROUWELIJK, GEHEIM, DO NOT DISTRIBUTE)

ClamAV's file decomposition engine unpacks containers before scanning — a YARA rule matching "CONFIDENTIAL" will fire on a `.docx` file because ClamAV unpacks the ZIP and scans `word/document.xml` directly.

**Layer 2 — Uploads (Python DLP REQMOD):**  
The Python DLP server intercepts POST/PUT/PATCH request bodies via ICAP REQMOD. It correctly parses `multipart/form-data` (which ClamAV cannot) and applies algorithmic validators:
- Credit cards — Luhn algorithm (validates the check digit, not just pattern format)
- IBAN — mod-97 checksum validation
- BSN (Dutch citizen ID) — 11-proof validation
- AWS access keys — `AKIA[A-Z0-9]{16}` regex

When a match exceeds the threshold, the server returns a block response. Squid presents a styled DLP block page distinct from its generic 403 page.

## Where it appears in the stack

**[ClamAV/c-icap](../components/clamav-cicap.md)** — DLP Layer 1, download path. YARA rules in `/var/db/clamav/dlp_custom.yar`. StructuredDataDetection in `/usr/local/etc/clamd.conf`. ICAP RESPMOD at `127.0.0.1:1344`.

**[Python DLP](../components/python-dlp.md)** — DLP Layer 2, upload path. Docker container on mgmt01, ICAP REQMOD at `192.168.122.23:1345`. Squid only routes POST/PUT/PATCH to this service (GET requests carry no user-uploaded content).

**[Squid](../components/squid.md)** — the ICAP client that orchestrates both layers. Routes requests to Python DLP (REQMOD) and responses to ClamAV (RESPMOD).

## Key distinctions

**Algorithmic validation vs pattern matching:** YARA rules in ClamAV match format patterns — they detect *candidate* IBANs or BSNs but do not validate checksums. The Python DLP server applies full algorithmic validation (mod-97, 11-proof, Luhn). False positive rates differ significantly: ClamAV Layer 1 may flag some accidental matches; Python DLP Layer 2 only flags mathematically valid values.

**Upload vs download DLP:** The two layers cover different threat vectors. Downloading a file containing sensitive data (e.g., a company document accidentally uploaded to a public share) is covered by ClamAV. Actively exfiltrating data via a web form upload or API call is covered by the Python DLP server.

**`bypass=on` for Python DLP:** Squid is configured with `bypass=on` for the Python DLP service. If the mgmt01 container is unreachable, uploads pass through. ClamAV scanning (downloads) is not bypassed. This is a deliberate availability trade-off: DLP failures should not block legitimate business traffic.

**ClamAV cannot parse multipart/form-data:** The c-icap `virus_scan` service receives POST bodies as raw bytes. Without proper multipart parsing, DLP pattern matching against field values in form submissions is unreliable. This is the core reason a separate Python ICAP server exists. See [Decision: Two-layer DLP](../decisions/two-layer-dlp.md).

## Sources

- `raw/Doc2_ClamAV_DLP_Pipeline.md` (full DLP pipeline architecture)
- `raw/Verslag21.md` (decision to add Python DLP server)
