---
title: "Concept: ICAP — Internet Content Adaptation Protocol"
tags: [icap, squid, clamav, c-icap, dlp, reqmod, respmod, proxy]
---

# Concept: ICAP — Internet Content Adaptation Protocol

**One-line definition:** A lightweight protocol (RFC 3507) that lets a proxy offload content inspection to external services — Squid sends HTTP transactions to ICAP servers, which return a verdict (pass, modify, or block).

## How it applies here

Squid on pop01 cannot inspect HTTPS content at the application layer without help. ICAP is the integration protocol that connects Squid to two external inspection services:

```
BYOD client → Squid (SSL Bump decrypts) → ICAP REQMOD → Python DLP (upload check)
                                        → ICAP RESPMOD → ClamAV/c-icap (download scan)
```

Both ICAP services run concurrently and independently. ICAP REQMOD processes the request before Squid forwards it; ICAP RESPMOD processes the response before Squid delivers it to the client.

## Where it appears in the stack

**[Squid](../components/squid.md)** — the ICAP client. Squid routes:
- POST/PUT/PATCH requests → ICAP REQMOD → `icap://192.168.122.23:1345/dlpscan` (Python DLP)
- All responses → ICAP RESPMOD → `icap://127.0.0.1:1344/virus_scan` (ClamAV/c-icap)

**[ClamAV/c-icap](../components/clamav-cicap.md)** — ICAP RESPMOD server. c-icap receives HTTP response bodies from Squid, passes them to ClamAV for malware + DLP scanning, and returns `204 No Content` (clean) or a redirect to a block page (detected).

**[Python DLP](../components/python-dlp.md)** — ICAP REQMOD server. Receives POST/PUT/PATCH request bodies, parses multipart/form-data correctly, applies algorithmic DLP validators (Luhn, mod-97, 11-proof, regex), and returns a block response or `200 OK`.

## Key distinctions

**REQMOD vs RESPMOD:**

| | REQMOD | RESPMOD |
|--|--------|---------|
| Timing | Before request is forwarded upstream | After response arrives from upstream |
| Payload | HTTP request (headers + body) | HTTP response (headers + body) |
| Use case | Upload DLP — check what user is sending | Malware scanning, download DLP |
| In this stack | Python DLP on mgmt01:1345 | ClamAV/c-icap on pop01:1344 |

**Why two ICAP servers:** ClamAV's c-icap `virus_scan` service does not correctly parse `multipart/form-data` request bodies — POST content arrives as raw bytes, making DLP pattern matching unreliable. The Python DLP server was added specifically to handle upload inspection with proper multipart parsing. See [Decision: Two-layer DLP](../decisions/two-layer-dlp.md).

**ICAP is Squid-global:** The ICAP service configuration in Squid applies to all listeners automatically — both the LAN listener (`10.0.0.1:3128`) and the NetBird overlay listener (`100.70.154.79:3128`). This differs from SSL Bump, which must be specified per `http_port` directive.

**`bypass=on` for the Python DLP service:** If the DLP container is unreachable, Squid passes traffic through without inspection (fail-open). ClamAV scanning (RESPMOD) remains active. A DLP outage does not block all traffic — a deliberate availability trade-off.

**`configctl proxy restart` required:** A Squid GUI "Apply" alone does not activate ICAP changes in the running daemon. Always restart via `configctl proxy restart` after modifying ICAP configuration.

## Sources

- `raw/Doc1_Squid_WPAD_PAC.md` §2 (Squid ICAP orchestration)
- `raw/Doc2_ClamAV_DLP_Pipeline.md` (ClamAV RESPMOD, why REQMOD doesn't work for uploads)
- `raw/Verslag21.md` (Python DLP decision, two-layer architecture)
