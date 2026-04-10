---
title: "Decision: Two-Layer DLP Architecture"
tags: [decision, dlp, icap, clamav, python, squid]
---

# Decision: Two-Layer DLP Architecture

**Status:** Implemented  
**Date:** March/April 2026 (Verslag21)

## Context

The ICAP pipeline needs to inspect both uploads (POST request bodies) and downloads (HTTP response bodies) for sensitive data. The initial approach was to use ClamAV/c-icap for both directions. ClamAV provides StructuredDataDetection (Luhn-validated credit cards) and YARA rules — both useful for DLP.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **ClamAV only (both directions)** | Single component, no extra service | c-icap `virus_scan` does not correctly parse `multipart/form-data` — POST bodies arrive as raw bytes, making DLP matching unreliable |
| **ClamAV (RESPMOD) + Python DLP (REQMOD)** | Python can parse multipart correctly; algorithmic validation (Luhn, mod-97, 11-proof) | Additional Docker container on mgmt01; more moving parts |
| **Python DLP only** | One server handles both directions | Would need to also replicate malware scanning logic |

## Decision

Two-layer architecture: ClamAV/c-icap as ICAP RESPMOD for downloads, Python DLP server as ICAP REQMOD for uploads.

The c-icap developer confirmed that `virus_scan` does not correctly parse `multipart/form-data` — uploaded content is passed as raw bytes to ClamAV. This makes DLP pattern matching on web form uploads unreliable. The Python DLP server was written specifically to handle this case: it implements proper multipart parsing and applies algorithmic validators (Luhn for credit cards, mod-97 for IBAN, 11-proof for BSN).

## Consequences

- Python DLP container (`/opt/dlp-icap/` on mgmt01) must be running for upload DLP to function
- `bypass=on` in Squid's ICAP config for the Python DLP service: if the container is down, uploads pass through (fail-open). ClamAV remains active for download scanning
- Only POST/PUT/PATCH requests are routed to Python DLP — GET requests carry no user-uploaded content and adding them only increases latency
- ClamAV YARA rules provide download DLP (matching after file decomposition, so `.docx` unpacking works); Python DLP provides upload DLP with full algorithmic validation
- pyicap library requires a Python 3.10+ compatibility patch in the Dockerfile — see [Finding: pyicap collections bug](../findings/pyicap-collections-bug.md)

## Related

- [Component: ClamAV/c-icap](../components/clamav-cicap.md)
- [Component: Python DLP](../components/python-dlp.md)
- [Component: Squid](../components/squid.md)
- [Concept: DLP](../concepts/dlp.md)
- [Concept: ICAP](../concepts/icap.md)
