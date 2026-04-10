---
title: "Python DLP ICAP Server — Upload DLP Layer 2"
tags: [python, dlp, icap, reqmod, docker, squid, multipart]
---

# Python DLP ICAP Server — Upload DLP Layer 2

**Role:** ICAP REQMOD service on mgmt01 — scans HTTP request bodies (uploads, form submissions) for sensitive data: credit card numbers (Luhn), IBAN (mod-97), BSN (11-proof), AWS access keys. Blocks POST/PUT/PATCH requests containing matches.  
**Version:** Custom Python 3.11, pyicap (patched), Dockerfile on mgmt01  
**Config location:** `/opt/dlp-icap/` on mgmt01, `/usr/local/etc/squid/pre-auth/dlp-icap.conf` on pop01

---

## How it works in this stack

The Python DLP server runs as a Docker container on mgmt01 and listens on `192.168.122.23:1345`. Squid on pop01 routes POST, PUT, and PATCH requests to this server via ICAP REQMOD before forwarding them to the internet.

The server receives the full HTTP request body, parses `multipart/form-data` content correctly (a capability ClamAV/c-icap lacks), and applies algorithmic validators:
- **Credit cards** — Luhn algorithm (validates check digit, not just format)
- **IBAN** — mod-97 checksum validation
- **BSN** (Dutch citizen ID) — 11-proof validation
- **AWS access keys** — regex pattern `AKIA[A-Z0-9]{16}`

If a match is found above the threshold, the server returns an ICAP block response. Squid presents a styled DLP block page to the client (distinct from the generic Squid 403 page).

**Why a separate server for upload DLP:** ClamAV's `virus_scan` c-icap service does not correctly parse `multipart/form-data` or URL-encoded POST bodies. The c-icap developer confirmed this limitation. POST data is passed as raw bytes, making DLP-pattern matching unreliable for uploaded content. See [Decision: Two-layer DLP](../decisions/two-layer-dlp.md).

---

## Configuration

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

The `sed` patches are required because pyicap on PyPI uses `collections.Callable`, which was removed in Python 3.10+. Without the patch, the container crashes on every ICAP request. See [Finding: pyicap collections bug](../findings/pyicap-collections-bug.md).

### Squid integration (pre-auth include)

File: `/usr/local/etc/squid/pre-auth/dlp-icap.conf`

```squid
icap_service svc_dlp_req reqmod_precache bypass=on icap://192.168.122.23:1345/dlpscan

acl dlp_methods method POST PUT PATCH

adaptation_access svc_dlp_req allow dlp_methods localnet
adaptation_access svc_dlp_req allow dlp_methods subnets
adaptation_access svc_dlp_req deny all
```

Key design choices:
- `bypass=on` — if the DLP container is down, traffic passes through (fail-open). ClamAV malware scanning remains active. A DLP outage does not block legitimate business traffic.
- Only POST/PUT/PATCH — GET requests contain no uploads; routing them through DLP adds latency with no benefit.
- Pre-auth include — survives GUI changes and Squid updates.

After creating this file: `configctl proxy restart`.

### Verify connectivity from pop01

```bash
printf "OPTIONS icap://192.168.122.23:1345/dlpscan ICAP/1.0\r\nHost: 192.168.122.23\r\n\r\n" \
  | nc -w 5 192.168.122.23 1345
# Expected: ICAP/1.0 200 OK, Methods: REQMOD
```

Use `printf`, not `echo -e` — FreeBSD's `/bin/sh` does not interpret `-e` as an escape flag, so `\r\n` is sent literally.

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [Squid](squid.md) | ICAP REQMOD ← | Squid sends POST/PUT/PATCH bodies before forwarding |
| [ClamAV/c-icap](clamav-cicap.md) | complementary | ClamAV handles downloads (RESPMOD); Python handles uploads (REQMOD) |
| mgmt01 Docker | runtime dependency | Container must be running; `restart: unless-stopped` in compose |

**Network path:** pop01 → mgmt01 via `192.168.122.0/24` (WAN/management segment). This path exists independently of the NetBird overlay. The DLP server functions even when the NetBird tunnel is down.

---

## Known issues / gotchas

**pyicap Python 3.10+ incompatibility** — see [Finding: pyicap collections bug](../findings/pyicap-collections-bug.md). The Dockerfile patches this automatically.

**Client IP appears as "unknown" in DLP logs** — the server reads `X-Client-IP` from `self.enc_req_headers` (encapsulated headers), but Squid sends the client IP as an ICAP-level header accessible via `self.headers`. This is a cosmetic issue; blocking works correctly.

**Container logs in Docker stdout** — for Wazuh SIEM integration, add a Docker logging driver (syslog pointed at Wazuh) or configure the Wazuh agent on mgmt01 to monitor container logs. This is a planned Phase 4 task.

**Algorithmic validators have thresholds** — a single credit card number in a POST body does not trigger a block by default. Tune thresholds in `dlp_server.py` according to the use case.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Component: Squid](squid.md)
- [Component: ClamAV/c-icap](clamav-cicap.md)
- [Decision: Two-layer DLP](../decisions/two-layer-dlp.md)
- [Finding: pyicap collections bug](../findings/pyicap-collections-bug.md)
